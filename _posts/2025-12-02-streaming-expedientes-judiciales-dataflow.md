---
layout: post
title: "Canalización en Streaming de Expedientes Judiciales con Dataflow"
date: 2025-12-02
author: "Ángel Argibay"
tags: [Dataflow, Streaming, BigQuery, PubSub, VertexAI, LegalTech]
---

<p align="center">
  <img src="/assets/images/portada-expedientes-dataflow.png" alt="Portada del artículo" style="max-width: 800px; width: 100%; border-radius: 6px;">
</p>

<style>
.post-content p {
  text-align: justify;
}
.post-content li {
  text-align: justify;
}
</style>

> **Resumen rápido:** Canalización en Streaming con Dataflow para registrar y visualizar actualizaciones en tiempo real de expedientes judiciales. Combinada con Vertex AI, esta arquitectura permite escalar hacia soluciones inteligentes sin intervención humana.

En la entrada de hoy os traigo una canalización en Streaming que registra cambios en los expedientes judiciales utilizando **Dataflow**. Los objetivos perseguidos con esta canalización son los siguientes:

- Seguimiento en tiempo real de los cambios en los expedientes.
- Reducción al mínimo del coste.
- Registro histórico de los cambios fácilmente accesible.
- Visualizar el estado de los expedientes mediante Looker Studio.

Además, esta canalización tiene un gran potencial si la combinamos con las posibilidades que aporta **Vertex AI**.

<p align="center">
  <img src="/assets/images/diagramadataflow.png" width="500"/>
</p>

Puedes ver el código completo del proyecto en GitHub aquí:  
[🔗 Repositorio del proyecto](https://github.com/Gonzati/gcp-expedientes-streaming)

## Componentes utilizados

**Pub/Sub** recibe los mensajes JSON con las actualizaciones de los expedientes. Cada JSON incluye:

- Referencia (única)
- Actualización del estado del procedimiento
- Una posible actualización de la cuantía
- Timestamps correspondientes

**BigQuery** se utiliza para registrar los datos de forma histórica y eficiente. Para ello se crean dos tablas:

- `Expedientes`

<p align="center">
  <img src="/assets/images/tablaexpedientes.png" width="500"/>
</p>

- `Expedientes_Staging` (con campo `ingestion_timestamp`)

<p align="center">
  <img src="/assets/images/Expedientes_Staging.png" width="500"/>
</p>


La tabla `Expedientes` contiene campos *Nested and Repeated* para cuantía y estado, lo que permite registrar múltiples cambios con sus respectivos timestamps.
Así vemos el resultado en BigQuery de la tabla Expedientes:

<p align="center">
  <img src="/assets/images/expedientes_poblados.png" width="500"/>
</p>

La tabla `Expedientes_Staging` es *append-only*. Cada 3 horas se lanza un `MERGE` optimizado para combinar los datos de staging con la tabla final, minimizando costes:

```sql
MERGE `Proyecto.Expedientes_OLAP.Expedientes` AS T
USING (
  SELECT
    T.Ref,
    ARRAY(
      SELECT AS STRUCT importe, timestamp
      FROM (
        SELECT c.importe, c.timestamp FROM UNNEST(T.Cuantia) AS c
        UNION DISTINCT
        SELECT c2.importe, c2.timestamp FROM UNNEST(S_cuantia) AS c2
      )
      ORDER BY timestamp
    ) AS merged_cuantia,
    ARRAY(
      SELECT AS STRUCT estado, timestamp
      FROM (
        SELECT e.estado, e.timestamp FROM UNNEST(T.Estado) AS e
        UNION DISTINCT
        SELECT e2.estado, e2.timestamp FROM UNNEST(S_estado) AS e2
      )
      ORDER BY timestamp
    ) AS merged_estado
  FROM `Proyecto.Expedientes_OLAP.Expedientes` AS T
  JOIN (
    SELECT
      Ref,
      ARRAY_CONCAT_AGG(Cuantia) AS S_cuantia,
      ARRAY_CONCAT_AGG(Estado)  AS S_estado
    FROM `Proyecto.Expedientes_OLAP.Expedientes_Staging`
    GROUP BY Ref
  ) AS S
  USING (Ref)
) AS M
ON T.Ref = M.Ref
WHEN MATCHED THEN
  UPDATE SET
    Cuantia = M.merged_cuantia,
    Estado  = M.merged_estado;
```

## Dataflow: conectando Pub/Sub con BigQuery

**Dataflow**:

- Lee los datos de Pub/Sub en tiempo real
- Parsea el JSON
- Escribe los datos en `Expedientes_Staging`
- Garantiza consistencia e idempotencia

<p align="center">
  <img src="/assets/images/dataflow.png" width="500"/>
</p>

## Dataset simulado para el ejercicio

Antes de lanzar el pipeline, se generaron 5.000 expedientes artificiales con un script de Python. Las secuencias de estados simuladas fueron:

**SECUENCIA_VERBAL**:

- Demanda → Contestación → (Vista 40%) → Sentencia → (Recurso 20%) → Oposición → Sentencia definitiva

**SECUENCIA_ORDINARIO**:

- Demanda → Contestación → Audiencia Previa → (Juicio 20%) → Sentencia → (Recurso 20%) → Oposición → Sentencia definitiva

Posteriormente, otro script simuló los eventos de Pub/Sub siguiendo estas secuencias.

## Ejecución del pipeline

```bash
--runner DataflowRunner
--project your-project
--region europe-west1
--temp_location gs://your-bucket/temp
--staging_location gs://your-bucket/staging
--input_topic projects/your-project/topics/expedientes-updates
```

## Visualización en Looker Studio

Looker Studio permite crear dashboards con métricas como:

- Distribución por tipo de procedimiento
- Evolución temporal de estados
- Carga de trabajo por juzgado

<p align="center">
  <img src="/assets/images/looker_expedientes.png" width="500"/>
</p>

## Potencial combinado con Vertex AI y otras soluciones

Este pipeline puede ampliarse con:

- **Vertex AI** para procesar notificaciones judiciales y generar automáticamente los JSON. [🔗 Clasificación Jurídica con VertexAI](https://gonzati.github.io/2025/11/25/clasificacion-juridica-vertexai.html)
- **Clasificación jurídica** de las resoluciones recibidas mediante modelos NLP.
- **Predicción de costas judiciales** usando modelos como los descritos en entradas anteriores. [🔗 Predicción de Costas con BigQuery](https://gonzati.github.io/2025/11/18/prediccion-de-costas-bigquery-ml.html)

Este enfoque permitiría automatizar por completo el seguimiento de expedientes desde que se recibe la notificación hasta su análisis predictivo.

➡️ Consulta el repositorio completo en GitHub para ver el código y la arquitectura. [🔗 Repositorio del proyecto](https://github.com/Gonzati/gcp-expedientes-streaming)

### Recursos adicionales

[🔗 Simplify historical data tracking in BigQuery with Datastream's append-only CDC](https://cloud.google.com/blog/products/data-analytics/understanding-datastream-append-only-mode/)

