# Predicción de perfiles de ley de cobre en un yacimiento de veta chileno

**DII8093 Redes Neuronales Profundas — Informe Final**
Nicolás Fierro Oñate — Pontificia Universidad Católica de Valparaíso

Comparación bajo validación espacial estricta (bloques 3D de 100 m) entre
Indicator Kriging 3D, un Transformer espacial con prior de covarianza Matérn
aprendible en la atención, y un GNN EdgeConv, más generación de sondajes
sintéticos con WGAN-GP y evaluación de su efecto sobre los tres modelos.

## Estructura del repositorio

| Archivo | Descripción |
|---|---|
| `Sondajes_finales.ipynb` | Pipeline completo (split, IK, Transformer, GNN, WGAN, gemelos, figuras). **Ejecutado: las salidas guardadas respaldan los resultados del informe.** |
| `Limpieza_de_sondajes.ipynb` | Preprocesamiento: `sondajes.xlsx` → `sondajes_clean.xlsx` (duplicados, capping p99, imputación de litología). |
| `sondajes.xlsx` | Datos crudos (yacimiento de veta chileno). |
| `sondajes_clean.xlsx` | Dataset limpio que consume el pipeline (122.926 intervalos, 1.269 sondajes). |
| `requirements.txt` | Dependencias. |
| `resultados/` | Salidas generadas al ejecutar: predicciones (`.npy`), métricas (`.json`, `.csv`) y figuras (`.png`). |

## Cómo reproducir

**Opción A — Google Colab (recomendada):** abrir `Sondajes_finales.ipynb` en
Colab (botón *Open in Colab* o subir el archivo) y ejecutar las celdas en
orden. La CELDA 1 descarga automáticamente `sondajes_clean.xlsx` desde este
repositorio si no está presente; no requiere montar Drive. Entorno sugerido:
GPU T4 (acelera Transformer/GNN/WGAN; el IK corre en CPU).


**Opción B — Local:**
```bash
git clone https://github.com/ARSENICOX99/Proyecto-final-RNP.git
cd Proyecto-final-RNP
pip install -r requirements.txt
jupyter notebook Sondajes_finales.ipynb
```
Las celdas que comienzan con `!pip install pykrige -q` / `!pip install
dtw-python -q` funcionan igual en Colab y Jupyter.

## Orden de ejecución y tiempos aproximados (Colab, GPU T4)

1. **CELDAS 1–4**: carga, transformación del target, split por bloques,
   construcción de vecindarios (~5 min).
2. **CELDA 5 (IK baseline)**: ~25–30 min (7 umbrales × ~4 min; guarda
   checkpoint `resultados/F_hat_ik_ckpt.pkl` y se puede reanudar si se
   interrumpe).
3. **CELDAS 6–8**: Transformer espacial, entrenamiento y evaluación (~10 min).
4. **CELDAS 9–11**: GNN EdgeConv **versión original (DEPRECADA, ver abajo)**.
5. **CELDAS 12–17**: WGAN-GP (~15 min), generación de sondajes gemelos y
   re-entrenamiento del Transformer aumentado (~10 min).
6. **CELDAS PARCHE 1–6** (al final del notebook): fix del GNN, GNN corregido,
   baselines triviales, IK+gemelos (~30 min), GNN+gemelos y **tabla final
   consolidada** (~50 min en total).
7. **CELDAS 18–22 (v2)**: figuras del informe (DTW, Moran, mapas, trade-off).

## Semillas y reproducibilidad

`SEED = 42` fija numpy, python y TensorFlow (`tf.keras.utils.set_random_seed`)
en la CELDA 1. El split por bloques y el IK son completamente deterministas.
El entrenamiento en GPU puede introducir no-determinismo de bajo nivel
(reducciones CUDA), por lo que las métricas de las redes pueden variar en la
tercera decimal entre corridas; las conclusiones del informe son robustas a
esa variación. Los resultados exactos reportados quedan respaldados por las
salidas guardadas en el notebook y por los archivos de `resultados/`.

## Trazabilidad de celdas deprecadas

Durante el desarrollo se detectó y corrigió un defecto en la agregación max
del GNN (162 puntos de train sin vecinos válidos tras la exclusión anti-fuga
contaminaban la pérdida con un valor centinela). Por trazabilidad se conservan
en el notebook:

- **CELDA 10 (GNN v1)** — deprecada; los resultados del informe provienen de
  la **CELDA PARCHE-2** (GNN v2, corregido).
- **Moran's I v1 (CELDA 20)** — deprecado (calculaba sobre |error| e incluía
  vecinos intra-sondaje); los valores del informe provienen de la
  **CELDA 20b v2** (residuo con signo, vecinos inter-sondaje).

## Correspondencia resultados ↔ informe

| Elemento del informe | Origen en el código |
|---|---|
| Tabla I (comparación 9 modelos) | `resultados/tabla_final_completa.csv` (PARCHE-6) |
| Tabla II (DTW y Moran's I) | `resultados/dtw_medio_test.json` (CELDA 18 v2) + salida CELDA 20b v2 |
| Fig. 1 (perfiles + DTW) | `resultados/fig1_perfiles_dtw.png` (CELDA 18 v2) |
| Fig. 2 (trade-off con baselines) | `resultados/fig5_tradeoff_v2.png` (CELDA 22 v2) |
| Fig. 3 (mapa de error en planta) | `resultados/fig3_mapa_error.png` (CELDA 20) |
| Fig. 4 (WGAN real vs sintético) | `resultados/wgan_real_vs_sintetico.png` (CELDA 14) |
| MAE por clase de ley | salida CELDA 19 v2 |
| Efecto de gemelos por modelo (Δ) | salida final de PARCHE-6 |

## Nota sobre los datos

Los datos provienen de un yacimiento real y se incluyen únicamente con fines
de evaluación académica del curso DII8093.
