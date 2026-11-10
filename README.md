# labo2026v-expcol09-undersampling
El experimento mide hasta qué porcentaje de CONTINUA se puede reducir el dataset de la Optimización Bayesiana (OB) sin perder ganancia en el modelo final. Se comparan 6 niveles de undersampling contra un control sin undersampling, con 10 semillas por nivel (60 corridas)
Como Grupo B, se trabaja con un futuro de clase conocida (202107): la ganancia se calcula localmente, sin submits a Kaggle. El notebook es z621_WorkFlow_P9_soloBO_produccionJunior.ipynb, armado sobre el Workflow Analista Jr con bloques del Workflow Analista Sr, sin código propio.
Hipótesis experimental
Reducir los CONTINUA del dataset de la OB baja el tiempo de ejecución, pero a partir de cierto nivel empeora la ganancia del modelo final. Ese nivel es el punto de quiebre.
•	H0: para un nivel dado, la ganancia del modelo final no es menor que la del control (1.0).
•	H1: para un nivel dado, la ganancia del modelo final es menor que la del control.
El undersampling solo afecta a la OB. El modelo final se entrena siempre con el 100 % de los datos, así que la diferencia de ganancia mide cuánto empeoran los hiperparámetros elegidos con menos datos.
Workflow utilizado
La base es el Workflow Analista Jr (Grupo B). Del Workflow Analista Sr se trajeron celdas completas, sin modificar ni agregar código.
Etapa	Origen	Contenido
Preprocesamiento	Jr	Catastrophe Analysis, Data Drifting (IPC), FE intra-mes, lags y delta-lags de orden 1 y 2
Training strategy	Jr	clase01 = {BAJA+1, BAJA+2}, undersampling por training_pct, dtrain y dvalidate
Hyperparameter tuning	Sr	Bayesian Optimization con mlrMBO: reemplaza al Grid Search del Jr
param_final	Sr	Asigna los mejores hiperparámetros con su transformación 2^x; corrige la duplicación de hiperparámetros del Jr
Modelo final	Jr	Un solo lgb.train sobre el final train, con semilla_primigenia como semilla, y predict sobre el futuro. Sin ensemble semillerío.
Evaluación	Jr	Curva de ganancia local sobre 202107
No se usan del Sr: el FE de tendencias, las variables de Random Forest, los canaritos, el semillerío ni el submit a Kaggle. Lo único agregado es una celda de texto que indica los valores a cambiar en cada corrida.
Training strategy
Se mantienen las fechas del Workflow Analista Jr. Solo el training de la OB recibe undersampling.
Conjunto	Meses	Undersampling	Uso
Training (OB)	201901 – 202103	sí, según training_pct	entrenar cada modelo de la OB
Validation (OB)	202105	no	medir el AUC de cada punto de la OB
Final train	201901 – 202105	no	entrenar el modelo final
Futuro	202107	no	calcular la ganancia (clase conocida)
202104 y 202106 quedan como meses de separación, porque la clase BAJA+2 de un mes se conoce dos meses después.
Factores del experimento
El diseño es 6 niveles × 10 semillas = 60 corridas, una ejecución del notebook por corrida.
•	Factor: PARAM$trainingstrategy$training_pct, el porcentaje de CONTINUA que se conserva en el training de la OB. Los BAJA+1 y BAJA+2 se conservan siempre.
•	Niveles: 1.0 (control), 0.5, 0.2, 0.1 y 0.05.
•	Repeticiones: 10 semillas (PARAM$semilla_primigenia), las mismas en los 5 niveles.
Con la misma semilla, la columna azar es idéntica en todos los niveles. Por eso las muestras quedan anidadas: los CONTINUA de 0.05 están dentro de los de 0.1, y así hasta 1.0. Dentro de una semilla, lo único que cambia es el nivel, lo que permite un análisis pareado.
Queda constante en las 50 corridas: preprocesamiento, fechas, espacio de búsqueda, 30 iteraciones de la OB, modelo final de una sola semilla y cálculo de la ganancia.
Numeración propuesta de corridas: PARAM$experimento = 91LS, con L = nivel (1 = 1.0, 2 = 0.75, 3 = 0.5, 4 = 0.2, 5 = 0.1, 5 = 0.6) y S = índice de semilla (0 a 9). Las corridas van de 9110 a 9169. Cada una escribe en su propia carpeta WF91LS, para que la OB no retome un HT.RDATA ni reutilice modelos de otra corrida.
Una corrida: OB, modelo final y ganancia
Cada corrida sigue los mismos tres pasos; el nivel de undersampling interviene solo en el primero.
 
una corrida · 7 pasos, el factor destacado
El recuadro destacado es el único paso que cambia entre niveles; todo lo demás es idéntico en las 50 corridas.
1.	Optimización Bayesiana (mlrMBO, surrogate kriging matern3_2, criterio Expected Improvement).
o	Optimiza el AUC en validation 202105, no la ganancia.
o	Hiperparámetros: num_iterations (2^2 a 2^10), num_leaves (2^2 a 2^11), learning_rate (2^-7 a 2^-0.1), feature_fraction (0.05 a 0.9) y min_sum_hessian_in_leaf (2^-10 a 2^10).
o	Puntos iniciales al azar más 30 iteraciones inteligentes. El historial queda en BO_log.txt.
2.	Modelo final.
o	Se entrena sobre 201901 – 202105 completo, sin undersampling, con los mejores hiperparámetros de la OB.
o	Un solo modelo, con semilla_primigenia como semilla. La probabilidad de cada cliente es la de ese modelo.
3.	Ganancia sobre 202107.
o	+0.975 por cada BAJA+2 contactado y −0.025 por cada otro cliente.
o	Se ordena por probabilidad descendente, se acumula la ganancia y se suaviza con una media móvil de ancho 400.
o	Se registran la ganancia suavizada máxima y la cantidad de envíos donde ocurre.
En Producción no se hace ensemble semillerío. Las 10 semillas no se promedian: son repeticiones del experimento, y cada una da su propia ganancia.
Métricas
La métrica principal es la ganancia en un corte fijo; el resto sirve de apoyo. Todas salen de los archivos que el notebook ya genera, y se reúnen después en una tabla de 50 filas (una por corrida).
Métrica	Archivo	Rol
Ganancia en un corte fijo de envíos (por ejemplo, los envíos óptimos medianos del control)	ganancias.txt (gan_acum en la fila N)	principal: no elige el mejor punto mirando el propio mes
Ganancia suavizada máxima y envíos óptimos	PARAM.yml (resultado)	secundaria: optimista, porque el máximo se elige sobre 202107
Mejor AUC en validation 202105	BO_log.txt (columna y)	apoyo: muestra si la OB encuentra peores hiperparámetros
Tiempo de la OB	BO_log.txt (suma de exec.time)	costo: la otra mitad de la hipótesis
Variabilidad entre semillas	las 10 corridas de cada nivel	apoyo: estabilidad del nivel
Análisis estadístico y criterio de quiebre
Un nivel “rompe el modelo” cuando pierde ganancia frente al control en forma significativa y relevante. El punto de quiebre es el primer nivel, bajando desde 1.0, que cumple ambas condiciones.
1.	Diferencias pareadas. Para cada semilla s y nivel n: d(n, s) = ganancia(n, s) − ganancia(1.0, s).
2.	Prueba global. Test de Friedman sobre los 5 niveles (bloques = semillas), para saber si hay alguna diferencia entre niveles.
3.	Comparaciones contra el control. Test de rangos con signo de Wilcoxon, pareado y unilateral (H1: el nivel gana menos), sobre las 10 diferencias de cada nivel. Son 4 pruebas, con p-valores ajustados por Holm.
4.	Criterio de quiebre. El nivel se considera roto si se cumplen las dos condiciones:
o	p ajustado < 0.05;
o	la mediana de la pérdida supera el 2 % de la ganancia mediana del control (umbral fijado antes de correr).
5.	Nivel recomendado. El menor training_pct que no está roto, contrastado con el ahorro de tiempo de la OB.
Un resultado no significativo no se interpreta como estabilidad. Con n = 10 la prueba tiene poca potencia (el menor p-valor unilateral posible es 1/1024). Para afirmar que un nivel no se rompe, el intervalo de confianza de la mediana de las diferencias (conf.int = TRUE) tiene que quedar por encima de −2 %; si no, el nivel se reporta como no concluyente.
Costo, limitaciones y puntos a aprobar
El experimento requiere unos 2.550 entrenamientos: unos 2.500 de OB y 50 modelos finales, uno por corrida.
Se supone que cada OB entrena unos 50 modelos (unos 20 puntos iniciales, valor por defecto de mlrMBO, más 30 iteraciones). Los modelos finales usan el 100 % de los datos, así que pesan más que un entrenamiento de OB. Antes de lanzar las 50 corridas se hará una corrida piloto (nivel 1.0) para medir el tiempo.
Limitaciones
•	Se evalúa un solo mes futuro (202107); el resultado puede no trasladarse a otros meses.
•	Con 10 semillas la potencia es baja: pérdidas pequeñas pueden no detectarse.
•	La OB optimiza AUC, no ganancia: un nivel puede mantener el AUC y perder ganancia, o al revés.
•	El undersampling es aleatorio simple; no se prueban métodos dirigidos.
