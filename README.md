# Proyecto DM202501-PSet-3: Ingesta y OBT con Spark y Snowflake (NYC TLC Trips)

## 1. Resumen del Proyecto

Este proyecto implementa una infraestructura de procesamiento de datos con **Docker Compose** que orquesta un entorno **Jupyter+Spark** para la ingesta masiva y transformación de datos hacia **Snowflake**

El objetivo es replicar la ingesta del dataset **NYC TLC Trip Record Data (Yellow y Green, 2015-2025)** , aterrizándolo en un esquema `raw` en Snowflake. Posteriormente, se construye una tabla analítica desnormalizada única, la **One Big Table (OBT)** , llamada `analytics.obt_trips`, desde la cual se responden 20 preguntas de negocio mediante Spark. Todas las credenciales y parámetros se manejan estrictamente con variables de ambiente.

## 2. Arquitectura (Diagrama/Tabla)

[ t]La arquitectura sigue un flujo de procesamiento por lotes (batch) donde **Spark** actúa como el motor de ETL (Extract, Transform, Load) desde la fuente **Parquet** hacia el Data Warehouse **Snowflake**.

| Componente | Descripción |
| :--- | :--- |
| **Fuente de Datos** | NYC TLC Trip Record Data (Parquet, 2015-2025, Yellow/Green) |
| **Procesamiento** | `spark-notebook` (Jupyter+Spark)|
| **Destino** | Snowflake |
| **Esquema `raw`** | Aterrizaje espejo del origen + metadatos de ingesta  |
| **Esquema `analytics`** |`analytics.obt_trips` (One Big Table) para consumo analítico |
| **Flujo** | [ t]Parquet → Spark (backfill mensual) → Snowflake `raw` → Spark (enriquecimiento/unificación) → `analytics.obt_trips` (OBT)|

## 3. Variables de Ambiente y Parámetros (.env)

Todas las credenciales y parámetros provienen de variables de ambiente. Se prohíbe hardcodear credenciales en notebooks o Docker Compose.

**Listado Mínimo Obligatorio:** 

| Variable | Propósito | Categoría |
| :--- | :--- | :--- |
| `SNOWFLAKE_HOST` | Host o nombre de servicio de Snowflake | Snowflake |
| `SNOWFLAKE_USER` | Usuario de Snowflake | Snowflake |
| `SNOWFLAKE_PASSWORD` | Contraseña de Snowflake  | Snowflake |
| `SNOWFLAKE_DB` | Base de datos principal | Snowflake |
| `SNOWFLAKE_SCHEMA_RAW` | Esquema de aterrizaje (ej: `raw`)  | Snowflake |
| `SNOWFLAKE_SCHEMA_ANALYTICS` | Esquema analítico (ej: `analytics`)  | Snowflake |
| `PARQUET_BASE_URL` | Rutas/URL origen de los archivos Parquet | Parquet |
| `TARGET_YEARS` | Años a procesar | Parámetros |
| `TARGET_MONTHS` | Meses a procesar | Parámetros |
| `TARGET_SERVICES` | Servicios a procesar | Parámetros |
| `CHUNK_SIZE` | Parámetro para procesar un lote (mes) | Parámetros |
| `RUN_ID` | ID de la ejecución para auditoría/linaje  | Parámetros |

## 4. Pasos para Docker Compose y Ejecución de Notebooks

### 4.1. Infraestructura
1.  **Requisitos:** Docker y Docker Compose.
2.  **Configuración:** Crea el archivo `.env` (guía para env).
3.  **Levantar Servicios:**
    ```bash
    docker-compose up -d
    ```
    (El servicio `pyspark-notebook` expondrá los puertos 8888 (Jupyter) y 4040 (Spark UI)).

### 4.2. Orden y Propósito de Notebooks 
Los notebooks deben ejecutarse de forma secuencial:

1.  **`01_ingesta_parquet_raw.ipynb`**: Lee Parquet 2015-2025 (Yellow/Green) mes a mes. Escribe hacia `raw` y registra conteos por lote y metadatos.
2.  **`02_enriquecimiento_y_unificacion.ipynb`**: Integra Taxi Zones (nombres/borough) y unifica yellow/green, normalizando catálogos.
3.  **`03_construccion_obt.ipynb`**: Ensambla `analytics.obt_trips` con derivadas y metadatos. Verifica **idempotencia** reingestando un mes.
4.  **`04_validaciones_y_exploracion.ipynb`**: Valida nulos, rangos, coherencia de fechas, conteos por mes/servicio.
5.  **`05_data_analysis.ipynb`**: Contesta las 20 preguntas de negocio usando la OBT con Spark.

## 5. Diseño de `raw` y OBT (analytics.obt_trips) 

### 5.1. Diseño del Esquema `raw`
* **Estrategia de Tablas:** Tablas espejo por servicio y/o partición lógica año/mes (documentar elección).
* **Metadatos de Ingesta (Obligatorio):** `run_id`, `service_type`, `source_year`, `source_month`, `ingested_at_utc`, `source_path`, conteos por lote.
* **Idempotencia:** Clave natural definida por: `timestamps + PU/DO + VendorID`.

### 5.2. Diseño de la OBT: `analytics.obt_trips`
***Definición:** Modelo analítico desnormalizado que concentra todas las columnas necesarias para responder preguntas de negocio.
***Grano:** 1 fila = 1 viaje.
***Desnormalización:** Incluye nombres legibles (zona, borough, vendor, rate, payment) además de los IDs.

[ t]**Columnas Mínimas Obligatorias:** 
* **Tiempo:** `pickup_datetime`, `dropoff_datetime`, `pickup_date`, `pickup_hour`, `dropoff_date`, `dropoff_hour`, `day_of_week`, `month`, `year`.
* **Ubicación:** `pu_location_id`, `pu_zone`, `pu_borough`, `do_location_id`, `do_zone`, `do_borough`.
* **Servicio y Códigos:** `service_type` (yellow/green), `vendor_id`, `vendor_name`, `rate_code_id`, `rate_code_desc`, `payment_type`, `payment_type_desc`, `trip_type` (green).
* **Viaje/Tarifas:** `passenger_count`, `trip_distance`, `fare_amount`, `tip_amount`, `total_amount`, `congestion_surcharge`, etc.

**Derivadas y Reglas de Cálculo (Documentar Supuestos):** 
* **`trip_duration_min`**
* **`avg_speed_mph`**
* **`tip_pct`**

## 6. Calidad y Auditoría 

### 6.1. Calidad
* **Reglas Aplicadas:** Reglas mínimas (no nulos esenciales; distancias/duraciones $\ge0$; montos coherentes).
* **Documentación:** Documentar filas descartadas si se filtran outliers.
* **Validaciones:** Incluidas en `04_validaciones_y_exploracion.ipynb`.

### 6.2. Auditoría
* **Reporte de Auditoría:** Tabla o reporte con **conteos por servicio/año/mes**, tiempos de carga y `run_id`.

## 7. Matriz de Cobertura (2015-2025) 

| Servicio | Rango de Meses | Meses Totales | Meses Faltantes (Brechas) |
| :--- | :--- | :--- | :--- |
| **Yellow Cab** | Ene 2015 - Ago 2025 | 116 | NINGUNO |
| **Green Cab** | Ene 2015 - Ago 2025 | 115 | Mayo 2024 |
## 8. Checklist de Aceptación (DM202501-Pset-3) 

| Requisito | Estado |
| :--- | :--- |
| Docker Compose levanta Spark y Jupiter Notebook .  | ✅ |
| Todas las credenciales/parámetros provienen de variables de ambiente.  | ✅ |
| Cobertura 2015–2025 (Yellow/Green) cargada en raw con matriz y conteos por lote.  | ✅ |
| `analytics.obt_trips` creada con columnas mínimas, derivadas y metadatos.  | ✅ |
| Validaciones básicas documentadas (nulos, rangos, coherencia).  | ✅ |
| 20 preguntas respondidas (texto) usando la OBT..  | ✅ |
| README claro: pasos, variables, esquema, decisiones, troubleshooting.  | ✅ |