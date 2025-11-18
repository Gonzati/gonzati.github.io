---
layout: post
title: "Predicción de costas judiciales con BigQuery ML"
date: 2025-11-18
---
![Portada del artículo](/assets/images/portada-prediccion-costas.png)
Uno de los principales problemas de la litigación masiva —y también de la no tan masiva— es **estimar con antelación las costas que pueden generarse si se pierde un procedimiento**. La dificultad se ha agudizado en los últimos años por el **abandono progresivo de los baremos tradicionales** utilizados en las tasaciones.

En esta entrada explico cómo construí un **modelo de regresión en BigQuery ML**, cómo diseñé una **canalización incremental en Dataform**, y cómo generé un flujo completo que permite a un equipo de analistas **obtener predicciones actualizadas de manera automática**, usando solo SQL.

Todas las piezas del proyecto están disponibles en este repositorio:  
👉 https://github.com/Gonzati/prediccion_de_costas_pipeline_dataform

---

# 🔍 1. ¿Es posible predecir las costas judiciales?

La pregunta inicial fue doble:

- **¿Es técnicamente posible predecir las costas?**  
- **En caso afirmativo, ¿podemos sistematizar su predicción en un pipeline reproducible?**

La respuesta corta es: **sí**.

Existe una **relación estadísticamente muy fuerte entre la cuantía reclamada y las costas generadas**. Esa correlación permite utilizar un **modelo de regresión lineal sencillo**, sin necesidad de recurrir a AutoML, para alcanzar una precisión más que razonable.

En esta demostración utilicé **datos sintéticos** (5.000 filas) generados únicamente para documentar el procedimiento.

---

# 📊 2. El dataset de origen

El dataset consistía en un CSV con las siguientes columnas:

- `CUANTIA`
- `COSTAS`
- `FECHA_SENTENCIA`
- `FECHA_COBRO`

Con estas variables es posible predecir:

1. **Cuánto** pagaremos de costas (modelo de regresión)
2. **Cuándo** se producirá la tasación (aprox. 86 días tras la sentencia en la simulación)

Tras subir el CSV al bucket de origen, creé un dataset en BigQuery llamado "Modelo_Costas", y la Tabla "Modelo_costas.datos"


---

# 🤖 3. Entrenamiento del modelo en BigQuery ML

No utilicé AutoML porque ya conocía la relación entre cuantía y costas. Opté por un enfoque transparente:

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

El modelo se entrenó correctamente, mostrando unas métricas sólidas y coherentes con el comportamiento esperado en datos reales.

Con esto ya teníamos un modelo funcional y evaluado, accesible desde SQL mediante ML.PREDICT.

🔧 4. Una canalización SQL-first para analistas: Dataform

Imaginemos que distintos analistas deben generar predicciones continuamente sobre nuevos casos. Para ellos, lo ideal es una herramienta donde pudieran construir transformaciones usando exclusivamente SQL.

La respuesta natural es Dataform.

Construí un ejemplo de canalización incremental:
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
  CAST(FARM_FINGERPRINT(CONCAT(CAST(CUANTIA AS STRING), '|', CAST(FECHA_SENTENCIA AS STRING))) AS INT64) AS row_id,
  ROUND(CUANTIA, 2) AS CUANTIA,
  FECHA_SENTENCIA,
  DATE_ADD(FECHA_SENTENCIA, INTERVAL 86 DAY) AS FECHA_COBRO,
  ROUND(LEAST(50000.0, GREATEST(1.0, predicted_COSTAS)), 2) AS COSTAS_PREDICHAS
FROM pred;
Este script:

- Calcula predicciones con ML.PREDICT

- Genera un identificador único por fila

- Calcula una fecha estimada de cobro

- Limita valores extremos

- Inserta solo nuevas filas mediante incremental

📈 5. Visualización en Looker Studio

Una vez creada la tabla de predicciones en BigQuery, solo quedaba conectarla a Looker Studio.

El resultado era un dashboard sencillo pero funcional, donde se podían analizar:

Cuantías

Fechas de sentencia

Predicciones de costas

Tendencias por mes de cobro

Ideal para analistas acostumbrados a consumir datos de manera visual.

🧩 6. Resultado: un pipeline end-to-end simple y eficaz

Con muy pocas herramientas:

BigQuery ML

Dataform

Looker Studio

…se puede construir una solución automática y escalable que permita predecir costes futuros de litigación y alimentar decisiones de negocio sin necesidad de herramientas externas ni Python.

El valor clave de este enfoque es que todo se ejecuta con SQL, lo que permite que equipos no familiarizados con frameworks complejos puedan operar, mantener y extender la solución.

En futuras entradas documentaré variantes del modelo, el uso de otras features y la integración con canalizaciones orquestadas mediante Composer.

