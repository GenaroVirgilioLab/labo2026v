# Entrega final - Laboratorio de Implementación 1

Competencia Kaggle `labo-1-virtual-2026-analista-sr`.

**Envío final elegido: `KA7_s300_10000.csv`** (semillerío de 300 semillas, 10000 envíos).

## Archivos

| Archivo | Qué es |
|---|---|
| `Prueba_7.ipynb` | Notebook que genera el envío a partir del dataset original. Incluye las salidas de la corrida que produjo el archivo final. |
| `PARAM.yml` | Parámetros de esa corrida, grabados por el notebook (incluye las 300 semillas y los hiperparámetros ganadores). |
| `HT.RDATA` | Estado completo de la Bayesian Optimization (60 puntos). |
| `BO_log.txt` | Los 60 puntos de la Bayesian Optimization ordenados por AUC. |

## Cómo reproducir

1. Google Colab con runtime R sobre una máquina virtual de 128 GB de RAM, con el bucket montado en `/content/buckets/b1` y el dataset `analistasr_competencia_2026.csv.gz` en `/content/datasets/`.
2. Abrir `Entrega_Final.ipynb` y correr todas las celdas en orden.
3. El notebook trabaja en `/content/buckets/b1/exp/WF7` y deja los envíos en `WF7/kaggle/`. Genera los nueve cortes de 8000 a 12000 y los sube a Kaggle; el envío final es `KA7_s300_10000.csv`.

La Bayesian Optimization tarda unas 18 horas (60 puntos, 100 modelos por punto). Hay dos formas de correr:

- **Con `HT.RDATA` copiado en `exp/WF7/` antes de empezar:** el notebook detecta que la optimización ya terminó, toma su resultado y pasa directo al entrenamiento final. Así se hizo la corrida cuyas salidas están en el notebook. Tarda entre 3 y 5 horas.
- **Con la carpeta `exp/WF7/` vacía:** corre la Bayesian Optimization completa desde cero. Tarda unas 25 horas en total.

Todo el azar sale de `PARAM$semilla_primigenia <- 149911`.

## Cambios respecto del workflow z739 Analista Sr de la cátedra

El preprocesamiento no se modificó: Catastrophe Analysis, Data Drifting, Feature Engineering intra-mes e histórico, variables de Random Forest y Canaritos Asesinos quedan igual. Los cambios están en la Bayesian Optimization y en el semillerío final.

| Qué | Cátedra | Esta entrega |
|---|---|---|
| Evaluación de cada punto de la Bayesian Optimization | 1 LightGBM, AUC en validación | semillerío de 100 LightGBM, AUC del promedio de sus probabilidades |
| `num_iterations` (rango de búsqueda) | 4 a 1024 | 4 a 64 |
| `learning_rate` (límite inferior) | 2^-7 | 2^-4 |
| Iteraciones de la Bayesian Optimization | 30 | 40 |
| Semillas del modelo final | 10 | 300 |
| Semilla primigenia | 102191 | 149911 |

**Semillerío dentro de la Bayesian Optimization.** Cada configuración de hiperparámetros se evalúa entrenando 100 LightGBM con semillas distintas, promediando sus probabilidades sobre 202107 y calculando la AUC de ese promedio. La optimización busca entonces hiperparámetros que sean buenos para un semillerío, que es lo que se usa en el modelo final, y no para un único modelo. Las 100 semillas son las mismas en todos los puntos, de modo que la diferencia de AUC entre dos configuraciones se debe a los hiperparámetros.

**Modelos chicos.** Como el modelo final es un promedio de muchos modelos, el espacio de búsqueda se limitó a modelos de hasta 64 árboles con learning rate alto.

**Semillerío final de 300.** El modelo final promedia 300 LightGBM entrenados con todos los meses 201901-202107.

## Resultado de la Bayesian Optimization

Punto ganador (iteración 24 de 40):

| Hiperparámetro | Valor |
|---|---|
| `num_iterations` | 46 |
| `num_leaves` | 734 |
| `learning_rate` | 0.292 |
| `feature_fraction` | 0.288 |
| `min_sum_hessian_in_leaf` | 37.0 |
| AUC del semillerío en 202107 | 0.94126 |

El resto de los parámetros de LightGBM son los fijos de la cátedra y están en `PARAM.yml`.

## Elección del corte

El corte óptimo es el punto donde la proporción de BAJA+2 entre los clientes que se agregan cae por debajo de 2.5 %, que es el equilibrio de la función de ganancia (+0.975 por cada BAJA+2, -0.025 por el resto).

### Backtest con Montecarlo: el corte óptimo cambia de mes a mes

Primero intenté fijar el corte con los meses que tienen clase. Para cada mes M de 202103 a 202107 entrené con los meses hasta M-2 un semillerío de 30 semillas con los hiperparámetros finales, predije M y simulé 1000 particiones 30/70 como el público y el privado de Kaggle. De cada mes tomé el centro de la meseta de ganancia (cortes a menos de 0.5 % del máximo) y lo llevé a la cantidad de clientes de 202109.

| Mes | BAJA+2 en el mes | Corte óptimo equivalente en 202109 |
|---|---|---|
| 202103 | 0.594 % | 8625 |
| 202104 | 0.697 % | 9905 |
| 202105 | 0.521 % | 8254 |
| 202106 | 0.665 % | 13578 |
| 202107 | 0.793 % | 12917 |

Lo que deja este análisis:

- **Los meses no coinciden.** El corte óptimo va de 8254 a 13578 envíos según el mes, una diferencia mucho mayor que la que hay entre dos envíos candidatos.
- **El corte depende de cuántas bajas tiene el mes.** El mes con menos BAJA+2 (202105) pide el corte más bajo y el mes con más (202107) uno de los más altos. La proporción de bajas de 202109 no se conoce, así que el backtest no alcanza para ubicar su corte.
- **Ningún corte fijo sirve para todos los meses.** El mejor corte único logra en promedio el 97.9 % de la ganancia máxima de cada mes.
- **La curva de cada mes es plana cerca de su máximo.** Dentro de un mismo mes, moverse unos cientos de envíos cambia poco la ganancia; el riesgo está en que el mes sea distinto, no en afinar el corte.

Por eso el corte final no sale del backtest sino del leaderboard público, que es la única información disponible sobre 202109. Este análisis no está en el notebook porque no interviene en la generación del envío.

### Corte a partir del leaderboard público

Para ubicar ese punto usé todas mis series de envíos de la competencia (37 series, 461 envíos) y no el máximo público de un envío aislado, que depende de 3 o 4 clientes. La diferencia de puntaje público entre dos cortes consecutivos de una misma serie permite estimar la proporción de BAJA+2 de esa franja de 500 clientes. En el conjunto de las series esa proporción baja de forma pareja: 3.7 % en 8000, 2.5 % cerca de 10000 y 1.4 % en 12000.

El modelo final muestra el mismo quiebre:

| Envíos | 8000 | 8500 | 9000 | 9500 | **10000** | 10500 | 11000 | 11500 | 12000 |
|---|---|---|---|---|---|---|---|---|---|
| Público | 450.998 | 452.998 | 460.498 | 457.164 | **463.998** | 457.581 | 455.498 | 454.998 | 446.664 |

---

---

<h2 align="center">🌱 In Seeds We Trust 🌱</h2>
