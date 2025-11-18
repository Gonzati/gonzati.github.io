---
layout: post
title: "Predicción de costas judiciales con BigQuery ML y Dataform"
date: 2025-11-18
---

> **TL;DR**: uso BigQuery ML para entrenar un modelo de regresión lineal que predice costas judiciales a partir de la cuantía, y Dataform para montar una canalización incremental 100% SQL que genera predicciones listas para explotar en Looker Studio.

---

## 1️⃣ El problema: ¿cuánto me van a condenar en costas?

Uno de los principales problemas de la litigación masiva (y no tan masiva) es poder realizar una **previsión razonable de las costas** que se pueden generar en caso de perder un procedimiento.

Este problema se agrava por el **abandono progresivo de los baremos** tradicionales que se utilizaban en las tasaciones.

Aquí me planteé dos preguntas:

- ¿Es técnicamente posible predecir las costas que se van a generar?
- Y si es posible, ¿podemos **sistematizar** esa predicción en un pipeline reproducible?

La respuesta es **sí** a ambas.

---

## 2️⃣ Dataset utilizado

Para este ejercicio he utilizado un CSV con datos **sintéticos** (5.000 filas) con estas columnas:

- `CUANTIA`
- `COSTAS`
- `FECHA_SENTENCIA`
- `FECHA_COBRO`

Con esto podemos estimar:

- **Cuánto** pagaremos de costas (`COSTAS`)
- **Cuándo** se practicará la tasación (`FECHA_COBRO`)

El CSV se carga en un bucket de Cloud Storage y desde ahí en una tabla de BigQuery:

- Dataset: `Modelo_costas`
- Tabla: `Modelo_costas.datos`

---

## 3️⃣ Entrenando el modelo en BigQuery ML

Como ya conocía que la relación es básicamente lineal, en lugar de usar AutoML opté por un **modelo de regresión lineneal explícito** con BigQuery ML:

```sql
CREATE OR REPLACE MODEL `Modelo_costas.modelo_costas_lr`
OPTIONS (
  model_type = 'linear_reg',
  input_label_cols = ['COSTAS'],
  data_split_method = 'RANDOM',
  data_split_eval_fraction = 0.20,
  category_encoding_method = 'DUMMY_ENCODING',
  calculate_p_values = TRUE
) AS
SELECT
  CUANTIA,
  COSTAS
FROM `Modelo_costas.datos`;
```

Con esto BigQuery:

- Hace un **split 80/20** entrenamiento / evaluación  
- Ajusta un modelo de regresión lineal clásico  
- Calcula **p-values** para evaluar la significancia de las variables  

El resultado es un modelo con un ajuste muy razonable para un caso tan simple.

---

## 4️⃣ Canalización SQL-first con Dataform

Imaginemos ahora que las predicciones las van a consumir **analistas que se manejan muy bien con SQL**, pero no quieren entrar en Dataflow, Python, etc.

Aquí entra en juego **Dataform**, que permite definir transformaciones y tablas derivadas únicamente con SQL.

A modo de ejemplo, esta es la tabla incremental `predicciones_detalle`:

```sql
config {
  type: "incremental",
  name: "predicciones_detalle",
  database: "rag-vertex-477211",
  schema: "Modelo_costas",
  uniqueKey: ["row_id"],
  bigquery: {
    partitionBy: "DATE_TRUNC(FECHA_COBRO, MONTH)",
    clusterBy: ["FECHA_COBRO", "FECHA_SENTENCIA"]
  },
  tags: ["predict"]
}

WITH src AS (
  SELECT
    CUANTIA,
    FECHA_SENTENCIA
  FROM `rag-vertex-477211.Modelo_costas.nuevos_casos_ext`
  WHERE CUANTIA IS NOT NULL AND FECHA_SENTENCIA IS NOT NULL
),

pred AS (
  SELECT
    CUANTIA,
    FECHA_SENTENCIA,
    predicted_COSTAS
  FROM ML.PREDICT(
    MODEL `rag-vertex-477211.Modelo_costas.modelo_costas_lr`,
    TABLE src
  )
)

SELECT
  CAST(
    FARM_FINGERPRINT(
      CONCAT(CAST(CUANTIA AS STRING), '|', CAST(FECHA_SENTENCIA AS STRING))
    ) AS INT64
  ) AS row_id,
  ROUND(CUANTIA, 2) AS CUANTIA,
  FECHA_SENTENCIA,
  DATE_ADD(FECHA_SENTENCIA, INTERVAL 86 DAY) AS FECHA_COBRO,
  ROUND(LEAST(50000.0, GREATEST(1.0, predicted_COSTAS)), 2) AS COSTAS_PREDICHAS
FROM pred;
```

---

## 5️⃣ Visualización en Looker Studio

Una vez creada la tabla de predicciones en BigQuery, solo quedaba conectarla a Looker Studio.

El resultado es un dashboard sencillo pero funcional, donde se pueden analizar:

- Cuantías  
- Fechas de sentencia  
- Predicciones de costas  
- Tendencias por mes de cobro  

Ideal para analistas acostumbrados a consumir datos de manera visual.

---

## 🧩 6. Resultado: un pipeline end-to-end simple y eficaz

Con muy pocas herramientas:

- **BigQuery ML**  
- **Dataform**  
- **Looker Studio**

…se puede construir una solución **automática y escalable** que permita predecir costes futuros de litigación y alimentar decisiones de negocio sin necesidad de herramientas externas ni Python.

El valor clave de este enfoque es que **todo se ejecuta con SQL**, lo que permite que equipos no familiarizados con frameworks complejos puedan operar, mantener y extender la solución.

En futuras entradas documentaré variantes del modelo, el uso de otras features y la integración con canalizaciones orquestadas mediante Composer.
---
