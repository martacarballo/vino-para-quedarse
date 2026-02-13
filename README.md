# Vino para Quedarse
Proyecto de análisis de datos sobre vino.

## Contenido
## Contenido
- [Descripción](#descripción)
- [Tecnologías](#tecnologías)
- [Visualización en Power BI (Dashboard)](#visualización-en-power-bi)  <
- [Extracción de datos](#extracción-de-datos)
- [Scraping](#scraping)
- [Unión de datasets](#unión-de-datasets)
- [Análisis de Machine Learning](#análisis-de-machine-learning)
  - [Limpieza y Preprocesamiento](#limpieza-y-preprocesamiento)
  - [Ingeniería de Variables (Feature Engineering)](#ingeniería-de-variables-feature-engineering)
  - [Modelado y Evaluación](#modelado-y-evaluación)
  - [Simulador Climático Élite](#simulador-climático-élite)
- [Resultados](#resultados)
- [Futuras mejoras](#futuras-mejoras)
- [Autores](#autores)


🍷 Vino para Quedarse
Análisis climático y su influencia en la calidad y precio del vino en Ribera del Duero (2004–2024)
📌 Descripción del proyecto

Este proyecto analiza cómo las temperaturas, precipitaciones y humedad de la Ribera del Duero influyen en la añada del vino, y cómo estos factores afectan al precio y rating de los vinos según Vivino.

Para ello se utilizan dos fuentes de datos principales:

🔹 Datos climáticos (AEMET API, 2004–2024)
🔹 Datos enológicos (Vivino API / Scraping)

Finalmente, ambos datasets se integran por año y se analizan conjuntamente para identificar patrones.

🛠️ Tecnologías utilizadas
🔹 Lenguajes y librerías

Python

Pandas

Requests

Python-dotenv

BeautifulSoup / Requests (scraping)

Scikit-Learn (Modelos de ML)

Seaborn / Matplotlib (Visualización de datos)

XGBoost / RandomForest (Algoritmos predictivos)

Jupyter Notebook

## 📊 Visualización en Power BI

Se ha desarrollado un Dashboard interactivo que conecta los datos históricos con el modelo de predicción. 
- **Calculadora Predictiva:** Interfaz visual para simular el precio del kilo de uva según variables meteorológicas.

🔹 Fuentes externas

AEMET (Agencia Estatal de Meteorología)

Vivino API no oficial / scraping JSON

📂 Estructura del proyecto
/data
    datos_mensuales_aemet.xlsx
    vinos_vivino2.xlsx
    vinos_aemet_merged.xlsx
/src
    aemet_api.py
    vivino_scraper.py
    merge_datasets.py
README.md
🌦️ 1. Extracción de datos climáticos — AEMET API

Se utiliza una API key protegida con .env, cargada mediante dotenv.

import os
from dotenv import load_dotenv
import requests
import pandas as pd
from io import StringIO
import time

load_dotenv()
API_KEY = os.getenv("AEMET_API_KEY")

if API_KEY is None:
    print("No se ha encontrado AEMET_API_KEY. Revisa tu archivo .env")
else:
    print("✅ API key cargada correctamente. Longitud:", len(API_KEY))
    print("Inicio de la key:", API_KEY[:10], "... (oculto)")

Este script autentica correctamente la API key y sirve como base para realizar solicitudes a AEMET (temperaturas, precipitaciones y humedad por año).

🍇 2. Scraping de Vivino — API JSON no documentada

Para obtener los precios, rating y características de vinos de la Ribera del Duero se utiliza la URL:
https://www.vivino.com/api/explore/explore

import requests
import time
import pandas as pd

URL = "https://www.vivino.com/api/explore/explore"

HEADERS = {
    "User-Agent": (
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
        "AppleWebKit/537.36 (KHTML, like Gecko) "
        "Chrome/123.0 Safari/537.36"
    )
}

**Parámetros base:**

PARAMS_BASE = {
    "currency_code": "EUR",
    "min_rating": 0,
    "order_by": "price",
    "order": "desc",
    "price_range_min": 0,
    "price_range_max": 500,
    "discount_prices": "false",
    "wine_style_ids[]": 180,   # Ribera del Duero tinto
    "country_codes[]": "es",
    "wine_type_ids[]": 1,      # vino tinto
    "region_ids[]": 405        # Ribera del Duero
}

**Bucle de scraping (hasta 100 páginas)**

NUM_PAGES = 100
all_wines = []

for page in range(1, NUM_PAGES + 1):
    print(f"Scrapeando página {page}...")
    params = PARAMS_BASE.copy()
    params["page"] = page

    response = requests.get(URL, headers=HEADERS, params=params)
    response.raise_for_status()

    data = response.json()
    explore_data = data.get("explore_vintage") or data.get("exploreV2", {}).get("explore_vintage")
    records = explore_data.get("records", [])

    if not records:
        print("Sin más vinos en esta página, paro el bucle.")
        break

    for rec in records:
        vintage = rec.get("vintage", {})
        price_info = rec.get("price") or (rec.get("prices") or [None])[0]
        stats = vintage.get("statistics", {})

        wine_dict = {
            "pagina": page,
            "vintage_id": vintage.get("id"),
            "nombre": vintage.get("name"),
            "seo_name": vintage.get("seo_name"),
            "year": vintage.get("year"),
            "rating": stats.get("ratings_average"),
            "n_ratings": stats.get("ratings_count"),
            "precio": price_info.get("amount") if price_info else None,
            "moneda": (price_info.get("currency") or {}).get("code") if price_info else None
        }

        all_wines.append(wine_dict)

    time.sleep(1)

**Guardado y limpieza**

df_vinos = pd.DataFrame(all_wines)
df_vinos = df_vinos.drop_duplicates(subset=['vintage_id'])
df_vinos.to_excel("vinos_vivino2.xlsx", index=False)

🔗 3. Unión de datasets (Vivino + AEMET)

Se enlazan las dos fuentes por año.

import pandas as pd

df_aemet = pd.read_excel("datos_mensuales_aemet.xlsx")   # columna "año"
df_vivino = pd.read_excel("vinos_vivino2.xlsx")          # columna "year"

df_aemet["año"] = pd.to_numeric(df_aemet["año"], errors="coerce")
df_vivino["year"] = pd.to_numeric(df_vivino["year"], errors="coerce")

df_merged = df_vivino.merge(
    df_aemet,
    left_on="year",
    right_on="año",
    how="left"
)

df_merged.to_excel("vinos_aemet_merged.xlsx", index=False)

El resultado incluye:

✔️ Datos del vino (precio, rating, bodega, vintage…)

✔️ Datos climáticos asociados al mismo año (temperatura, humedad, precipitación)

🤖 4. Análisis de Machine Learning

Se ha desarrollado un pipeline completo de Ciencia de Datos para predecir el precio y la calidad del vino basándose en factores climáticos y enológicos.

### 🔹 Preprocesamiento y Limpieza
- **Tratamiento de Nulos:** Uso de `KNNImputer` y eliminación estratégica de registros sin precio real.
- **Limpieza de Datos:** Normalización de precios (eliminación de símbolos de moneda y ajuste de decimales) y conversión de porcentajes de alcohol.

### 🔹 Ingeniería de Variables (Feature Engineering)
Para mejorar la precisión del modelo, se crearon variables sintéticas de alto impacto:
- **Bodega_Score_Rating:** Un índice que captura el prestigio histórico de cada bodega basándose en sus precios y ratings.
- **Variables Trimestrales:** Agrupación de datos climáticos (temperatura, precipitación, humedad) en trimestres (T1, T2, T3, T4) para identificar ciclos críticos de la vid.
- **Análisis de Uvas:** Codificación mediante `MultiLabelBinarizer` para manejar mezclas de variedades y creación de un flag específico para *Tempranillo Puro*.
- **Master_Score:** Un "ADN de precio" que agrupa la relación histórica entre Bodega-Año-Rating.

### 🔹 Modelado Predictivo
Se entrenaron y evaluaron diferentes arquitecturas:
1. **Modelos de Regresión:** Para la predicción del precio exacto (€/kg) utilizando `RandomForestRegressor` y `KNeighborsRegressor`.
2. **Modelos de Clasificación:** Para categorizar la calidad del vino.
3. **Clustering (K-Means):** Utilizado para segmentar los vinos en "Perfiles Económicos" o "Perfiles de Gran Prestigio" según el impacto de la sequía y el clima.

### 🏆 Modelo Maestro y Simulador
El proyecto culmina con un **Modelo Maestro (Random Forest de 800 árboles)** optimizado mediante `GridSearchCV`, capaz de realizar predicciones con un rango de confianza específico.

Se incluye una **Calculadora Climática Final** que permite simular escenarios (ej. heladas en mayo o variaciones de humedad en septiembre) para predecir el precio de mercado resultante.

📊 4. Resultados del análisis

El dataset combinado permite:

Calcular correlaciones clima → rating/precio

Estudiar la influencia del clima en cada añada

Identificar años climáticamente buenos y malos

Preparar modelos predictivos basados en clima

Modelo predictivo capaz de estimar precios con un intervalo de seguridad basado en variables climáticas críticas como las heladas de mayo y la humedad de febrero.

🚀 Futuras mejoras

- **Automatización anual:** Conexión mediante scripts automáticos a la API de AEMET para que el dataset se actualice solo al finalizar cada año agrícola.
- **Expansión geográfica:** Escalar el modelo a otras Denominaciones de Origen (Rioja, Priorat, Rueda) para identificar patrones climáticos comunes en toda España.
- **Despliegue Web (Streamlit):** Crear una versión web ligera del modelo de Machine Learning para consulta rápida desde dispositivos móviles en el viñedo.
- **Análisis de Series Temporales:** Implementar modelos de Deep Learning (LSTM) para predecir la evolución del mercado en la próxima década frente al cambio climático.
- **Inclusión de Imágenes Satelitales:** Utilizar datos de la misión Copernicus para analizar el índice de vegetación (NDVI) y afinar aún más la precisión del modelo maestro.

👥 Autores

Marta Carballo

Alejandro de Tuero