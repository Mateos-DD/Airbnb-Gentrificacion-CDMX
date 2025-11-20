# Airbnb, gentrificación y precios de renta en la Ciudad de México (2020–2025)

> Informe de investigación basado en los notebooks:
> - `Limpieza de datos 24_25.ipynb`
> - `UNION  BASES  DATOS.ipynb`
> - `Preparacion_datos AIRBNB  2025 listing_18.ipynb`
> - `AIRBNB  2025 listing_18 .ipynb`
> - `EDA - K_MEANS AIRBNB  LISTING_GZ.ipynb`
> - `Factores _ Ganancia anual.ipynb`

---

## 1. Objetivo general del proyecto

Este proyecto analiza la evolución y la distribución espacial de la oferta de alojamientos de Airbnb en la Ciudad de México entre 2020 y 2025, con el fin de:

- Caracterizar la oferta de **alojamientos completos** (Entire home/apt) y otros tipos de habitación.
- Explorar la **concentración espacial** de los anuncios por alcaldía.
- Construir **series de tiempo** (diarias, mensuales, trimestrales) por alcaldía y tipo de alojamiento.
- Identificar **clusters** de oferta similares mediante técnicas de aprendizaje no supervisado.
- Analizar factores asociados a la **rentabilidad anual estimada** de los alojamientos.

La motivación central es entender si los patrones espaciales y temporales observados son consistentes con procesos de **gentrificación**, en particular en las alcaldías centrales de la CDMX (Cuauhtémoc, Miguel Hidalgo, Benito Juárez, etc.).

---

## 2. Fuentes de datos

### 2.1 Inside Airbnb (CDMX, múltiples cortes)

Los notebooks usan principalmente archivos descargados del proyecto **Inside Airbnb**, que captura información pública de la plataforma:

- **Módulo listings**:  
  - `listings.csv` y `listings.csv.gz` para varios cortes de tiempo:  
    - 25 SEP 2024  
    - 27 DIC 2024  
    - 19 MAR 2025  
    - 25 JUN 2025  
  - Versiones comprimidas por trimestre: `listings_gz_MAR.csv`, `listings_gz_JUN.csv`, `listings_gz_SEP.csv`, `listings_gz_DIC.csv`, etc.
- **Módulo calendar**:
  - Datos diarios de disponibilidad y precio (`calendar.csv`) para distintos años / trimestres.
- **Módulo reviews**:
  - Reseñas por estancia (`reviews.csv` y archivos `detailed_reviewsQ1.csv`, …, `detailed_reviewsQ4.csv`).

Estos archivos contienen información estándar de Inside Airbnb, como:
- Identificador del anuncio (`id`, `listing_id`).
- Coordenadas (`latitude`, `longitude`).
- Tipo de habitación (`room_type`).
- Precio por noche (`price`).
- Número de huéspedes (`accommodates`), número de camas (`beds`), baños (`bathrooms_text`), etc.
- Variables de host (`host_id`, `host_since`, `host_is_superhost`, etc.).
- Variables de reseñas y calificaciones (`number_of_reviews`, `review_scores_rating`, `reviews_per_month`…).

### 2.2 Conjuntos históricos adicionales

En `UNION  BASES  DATOS.ipynb` se incorporan, además:

- **Airbnb Mexico City – 2020–2021**  
  - `listings 2021.csv`  
  - `calendar 2021.csv`  
  - `reviews.csv`

- **Airbnb Mexico City – 2023 (por trimestre)**  
  - `listings_q1.csv`, `listings_q2.csv`, `listings_q3.csv`, `listings_q4.csv`  
  - `detailed_reviewsQ1.csv`, …, `detailed_reviewsQ4.csv`

### 2.3 Otras bases auxiliares

- Conjuntos preprocesados:
  - `UNION.csv`, `UNION_GZ.csv`, `UNION_18C.csv`.
  - Paneles y series:
    - `AIRBNB_SERIE_CALENDAR.csv`
    - `AIRBNB_SERIE_MENSUAL.csv`
    - `AIRBNB_SERIE_TRIMESTRAL.csv`
    - `airbnb_cdmx_month_panel.csv`
    - `airbnb_cdmx_quarter_panel.csv`
- Un dataset externo de guía (`Proyecto Github Guia`) que se usa como referencia de estructura y para comparar la lógica de integración.

---

## 3. Resumen de cada notebook

### 3.1 `Limpieza de datos 24_25.ipynb`

**Objetivo:**  
Realizar la **primera limpieza extensa** de los datos de Airbnb para los cortes 2024–2025, unificar esquemas de columnas y generar paneles y series de tiempo intermedias.

**Pasos principales:**

1. **Carga de múltiples cortes de listings**  
   - Se leen distintos `listings.csv` y `listings.csv.gz` (SEP 2024, DIC 2024, MAR 2025, JUN 2025).
   - Se homogeneizan nombres de columnas y tipos de datos (por ejemplo, `price` como numérico).

2. **Limpieza de variables clave**  
   - Conversión de precios de string a numérico (eliminando `$`, comas, etc.).
   - Manejo de valores faltantes y columnas redundantes.
   - Parsing de texto de baños (`bathrooms_text`) con una función como:
     - Detección de “half bath” y números explícitos → creación de `bathrooms_parsed`.

3. **Construcción de paneles y uniones**  
   - Se construyen uniones como:
     - `UNION.csv`, `UNION_GZ.csv`, `UNION_18C.csv`.
   - Se comienzan a armar paneles a nivel anuncio:
     - `airbnb_cdmx_month_panel.csv`
     - `airbnb_cdmx_quarter_panel.csv`

4. **Construcción de series de tiempo desde calendar**  
   - A partir del módulo `calendar` se construye una base consolidada `CALENDAR` y se guarda en `AIRBNB_SERIE_CALENDAR.csv`.
   - Luego, mediante agregaciones (`groupby` por alcaldía y periodo) se obtienen:
     - `AIRBNB_SERIE_MENSUAL.csv`
     - `AIRBNB_SERIE_TRIMESTRAL.csv`

   En estas series se calculan, entre otros:
   - Mediana del log-precio y del precio (`med_log_price`, `med_price`).
   - Proxy de ocupación promedio (`occ_proxy`).
   - Número de anuncios activos (`listings`) y número de observaciones (`obs`) por alcaldía y periodo.

**Relación con gentrificación:**  
Este notebook construye la **infraestructura temporal** que permite observar cómo cambian:
- La cantidad de anuncios por alcaldía.
- Los precios medianos por periodo.
- La ocupación aproximada.

Estos insumos son clave para detectar posibles tendencias de gentrificación (p. ej., alzas persistentes de precio y densificación de anuncios en alcaldías específicas).

---

### 3.2 `UNION  BASES  DATOS.ipynb`

**Objetivo:**  
Unir bases históricas (2020–2021, 2023) con las más recientes, y generar estructuras consolidadas (especialmente de `calendar` y `listings`) para análisis de largo plazo.

**Pasos principales:**

1. **Lectura de datos históricos de CDMX (2020–2021, 2023)**  
   - `listings_2021`, `calendar_2021`, `reviews_2021`.
   - `listings_2023q1`–`listings_2023q4` y `reviews_2023q1`–`reviews_2023q4`.

2. **Integración de datasets guía**  
   - Lectura de `Proyecto Github Guia/listings.csv`, `calendar.csv`, `reviews.csv`, `neighbourhoods.csv`, etc., para comparar estructura y tipos de variables.

3. **Construcción de grandes tablas unificadas**  
   - Uso intensivo de `pd.merge` para unir:
     - `calendar` de distintos años y trimestres → `cal_all`.
     - `listings` unificados → `lis_all`.
   - Al final, se plantea una gran unión:
     ```python
     call_all = call_all.merge(
         lis_all,
         on=["snapshot", "listing_id"],
         how="left"
     )
     ```
     con el objetivo de obtener una tabla calendario–anuncio con información detallada por fecha, anuncio y periodo.

4. **Primeros intentos de series diarias, semanales y mensuales**  
   - Se hace referencia a archivos como `serie_diaria.csv`, `serie_semanal.csv`, `serie_mensual.csv`, que contienen agregaciones temporales de precios y/o ocupación.

**Relación con gentrificación:**  
Este notebook amplía el análisis hacia un **horizonte temporal más largo** (2020–2025). Esto permite:
- Ver **antes y después** de la pandemia en la oferta de Airbnb.
- Comparar la expansión de la plataforma en periodos recientes con años previos.
- Preparar una estructura robusta para comparar cambios en barrios centrales vs. periféricos a lo largo del tiempo.

---

### 3.3 `Preparacion_datos AIRBNB  2025 listing_18.ipynb`

**Objetivo:**  
Preparar un conjunto de datos limpio y filtrado para el **análisis exploratorio principal de 2025**.

**Pasos principales:**

- **Contexto del proyecto**  
  - Se formaliza “Proyecto 1: Análisis de datos Airbnb de la Ciudad de México”.
  - Se define la **meta**: explorar los archivos de datos y crear un dataset para análisis descriptivo y modelos posteriores.
  - Se establece explícitamente que los datos provienen de **Inside Airbnb** (listings y reviews de CDMX).

- **Carga y exploración de `UNION_18C.csv`**  
  - Se examina estructura, dimensiones y consistencia de variables.
  - Se genera una versión filtrada `listings_filtrado.csv`.

- **Limpieza y tratamiento de outliers**  
  - Se identifican valores atípicos en:
    - `price` (precios extremadamente altos, como valores de hasta $1,838,000).
    - `minimum_nights` (valores máximos muy altos, ~1125 noches mínimas).
  - Se filtra el dataset para eliminar estos extremos, dejando un subconjunto más realista de anuncios activos.

- **Reducción de muestras y columnas**  
  - Se pasa de ~227,000 observaciones iniciales a ~31,011 registros (≈13.65%) luego de aplicar filtros de calidad y disponibilidad.
  - Se mantienen únicamente las columnas relevantes para el análisis descriptivo posterior.

**Relación con gentrificación:**  
Este notebook define una **muestra depurada** de anuncios que refleja de manera más fiel el mercado activo de Airbnb en 2025. Al centrarse en:
- Anuncios con precios razonables,
- Estancias mínimas acordes a renta de corta/media estancia,

se construye una base más adecuada para estudiar cómo se manifiesta la gentrificación a partir de la oferta “realista” de vivienda turística.

---

### 3.4 `AIRBNB  2025 listing_18 .ipynb`

**Objetivo:**  
Responder preguntas de investigación específicas sobre la **distribución espacial** y la **estructura de precios** de Airbnb en la CDMX (corte 2025) usando el dataset filtrado.

**Preguntas de investigación:**

1. ¿Cuántos alojamientos hay por alcaldía?  
   ¿Cuál es la alcaldía con más alojamientos ofertados?
2. ¿Cómo es la distribución espacial?  
   ¿Cómo se concentran geográficamente los anuncios?
3. ¿Cuáles son los tipos de alojamiento?  
   ¿Cuál es el tipo más ofertado?
4. ¿Cómo se comporta la distribución de precios (histograma)?
5. ¿Cuál es el precio promedio por tipo de habitación en las diferentes alcaldías?
6. ¿Cómo se relacionan las variables numéricas?

**Pasos principales:**

- **Carga del dataset filtrado**  
  - Se usa `listings_filtrado.csv` como base principal.

- **Análisis por alcaldía (neighbourhood_cleansed)**  
  - Se contabilizan ≈ **192,653** alojamientos.  
  - Las alcaldías con mayor presencia de anuncios son:
    - **Cuauhtémoc**: ~87,254 anuncios.  
    - **Miguel Hidalgo**: ~32,129 anuncios.  
    - **Benito Juárez**: ~25,741 anuncios.
  - Esto confirma una **fuerte concentración** en zonas centrales y de alto valor inmobiliario.

- **Tipos de alojamiento**  
  - Predomina **Entire home/apt** (departamento o casa completa), seguido de:
    - Private room,
    - Shared room,
    - Hotel room.
  - La categoría “vivienda completa” es la más relevante para procesos de gentrificación (sustitución de alquiler residencial por alquiler turístico completo).

- **Distribución de precios**  
  - Después de limpiar outliers, los precios oscilan típicamente entre **$10** y alrededor de **$3,089** MXN por noche.
  - Se documentan y eliminan valores extremos que distorsionan la distribución (por ejemplo, precios cercanos a $1,838,000).

- **Precios promedio por tipo de habitación y alcaldía**  
  - Para **Entire home/apt**, los precios promedio por noche son más altos en alcaldías como:
    - **Miguel Hidalgo** (~$1,478),
    - **Cuajimalpa de Morelos** (~$1,418),
    - **Cuauhtémoc** (~$1,375).
  - Para **Private room**, también destacan:
    - Cuajimalpa de Morelos, Miguel Hidalgo y Cuauhtémoc con precios promedio significativamente mayores que el resto.
  - Para **Shared room** y **Hotel room**, los precios más altos se concentran nuevamente en Cuajimalpa de Morelos y algunas alcaldías centrales, con valores promedio por arriba de los $1,100 en habitaciones compartidas y alrededor de $1,600 en hotel.

- **Mapas de calor y distribución espacial**  
  - Se usan `folium` y `HeatMap` para visualizar la densidad de anuncios.
  - El mapa de calor muestra fuertes concentraciones en corredores turísticos y zonas de alto valor (Roma–Condesa–Juárez, Polanco, Santa Fe, etc.).
  - El análisis concluye que **no se observa una relación lineal simple** entre precio y localización en el mapa de calor; es probable que existan patrones no lineales y efectos de otras variables (amenidades, tipo de alojamiento, etc.).

**Relación con gentrificación:**  

- La **concentración masiva de anuncios completos** en Cuauhtémoc, Miguel Hidalgo y Benito Juárez, junto con precios promedio por noche mucho más altos, es consistente con las zonas identificadas en la literatura como focos de gentrificación.
- La combinación de:
  - alta densidad de Airbnb,
  - precios elevados,
  - y ubicación en zonas centrales / cercanas a infraestructura y equipamiento urbano,
  sugiere una **presión adicional sobre el mercado de renta tradicional**.

---

### 3.5 `EDA - K_MEANS AIRBNB  LISTING_GZ.ipynb`

**Objetivo:**  
Aplicar técnicas de **clustering** (K-Means y DBSCAN) sobre la base `UNION_GZ.csv` para encontrar **tipologías de anuncios** y patrones espaciales que puedan relacionarse con procesos de gentrificación.

**Pasos principales:**

- **Carga y preparación de datos**  
  - Se utiliza `UNION_GZ.csv` (una unión consolidada de distintos cortes).
  - Se seleccionan columnas relevantes, incluyendo:
    - `latitude`, `longitude`,
    - `accommodates`, `bedrooms`, `beds`,
    - `price`, y otras características de los anuncios.
  - Se construye un `ColumnTransformer` (`ct`) que:
    - Estandariza variables numéricas (`StandardScaler`).
    - Codifica variables categóricas mediante `OneHotEncoder` cuando corresponde.

- **Clustering con K-Means (conjunto amplio de variables)**  
  - Se define `X = ct.fit_transform(data.drop(columns='id'))`.
  - Se ajusta un modelo K-Means con un cierto número de clusters (por ejemplo, 43) y se evalúa el puntaje de silhouette.
  - Se exploran diferentes valores de `k` y se grafican los scores de silhouette.

- **Clustering con K-Means solo en latitud/longitud**  
  - Se construye `X2 = ct2.fit_transform(data[['latitude', 'longitude']])`.
  - Se ajusta K-Means con un número reducido de clusters espaciales (por ejemplo, 7 clusters).
  - Esto permite obtener clusters **puramente espaciales**, sin introducir precio ni otras variables.

- **Exploración de DBSCAN (comentado)**  
  - Aparece configurado un DBSCAN, aunque en el notebook el enfoque principal queda en K-Means.

**Relación con gentrificación:**  

- Los clusters basados en **coordenadas** (lat/long) permiten identificar:
  - Zonas centrales altamente densas (cluster de alta intensidad),
  - Zonas periféricas con baja presencia de Airbnb,
  - Corredores turísticos / de negocios específicos.
- Los clusters con más variables (precio, capacidad, etc.) tienden a agrupar:
  - Anuncios de **alto precio y alta capacidad** en zonas centrales,
  - Anuncios más económicos y pequeños en la periferia.
- Aunque el notebook no entra en una interpretación narrativa, estos clusters pueden interpretarse como **tipos de barrios turísticos**:
  - Clusters con altos precios y fuerte presencia de “Entire home/apt” son candidatos a **zonas en proceso de gentrificación o ya gentrificadas**.

---

### 3.6 `Factores _ Ganancia anual.ipynb`

**Objetivo:**  
Explorar qué factores están asociados a la **ganancia anual estimada** de los anuncios de Airbnb en CDMX y cómo se relacionan con la localización y el tipo de alojamiento.

**Pasos principales:**

- **Carga de datos**  
  - Se utiliza `UNION_GZ.csv` y una versión procesada `airbnb_cleaned.csv`.
  - Se eliminan columnas no necesarias para centrarse en:
    - `price`, `room_type`, `neighbourhood_cleansed`,
    - variables de reseñas,
    - número de amenidades (`total_amenities`), etc.

- **Limpieza y transformación**  
  - Conversión de `price` a numérico.
  - Tratamiento de outliers de precio mediante boxplots e IQR.

- **Construcción de métricas de rentabilidad**  
  - Aunque los detalles exactos están en el código, se manejan variables como:
    - `ganancia_anual` (o ganancia total),
    - proxies de demanda (número de reseñas 2020–2025, etc.).
  - Se generan gráficas como:
    - Relación número de reseñas vs. ganancia total.
    - Relación número de amenidades (`total_amenities`) vs. `ganancia_anual`, coloreando por tipo de habitación.

**Hallazgos generales (a nivel cualitativo):**

- Los anuncios con **más reseñas** tienden a mostrar una **mayor ganancia estimada**, sugiriendo que la demanda (ocupación) es un componente clave de la rentabilidad.
- La **cantidad de amenidades** (amenities) también se asocia positivamente con la ganancia: anuncios mejor equipados tienden a cobrar más y a ser más demandados.
- Las **viviendas completas** (Entire home/apt) concentran gran parte de la ganancia estimada, especialmente en las alcaldías centrales.

**Relación con gentrificación:**  

- La rentabilidad elevada en alcaldías centrales crea un **fuerte incentivo económico** para destinar vivienda al alquiler turístico en lugar de al alquiler residencial tradicional.
- Zonas donde la `ganancia_anual` es más alta son candidatas a experimentar:
  - Incremento en precios de renta,
  - Sustitución de residentes,
  - Cambios en el tejido comercial (más servicios orientados al turista).

---

## 4. Síntesis de resultados en clave de gentrificación

Integrando los seis notebooks, se pueden destacar los siguientes puntos:

1. **Concentración espacial de anuncios**  
   - La mayoría de los anuncios de Airbnb se concentran en:
     - **Cuauhtémoc**, **Miguel Hidalgo** y **Benito Juárez**, con decenas de miles de anuncios.
   - Estas alcaldías coinciden con:
     - Alta accesibilidad (transporte público, metro),
     - Oferta cultural y de ocio,
     - Zonas históricamente asociadas a población de ingresos medios-altos.

2. **Precios y tipos de alojamiento**  
   - Los **precios promedio por noche** son sistemáticamente más altos en las alcaldías centrales, sobre todo para:
     - **Entire home/apt**,
     - **Private room**,
     - **Hotel room** en zonas como Cuajimalpa de Morelos y Miguel Hidalgo.
   - La abundancia de anuncios de vivienda completa sugiere un **uso intensivo de unidades residenciales como alojamiento turístico**, un mecanismo clásico asociado a gentrificación.

3. **Evolución temporal (2020–2025)**  
   - Los paneles y series construidos (mensuales, trimestrales) permiten observar:
     - Crecimientos en el número de anuncios activos,
     - Cambios en precios medianos,
     - Patrones de ocupación aproximada por alcaldía.
   - Aunque el análisis temporal completo aún puede profundizarse, la infraestructura está lista para:
     - Detectar si, por ejemplo, tras 2021 se acelera el crecimiento de Airbnb en ciertas zonas.

4. **Clusters espaciales y tipologías de oferta**  
   - Los modelos de **K-Means** identifican clusters que diferencian:
     - Zonas centrales con vivienda completa, altos precios y muchas amenidades,
     - Zonas periféricas con anuncios más baratos y de menor capacidad.
   - Estos clusters pueden interpretarse como:
     - Corredores turísticos consolidados vs. zonas de expansión vs. periferia con baja presencia de Airbnb.

5. **Rentabilidad y presión sobre el mercado de vivienda**  
   - Los análisis de **ganancia anual** muestran que:
     - La combinación de altos precios + alta demanda (reseñas) + muchas amenidades genera **ganancias sustanciales** en barrios específicos.
   - Esto refuerza la hipótesis de que:
     - Airbnb puede aumentar la presión para convertir vivienda de alquiler tradicional en vivienda turística, contribuyendo a procesos de desplazamiento y cambio socio-espacial.

---

## 5. Conclusiones generales

1. **Airbnb está fuertemente concentrado en algunas alcaldías centrales de la CDMX**, especialmente Cuauhtémoc, Miguel Hidalgo y Benito Juárez, tanto en cantidad de anuncios como en precios promedio.

2. **La oferta de vivienda completa (Entire home/apt) domina el mercado**, lo que implica que no solo se oferta “habitaciones sueltas”, sino unidades enteras que, en otra situación, podrían estar en el mercado de renta de largo plazo.

3. **Las series de tiempo construidas (2020–2025) y los paneles mensuales/trimestrales** permiten seguir la expansión de Airbnb a través del tiempo y por alcaldía, creando la base para futuros modelos de:
   - Crecimiento de oferta,
   - Evolución de precios,
   - Identificación de “puntos de inflexión” asociados a gentrificación.

4. **Los clusters de K-Means y el análisis de rentabilidad** apuntan a la existencia de:
   - **Submercados turísticos** especializados en zonas centrales con alta rentabilidad,
   - **Submercados periféricos** con oferta más barata y menor densidad.

5. Aunque este proyecto no incorpora aún variables socio-demográficas (INEGI, renta de largo plazo, desplazamiento residencial), los patrones encontrados son **coherentes con la hipótesis de procesos de gentrificación** en las zonas con:
   - Alta densidad de anuncios,
   - Precios elevados,
   - Alta rentabilidad potencial.

---

## 6. Trabajo futuro

Para consolidar el análisis de gentrificación se propone:

- Integrar los paneles de Airbnb con:
  - Datos censales y encuestas de hogares (INEGI),
  - Información de precios de renta y venta de vivienda,
  - Indicadores socio-demográficos (ingreso, nivel educativo, composición de hogares).

- Desarrollar modelos espacio-temporales (series de tiempo por alcaldía + modelos de panel) que:
  - Relacionen explícitamente la expansión de Airbnb con cambios en renta y composición socioeconómica de los barrios.

- Refinar los clusters incorporando variables adicionales:
  - Distancia a estaciones de metro,
  - Zonas de interés turístico,
  - Cambios en uso de suelo.

---

## 7. Uso de este repositorio

Sugerencia de organización:

```text
.
├── data/
│   ├── raw/              # Datos originales descargados de Inside Airbnb
│   ├── processed/        # UNION.csv, UNION_GZ.csv, UNION_18C.csv, paneles y series
├── notebooks/
│   ├── 01_Limpieza_datos_24_25.ipynb
│   ├── 02_UNION_BASES_DATOS.ipynb
│   ├── 03_Preparacion_datos_listing_18.ipynb
│   ├── 04_AIRBNB_2025_listing_18_analisis.ipynb
│   ├── 05_EDA_KMEANS_LISTING_GZ.ipynb
│   └── 06_Factores_Ganancia_Anual.ipynb
└── README.md o docs/INFORME.md  # (este archivo)
