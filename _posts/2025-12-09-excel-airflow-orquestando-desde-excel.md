---
layout: post
title: "Excel Airflow: un sistema completo de orquestación de datos en Excel + VBA"
date: 2025-12-09
author: "Ángel Argibay"
tags: [Excel, VBA, Data Engineering, ETL, Orquestación, LegalTech]
---

<p align="center">
  <img src="/assets/images/portada-excel-airflow.png" alt="Portada del artículo" style="max-width: 800px; width: 100%; border-radius: 6px;">
</p>

<style>
.post-content p {
  text-align: justify;
}
.post-content li {
  text-align: justify;
}
</style>

> **Resumen rápido:**  
> Excel Airflow es un sistema de orquestación inspirado en Apache Airflow, construido íntegramente en Excel + VBA para entornos donde no se permite instalar software externo. Permite crear DAGs, programar tareas, ejecutar pipelines, registrar logs y coordinar procesos de Access, Word y Excel desde un único panel de control.

---

# 🧩 Introducción

Durante años he trabajado en un entorno corporativo donde las restricciones técnicas impedían utilizar herramientas habituales en data engineering como Python, Airflow, Power Automate o servidores propios.

La necesidad, sin embargo, seguía existiendo:

- automatizar procesos repetitivos  
- coordinar tareas entre Access, Word y Excel  
- ejecutar pipelines completos  
- programar flujos diarios o recurrentes  
- registrar logs y tener trazabilidad  
- y hacerlo **sin instalar nada**  

De esa limitación nació **Excel Airflow**, un orquestador construido a mano en VBA, que hoy utilizo en producción y que ahora he refactorizado, anonimizado y publicado como proyecto técnico.

Esta entrada explica cómo funciona, cómo está diseñado y cómo puedes crear tus propios DAGs dentro del sistema.

---

# 🔧 ¿Qué es exactamente Excel Airflow?

Excel Airflow es un **motor de orquestación** compuesto por:

- un **scheduler**  
- un **dispatcher**  
- un **sistema visual de estado**  
- un **ejecutor de tareas externas** (Access, Word y Excel)  
- logs automáticos  
- soporte para programación tipo cron (`daily`, cada X minutos…)

Todo basado únicamente en:

- Excel (.xlsm)  
- Módulos VBA  
- Office interop  

Sin librerías externas, sin complementos y sin instalación.

---

# 🧱 Arquitectura general

Excel Airflow se compone de dos módulos principales:

### ✔ **AirflowCore / Módulo 1 – Ejecutor (Dispatcher)**  
- Identifica la tarea por su ID.  
- Marca en la interfaz su estado (amarillo/verde/rojo).  
- Llama a la subrutina correspondiente (Access, Word, Excel…).  
- Gestiona errores y rastrea tiempos.  

### ✔ **AirflowCore / Módulo 2 – Scheduler (Programación)**  
- Lee la periodicidad de cada fila (`daily`, `15`, `off`...).  
- Programa la próxima ejecución con `Application.OnTime`.  
- Relanza automáticamente los procesos.  
- Estandariza el funcionamiento tipo cron.  

---

# 🖥️ El panel de control (UI)

<p align="center">
  <img src="/assets/images/airflow-anon.jpg" alt="Panel Excel Airflow" style="max-width: 700px; width: 100%; border-radius: 6px;">
</p>

Cada fila representa:

- un proceso  
- un DAG  
- o un pipeline completo  

Con campos para:

- ID  
- Nombre  
- Descripción  
- Última ejecución  
- Estado  
- Programación  

---

# 🧠 Cómo funciona realmente un proceso

Cuando el usuario pulsa **EJECUTAR**:

1. Excel identifica el *procesoID* en la fila.  
2. El dispatcher entra en acción.  
3. Marca la celda de estado en amarillo.  
4. Ejecuta la tarea correspondiente (macro de Access, Word, Excel…).  
5. Cambia el estado a verde si todo sale bien, o rojo si hay error.  
6. El scheduler calcula la próxima ejecución si está programado.  

El flujo visual es casi idéntico al de Airflow “real”, pero en Excel.

---

# 🧩 Crear tu primer DAG

Excel Airflow permite crear DAGs en archivos `.bas` dentro de la carpeta `/DAGS/`.

Ejemplo de DAG totalmente anonimizado:

```vb
Sub Pipeline_NormalizacionYCarga()

    Call EjecutarTarea("LeerFuenteA", "N/A")
    Call EjecutarTarea("LimpiarFuenteA", "LeerFuenteA")
    Call EjecutarTarea("LeerFuenteB", "LimpiarFuenteA")
    Call EjecutarTarea("UnificarFuentes", "LeerFuenteB")
    Call EjecutarTarea("CargarEnStaging", "UnificarFuentes")

End Sub
```

Cada tarea se define como una macro independiente:

```vb
Sub LeerFuenteA()
    ' Lectura de fichero A
End Sub
```

Este sistema es extremadamente flexible: cualquier herramienta que se pueda automatizar con VBA es compatible.

---

# 📦 Un DAG real: limpieza, normalización y carga a staging

Incluyo un ejemplo totalmente anonimizado del DAG que publiqué en el repositorio.  
Este DAG:

- limpia dos ficheros  
- elimina duplicados  
- renombra columnas  
- los importa en Access  
- gestiona errores  
- elimina los ficheros si la importación es correcta  

```vb
Sub LimpiarYCargarFuentesAB()
    ' ... (código completo en el repositorio)
End Sub
```

Puedes ver la versión completa en:  
➡ [🔗 DAG de ejemplo](https://github.com/Gonzati/excel_airflow/blob/main/DAGS/EjemploDAG.bas)

---

# 🧰 ¿Qué permite hacer Excel Airflow?

✔ Crear orquestaciones reales complejas  
✔ Coordinar Access ↔ Excel ↔ Word ↔ Outlook  
✔ Ejecutar pipelines de limpieza, ETLs y cargas  
✔ Programar procesos diarios o recurrentes  
✔ Sustituir Airflow en entornos sin Python  
✔ Trazabilidad completa con logs  
✔ Integración fácil de nuevos módulos o DAGs  
✔ Funcionamiento sin dependencias externas  

Para entornos corporativos restrictivos, es un *game changer*.

---

# 🚀 Conclusión

Excel Airflow nació como una solución casera a un problema real:  
**cómo automatizar y orquestar procesos de datos cuando no puedes instalar nada**.

Hoy se ha convertido en uno de los proyectos que más orgulloso me hacen sentir, no solo por su utilidad práctica, sino porque ha acabado siendo un ejercicio de ingeniería artesanal, estable, mantenible y totalmente portable.
Es una solución mucho más potente de lo que a simple vista pueda parecer. Podemos utilizar SQL en access, y hay módulos que permiten parsear archivos JSON, por lo que prácticamente podemos hacer cualquier tarea en batch.

Si trabajas en un entorno con restricciones, o simplemente te interesa entender cómo funciona un orquestador desde dentro, te animo a explorar el repositorio:

[🔗 Repositorio del proyecto](https://github.com/Gonzati/excel_airflow)

---


