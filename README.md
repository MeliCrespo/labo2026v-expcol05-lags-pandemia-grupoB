# Problema #05 · Feature Engineering Histórico · Grupo B

Laboratorio de Implementación I, 2026 · Analista Jr · Melisa Crespo

> "Los *lags* podrían no ser óptimos si reintroducen datos no deseados como los de la pandemia"

## Hipótesis

Los lags de los meses posteriores a la pandemia reintroducen su información. Tratar esos lags (reemplazarlos o marcarlos) cambia la ganancia respecto de dejarlos como están.

## Configuraciones

Pandemia = 202003 a 202007. Lags de orden 1, 3 y 6, con sus delta lags.

| Código | Configuración | Qué pasa con un lag que apunta a la pandemia |
|---|---|---|
| 5B1 | Base | Se mantiene el valor original |
| 5B2 | Media histórica | Se reemplaza por la media del cliente hasta 202002 |
| 5B3 | Valores prepandemia | Se reemplaza por el valor del cliente en 202002 (lag 1), 201912 (lag 3) o 201909 (lag 6) |
| 5B4 | Dummy de pandemia | Se mantiene y se agrega `pandemia_lag1`, `pandemia_lag3` o `pandemia_lag6` = 1 |

En 5B2 y 5B3 el delta del lag reemplazado se recalcula como valor actual − lag nuevo. El mes corriente nunca se modifica.

## Lo que no varía

- **Dataset:** `analistajr_competencia_2026.csv.gz`, con el preprocesamiento del workflow z621 (Catastrophe Analysis, deflación por IPC, FE intra-mes).
- **Períodos:** training 201901–202103 (todos los meses, incluida la pandemia) · validación 202105 · entrenamiento final 201901–202105 · evaluación 202107 (con clase conocida).
- **Semillas:** 701669, 743027, 766957, 780523, 795493, 163741, 195401, 751643, 959473, 983327.
- **LightGBM:** binaria {BAJA+1, BAJA+2} contra CONTINUA, `max_bin` 31, `learning_rate` 0,03, early stopping 200 sobre AUC en 202105.
- **Grid search por semilla:** `num_leaves` {32, 64, 128} × `min_data_in_leaf` {512, 1024, 2048, 4096, 8192} × `feature_fraction` {0,5, 0,8}. Se elige la combinación de mayor AUC en validación y el modelo final usa esos hiperparámetros y esa cantidad de iteraciones.
- **Ganancia:** +0,975 por BAJA+2 y −0,025 por cualquier otro envío (en millones). Para cada semilla, el máximo de la curva de ganancia suavizada (media móvil de 400) en 202107.
- **Comparación:** Wilcoxon pareado por semilla entre los 6 pares de experimentos, bilateral, con corrección de Holm.

## Resultados

| Experimento | Ganancia media | Desvío | Mínimo | Máximo |
|---|---|---|---|---|
| 5B1 | 95,69 | 1,78 | 92,49 | 98,22 |
| 5B4 | 95,53 | 1,36 | 93,55 | 97,72 |
| 5B2 | 95,22 | 1,22 | 92,99 | 97,18 |
| 5B3 | 94,62 | 1,03 | 92,97 | 95,94 |

Ninguna de las 6 comparaciones es significativa (p Holm ≥ 0,50). El par más cercano es 5B1 contra 5B3: 5B1 gana en 8 de 10 semillas, p = 0,084 sin corregir.

**Conclusión:** tratar los lags que apuntan a la pandemia no cambia la ganancia de forma detectable.
**Recomendación:** usar los lags 1, 3 y 6 sin modificar (5B1).

## Estructura

```
notebooks/          workflow de cada experimento (5B1 a 5B4) y notebook de comparación
resultados/5Bx/     grid search, mejores hiperparámetros, ganancias de producción y PARAM
  por_semilla/      ganancias, predicciones, importancia de variables y curva de cada semilla
resultados/comparacion/   tabla resumen, Wilcoxon, ranking, curvas promedio y gráfico
primera_corrida/    grid search de la primera corrida (grilla original)
presentacion/       diapositivas del Grupo B
```

## Cómo correrlo

1. Copiar `analistajr_competencia_2026.csv.gz` a `/content/datasets/`.
2. Correr cada notebook `z621_WorkFlow_01_junior_grupoB_5Bx_Second_Run.ipynb` con kernel R (VM de 64 GB). Cada una corre las 10 semillas (unas 5 a 6 horas) y graba sus salidas en `/content/buckets/b1/exp/WF5Bx_Second_Run`. Si se interrumpe, conviene borrar la carpeta y empezar de cero: al reanudar, LightGBM arma los bins con otra semilla y los resultados no son idénticos.
3. Correr `z621_Comparacion_5B_Second_Run.ipynb`, que lee las cuatro carpetas y graba en `WF5B_comparacion_Second_Run`.

## Notas

- **Primera corrida:** con `num_leaves` 32–512 y `min_data_in_leaf` 64–2048, el óptimo de `min_data_in_leaf` quedó en el borde (2048) en 25 de 40 semillas. Se repitió con la grilla actual y el óptimo quedó en el interior.
- **5B4** se interrumpió en la segunda corrida y se volvió a correr completa. Las combinaciones repetidas entre corridas reproducen la AUC exacta.
- Respecto del workflow original se corrigió el armado de los hiperparámetros finales (`modifyList()` en lugar de `c()`), que hacía que el modelo final usara los valores fijos en lugar de los optimizados.
- No se suben los modelos (`modelo_s*.txt`), que son pesados y se regeneran al correr las notebooks.
