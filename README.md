# Problema #05 · Feature Engineering Histórico · Grupo B

Laboratorio de Implementación I, 2026 · Analista Jr · Melisa Crespo

> "Los *lags* podrían no ser óptimos si reintroducen datos no deseados como los de la pandemia"

## Hipótesis

Los lags de los meses posteriores a la pandemia reintroducen su información. Tratar esos lags (reemplazarlos o marcarlos) cambia la ganancia respecto de dejarlos como están.

## Sesgos cognitivos

Antes de correr, lo esperado era que marcar qué lags pertenecen a la pandemia (con una dummy, al estilo de la econometría) funcionara mejor que no modificar los lags, o que corregirlos o reemplazarlos. Para que esa expectativa no influyera, la métrica y la comparación se fijaron antes de correr, con las mismas 10 semillas en todas las configuraciones y corrección por comparaciones múltiples.

## Configuraciones

Pandemia = 202003 a 202007. Lags de orden 1, 3 y 6, con sus delta lags.

| Código | Configuración | Qué pasa con un lag que apunta a la pandemia |
|---|---|---|
| 5B1 | Base | Se mantiene el valor original |
| 5B2 | Media histórica | Se reemplaza por la media del cliente hasta 202002 |
| 5B3 | Valores prepandemia | Se reemplaza por el valor del cliente en 202002 (lag 1), 201912 (lag 3) o 201909 (lag 6) |
| 5B4 | Dummy de pandemia | Se mantiene y se agrega `pandemia_lag1`, `pandemia_lag3` o `pandemia_lag6` = 1 |

- En 5B2 y 5B3 el delta del lag reemplazado se recalcula como valor actual − lag nuevo. El mes corriente nunca se modifica.
- No se trata `kmes` (el mes calendario), porque su lag no contiene información de la pandemia.
- El mes al que apunta cada lag se determina con el mismo `shift()` por cliente con el que se construyen los lags.
- Los lags pandémicos aparecen en 10 de los 27 meses de training (lag 1: 202004–202008, lag 3: 202006–202010, lag 6: 202009–202101). Validación (202105) y evaluación (202107) no tienen ninguno, así que los tratamientos solo cambian lo que el modelo aprende.

## Lo que no varía

- **Dataset:** `analistajr_competencia_2026.csv.gz`, con el preprocesamiento del workflow z621 (Catastrophe Analysis, deflación por IPC, FE intra-mes).
- **Períodos:** training 201901–202103 (todos los meses, incluida la pandemia) · validación 202105 · entrenamiento final 201901–202105 · evaluación 202107 (32.938 clientes, con clase conocida).
- **Semillas:** 701669, 743027, 766957, 780523, 795493, 163741, 195401, 751643, 959473, 983327.
- **LightGBM:** binaria {BAJA+1, BAJA+2} contra CONTINUA, `max_bin` 31, `learning_rate` 0,03, hasta 4096 iteraciones con early stopping de 200 sobre AUC en 202105.
- **Grid search por semilla:** `num_leaves` {32, 64, 128} × `min_data_in_leaf` {512, 1024, 2048, 4096, 8192} × `feature_fraction` {0,5, 0,8}. Se elige la combinación de mayor AUC en validación y el modelo final usa esos hiperparámetros y esa cantidad de iteraciones.
- **Ganancia:** +0,975 por BAJA+2 y −0,025 por cualquier otro envío (en millones). Para cada semilla, el máximo de la curva de ganancia suavizada (media móvil de 400) en 202107.
- **Comparación:** Wilcoxon pareado por semilla entre los 6 pares de experimentos, bilateral, con corrección de Holm.

## Resultados

| Experimento | Ganancia media | Desvío | Mínimo | Máximo | Envíos óptimos (promedio) |
|---|---|---|---|---|---|
| 5B1 | 95,69 | 1,78 | 92,49 | 98,22 | 2.168 |
| 5B4 | 95,53 | 1,36 | 93,55 | 97,72 | 2.230 |
| 5B2 | 95,22 | 1,22 | 92,99 | 97,18 | 2.050 |
| 5B3 | 94,62 | 1,03 | 92,97 | 95,94 | 2.228 |

Las curvas de ganancia promedio de los cuatro experimentos se superponen, con el máximo cerca de los 2.000 envíos.

### Wilcoxon pareado (10 semillas, corrección de Holm)

| Par (A − B) | Semillas que gana A / B | Diferencia media | p | p Holm |
|---|---|---|---|---|
| 5B1 − 5B2 | 6 / 4 | +0,47 | 0,625 | 1,00 |
| 5B1 − 5B3 | 8 / 2 | +1,07 | 0,084 | 0,50 |
| 5B1 − 5B4 | 6 / 4 | +0,16 | 0,846 | 1,00 |
| 5B2 − 5B3 | 6 / 4 | +0,60 | 0,322 | 1,00 |
| 5B2 − 5B4 | 4 / 6 | −0,32 | 0,557 | 1,00 |
| 5B3 − 5B4 | 4 / 6 | −0,91 | 0,375 | 1,00 |

Ninguna de las 6 comparaciones es significativa. Ranking por ganancia media: 5B1 > 5B4 > 5B2 > 5B3, sin diferencias significativas entre puestos.

## Discusión y conclusión

- La hipótesis no se confirma: ninguna forma de tratar los lags pandémicos supera a dejarlos como están. Las diferencias entre configuraciones (hasta ~1 M) son menores que la variación entre semillas de una misma configuración (casi 6 M).
- La expectativa previa tampoco se cumple: la dummy (5B4) quedó 0,16 debajo de la base y le ganó en solo 4 de 10 semillas.
- Reemplazar por valores prepandemia fijos (5B3) fue lo más bajo, aunque sin significancia.
- Coincide con el Grupo A, donde anular los lags pandémicos tampoco superó a solo excluir los meses de pandemia.

**Conclusión:** tratar los lags que apuntan a la pandemia no cambia la ganancia de forma detectable.

**Recomendación:** usar los lags 1, 3 y 6 sin modificar (5B1), la configuración más simple y la de mayor ganancia media.

## Limitaciones

- **Un solo mes:** la ganancia se mide solo en 202107.
- **Semillas:** la misma configuración va de 92,5 a 98,2 según la semilla. Con 10 semillas y Holm, una diferencia solo es significativa si 9 o 10 semillas favorecen al mismo experimento.
- **Reconstrucción:** en 5B2 y 5B3 los valores de pandemia se reconstruyeron con reglas simples (la media hasta 202002 o un mes fijo). Podrían reconstruirse de mejores formas, por ejemplo con la tendencia o la estacionalidad de cada cliente.
- **Sin lags:** no se probó quitar todos los lags. Con meses atípicos podría ser óptimo directamente no usarlos, y el diseño no puede descartarlo.
- **Corte optimista:** cada semilla se evalúa en su mejor cantidad de envíos, elegida mirando 202107. El sesgo es el mismo para los cuatro experimentos.

## Futuros problemas

1. Correr una base sin lags y una con lags 1 y 2, para medir el aporte de los lags largos (el Grupo A encontró que los lags 3, 6 y 12 empeoran el modelo).
2. Combinar la dummy de pandemia con excluir esos meses del training.
3. Usar 20 o más semillas, para poder detectar diferencias del orden de 1 M.
4. Evaluar en una cantidad fija de envíos y en más de un mes.

## Estructura

```
notebooks/                workflow de cada experimento (5B1 a 5B4) y notebook de comparación
resultados/5Bx/           grid search, mejores hiperparámetros, ganancias de producción y PARAM
  por_semilla/            ganancias, predicciones, importancia de variables y curva de cada semilla
resultados/comparacion/   tabla resumen, Wilcoxon, ranking, curvas promedio y gráfico
primera_corrida/          grid search de la primera corrida (grilla original)
presentacion/             diapositivas del Grupo B
```

## Cómo correrlo

Requisitos: R con los paquetes `data.table`, `R.utils`, `lightgbm`, `yaml` y `mice` (las notebooks instalan los que falten), y una máquina de 64 GB de RAM.

1. Copiar `analistajr_competencia_2026.csv.gz` a `/content/datasets/`.
2. Correr cada notebook `z621_WorkFlow_01_junior_grupoB_5Bx_Second_Run.ipynb` con kernel R. Cada una corre las 10 semillas (unas 5 a 6 horas) y graba sus salidas en `/content/buckets/b1/exp/WF5Bx_Second_Run`. Si se interrumpe, conviene borrar la carpeta y empezar de cero: al reanudar, LightGBM arma los bins con otra semilla y los resultados no son idénticos.
3. Correr `z621_Comparacion_5B_Second_Run.ipynb`, que lee las cuatro carpetas y graba en `WF5B_comparacion_Second_Run`.

## Notas

- **Primera corrida:** con `num_leaves` 32–512 y `min_data_in_leaf` 64–2048, el óptimo de `min_data_in_leaf` quedó en el borde (2048) en 25 de 40 semillas. Se repitió con la grilla actual y el óptimo quedó en el interior.
- **5B4** se interrumpió en la segunda corrida y se volvió a correr completa. Las combinaciones repetidas entre corridas reproducen la AUC exacta.
- Respecto del workflow original se corrigió el armado de los hiperparámetros finales (`modifyList()` en lugar de `c()`), que hacía que el modelo final usara los valores fijos en lugar de los optimizados.
- No se suben los modelos (`modelo_s*.txt`), que son pesados y se regeneran al correr las notebooks.
