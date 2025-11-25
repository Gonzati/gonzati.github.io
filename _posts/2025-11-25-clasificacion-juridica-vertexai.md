---
layout: post
title: "Clasificación jurídica híbrida con Vertex AI, Data Fusion y procesamiento on-premise"
date: 2025-11-25
author: "Ángel Argibay"
tags: [VertexAI, DataFusion, PubSub, CloudComposer, LegalTech, ETL, GCP]
---

<p align="center">
  <img src="/assets/images/portada-clasificacion-vertex.png" alt="Portada del artículo" style="max-width: 800px; width: 100%; border-radius: 6px;">
</p>

<style>
.post-content p {
  text-align: justify;
}
.post-content li {
  text-align: justify;
}
</style>

> **Resumen rápido:** Pipeline híbrido donde Vertex AI clasifica resoluciones judiciales, Pub/Sub coordina un servidor on-premise que ejecuta notebooks, Cloud Composer orquesta el flujo y Data Fusion integra los resultados en BigQuery para su explotación en Looker Studio.

---

## 1️⃣ ¿Qué problema queremos resolver?

En este ejercicio quiero mostrar cómo clasificar **centenares de resoluciones judiciales** aprovechando Vertex AI y, al mismo tiempo, ejecutar parte del procesamiento **en local**.

El proyecto combina:

- IA generativa para extraer *insights* jurídicos  
- ejecución distribuida (Cloud ↔ On-Premise)  
- un pipeline orquestado con Composer  
- integración en BigQuery y visualización en Looker Studio  

Para la prueba utilicé **400 resoluciones públicas del CENDOJ**, relacionadas con derecho bancario y financiero.

Puedes ver el código completo del proyecto en GitHub aquí:  
[🔗 Repositorio del proyecto](https://github.com/Gonzati/Clasificacion-juridica-Vertex-AI-Data-Fusion)

---

## 2️⃣ ¿Qué aporta Vertex AI?

Gemini nos permite:

- Clasificar **motivos jurídicos** (multietiqueta)  
- Identificar el **demandado**  
- Determinar si la sentencia es **favorable o desfavorable**  
- Obtener resultados estructurados en minutos  

En mi caso uso dos notebooks:

- `demandadoyresultadogcp.ipynb`  
- `etiquetavertexgcp.ipynb`  

Ambos generan un CSV independiente que luego fusiono.

---

## 3️⃣ Arquitectura completa del pipeline

La arquitectura combina varios servicios:

- **Cloud Storage** — Origen de resoluciones y destino de CSV procesados  
- **Cloud Functions** — Trigger que lanza Composer al detectar nuevos `.txt`  
- **Cloud Composer** — Orquestación central del pipeline  
- **Pub/Sub** — Comunicación cloud ↔ home lab  
- **Servidor on-premise (Jupyter + Papermill)** — Ejecuta los notebooks  
- **Data Fusion** — Une los CSV y hace carga incremental en BigQuery  
- **BigQuery** — Data Warehouse  
- **Looker Studio** — Visualización final  

<p align="center">
  <img src="/assets/images/esquema.png" width="500"/>
</p>

---

## 4️⃣ Cloud Functions — Activando el DAG de Composer

Cada vez que aparece un nuevo `.txt` en el bucket origen, Cloud Functions lanza el DAG `run_notebooks_and_datafusion`.

### `main.py`
```python
import os, json, time
import functions_framework
from cloudevents.http import CloudEvent

from google.auth import default
from google.auth.transport.requests import AuthorizedSession

PROJECT  = os.environ["GCP_PROJECT"]
LOCATION = os.environ["LOCATION"]         # us-east4
ENV      = os.environ["COMPOSER_ENV"]     # composer-rag
DAG_ID   = os.environ["DAG_ID"]           # run_notebooks_and_datafusion

def _authed_session():
    creds, _ = default(scopes=["https://www.googleapis.com/auth/cloud-platform"])
    return AuthorizedSession(creds)

def _composer_base():
    return f"https://composer.googleapis.com/v1/projects/{PROJECT}/locations/{LOCATION}/environments/{ENV}"

@functions_framework.cloud_event
def trigger_dag(event: CloudEvent):
    data = event.data or {}
    bucket = data.get("bucket")
    name   = data.get("name")
    if not bucket or not name:
        print("Evento sin bucket/name; nada que hacer.")
        return "OK", 204

    dag_conf = {"bucket": bucket, "name": name}

    session = _authed_session()
    base = _composer_base()

    body = {
        "command": "dags",
        "subcommand": "trigger",
        "parameters": [DAG_ID, "--conf", json.dumps(dag_conf)]
    }
    r = session.post(f"{base}:executeAirflowCommand", json=body)
    print("executeAirflowCommand:", r.status_code, r.text)
    if r.status_code >= 300:
        raise RuntimeError(f"Composer API error (execute): {r.text}")

    exec_id = r.json().get("executionId")
    if not exec_id:
        return "OK", 200

    for _ in range(6):
        pr = session.post(f"{base}:pollAirflowCommand", json={"executionId": exec_id})
        print("pollAirflowCommand:", pr.status_code, pr.text)
        if pr.json().get("done"):
            break
        time.sleep(5)

    return "OK", 200
```

### `requirements.txt`
```text
functions-framework==3.*
google-auth==2.*
google-auth-oauthlib==1.*
```

---

## 5️⃣ Cloud Composer — El corazón de la orquestación

Composer realiza:

1. Generación de un `correlation_id`.  
2. Envío de un mensaje a Pub/Sub solicitando ejecutar los notebooks.  
3. Espera del mensaje `"done"`.  
4. Ejecución del pipeline de Data Fusion.

Aquí un extracto del DAG:

```python
with DAG(
    dag_id="run_notebooks_and_datafusion",
    description="Lanza los notebooks vía Pub/Sub y luego ejecuta el pipeline de Data Fusion.",
    start_date=datetime(2025, 11, 6),
    schedule_interval=None,
    catchup=False,
    tags=["rag", "datafusion"],
) as dag:

    start = EmptyOperator(task_id="start")

    def _make_correlation_id(**context):
        corr = f"corr-{uuid.uuid4()}"
        context["ti"].xcom_push(key="correlation_id", value=corr)
        return corr

    make_correlation_id = PythonOperator(
        task_id="make_correlation_id",
        python_callable=_make_correlation_id,
    )

    def _build_messages(**context):
        ti = context["ti"]
        corr = ti.xcom_pull(task_ids="make_correlation_id", key="correlation_id")
        body = {"command": "run_notebook_batch", "ts": time.time()}
        attrs = {"correlation_id": corr, "pipeline": "rag-notebooks"}
        ti.xcom_push(key="pubsub_message_body", value=body)
        ti.xcom_push(key="pubsub_message_attrs", value=attrs)

    build_messages = PythonOperator(
        task_id="build_messages",
        python_callable=_build_messages,
    )
```

---

## 6️⃣ Procesamiento on-premise — Ejecución de notebooks con Papermill

### `run_notebooks_batch.sh`
```bash
#!/usr/bin/env bash
set -euo pipefail

source ~/jupyter_env/bin/activate
echo "Usando entorno virtual: $VIRTUAL_ENV"

NB_DIR=~/jupyter_env/trabajo/pipeline
TS=$(date +%s)

OUT1="/tmp/demandadoyresultadogcp_out_${TS}.ipynb"
OUT2="/tmp/etiquetavertexgcp_out_${TS}.ipynb"

papermill "$NB_DIR/demandadoyresultadogcp.ipynb" "$OUT1"
papermill "$NB_DIR/etiquetavertexgcp.ipynb"      "$OUT2"

echo "Batch OK: $OUT1 y $OUT2"
```

### `publish_marker.py`
```python
from google.cloud import pubsub_v1

project_id = "xxxx"
topic_id = "notebook-finished"
message = "done"

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path(project_id, topic_id)

future = publisher.publish(topic_path, message.encode("utf-8"))
print(f"✅ Marker publicado en Pub/Sub: {future.result()}")
```

---

## 7️⃣ Data Fusion — Fusión de CSV y carga en BigQuery

Cuando Composer recibe el `"done"`, activa el pipeline de Data Fusion.  
Este:

- Une los dos CSV mediante el nombre de la resolución  
- Inserta los datos en BigQuery (carga incremental)  
- Mueve los archivos procesados a un bucket “procesadas”  
- Archiva los CSV en un bucket de almacenamiento  

<p align="center">
  <img src="/assets/images/datafusion2.png" width="500"/>
</p>


---

## 8️⃣ Visualización en Looker Studio

Con los datos ya en BigQuery, Looker Studio permite visualizar:

- Motivos de la sentencia  
- Demandados  
- Resoluciones favorables/desfavorables  
- Estadísticas por rango temporal  
- Tendencias por motivo jurídico  

---

## 🧩 9. Resultado: un pipeline híbrido totalmente funcional

Este ejercicio demuestra:

- Integración cloud ↔ on-premise con **Pub/Sub**  
- Orquestación avanzada con **Cloud Composer**  
- Uso realista de **Vertex AI** para análisis jurídico  
- ETL empresarial sin código en **Cloud Data Fusion**  
- Exploración analítica en **BigQuery + Looker Studio**

Una arquitectura elegante, modular y ampliable que permite analizar resoluciones judiciales a escala combinando IA y procesamiento distribuido.
