---
layout: post
title: "Clustering K-Means en Excel: aprendizaje no supervisado sin librerías externas"
date: 2025-12-16
author: "Ángel Argibay"
tags: [Excel, VBA, K-Means, Clustering, Machine Learning, Data Engineering]
---

<p align="center">
  <img src="/assets/images/portada-kmeans-excel.png" alt="Portada del artículo" style="max-width: 800px; width: 100%; border-radius: 6px;">
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
> En esta entrada implemento el algoritmo **K-Means** directamente en Excel usando **VBA**, aplicándolo sobre datos en formato *one-hot encoding*. Además, utilizo la **técnica del codo** para justificar el número óptimo de clusters. Todo el proceso es transparente, reproducible y sin usar Python, R ni librerías externas.

---

# 🧩 Introducción

Antes de entrar en la implementación práctica, conviene detenerse un momento en **qué es realmente K-Means y cómo funciona internamente**, más allá de la definición habitual.

**K-Means** es uno de los algoritmos más conocidos de **aprendizaje no supervisado** y se utiliza cuando:

- no existen etiquetas previas,  
- queremos descubrir patrones,  
- o necesitamos segmentar datos de forma automática.  

Su objetivo es sencillo de formular, pero interesante desde el punto de vista matemático:

> Agrupar observaciones en **K clusters** de forma que los elementos dentro de cada cluster sean lo más similares posible entre sí, y lo más distintos posible de los de otros clusters.

---

## 🔍 La intuición detrás de K-Means

El algoritmo parte de una idea muy simple:

1. Elegimos un número **K** de grupos.  
2. Colocamos **K puntos iniciales** llamados *centroides*.  
3. Cada observación se asigna al centro más cercano.  
4. Cada centro se mueve a la **media** de los puntos que tiene asignados.  
5. Repetimos el proceso hasta que los centros dejan de moverse.  

Aunque parezca trivial, este mecanismo iterativo genera estructuras sorprendentemente coherentes incluso en datasets complejos.

---

## 🔁 K-Means como proceso iterativo

Una de las claves para entender K-Means es que **no encuentra la solución de una sola vez**.  
Funciona por **aproximaciones sucesivas**:

- primero asigna,  
- luego corrige,  
- luego vuelve a asignar,  
- y así hasta converger.  

Este comportamiento se aprecia muy bien en la siguiente animación, donde se observa cómo los centroides se desplazan y cómo cambian las asignaciones en cada iteración:

<p align="center">
  <img src="/assets/images/K-means_convergence.gif" alt="Convergencia del algoritmo K-Means" style="max-width: 700px; width: 100%; border-radius: 6px;">
</p>

<p align="center">
  <em>
    By Chire - Own work, CC BY-SA 4.0,  
    <a href="https://commons.wikimedia.org/w/index.php?curid=59409335" target="_blank">
      https://commons.wikimedia.org/w/index.php?curid=59409335
    </a>
  </em>
</p>

En cada iteración ocurre lo siguiente:

- los puntos se reasignan al centro más cercano,  
- los centros se recalculan como la media de sus puntos,  
- las fronteras entre clusters se reajustan.  

El proceso continúa hasta que las asignaciones dejan de cambiar o el movimiento de los centroides es despreciable.

---

## 📐 El criterio matemático

Formalmente, K-Means intenta **minimizar la dispersión dentro de cada cluster**, lo que en la práctica equivale a minimizar la suma de las distancias cuadráticas de cada punto a su centroide.

Esta métrica es conocida como **SSE (Sum of Squared Errors)** y será clave más adelante cuando apliquemos la **técnica del codo** para elegir el número óptimo de clusters.

Dicho formalmente, el algoritmo busca minimizar la siguiente expresión:

SSE = Σᵢ Σₓ∈Cᵢ || x − μᵢ ||²

donde:

- **K** es el número de clusters
- **Cᵢ** es el conjunto de puntos asignados al cluster *i*
- **x** es una observación
- **μᵢ** es el centroide (media) del cluster *i*
- **|| x − μᵢ ||²** es la distancia euclídea al cuadrado

Esta magnitud, conocida como **SSE (Sum of Squared Errors)**, es la que se representa en la técnica del codo para justificar el número óptimo de clusters.
---

# 🔍 ¿Qué problema resolvemos?

El objetivo es agrupar observaciones en función de sus características, sin conocer previamente la categoría a la que pertenecen.

En el caso práctico del repositorio:

- cada fila representa una observación (por ejemplo, una sentencia, un documento o un registro),  
- cada columna representa una característica binaria (*one-hot encoding*), que en este caso son los motivos tratados en la sentencia, si representamos a demandante o demandado, y si la sentencia es favorable o desfavorable.  
- el algoritmo debe agrupar las filas en clusters coherentes según sus patrones.  

El enfoque es totalmente generalizable a cualquier otro dominio.

---

# 🧱 Implementación en Excel + VBA

La implementación se apoya en tres bloques fundamentales:

### ✔ Datos de entrada
- Matriz de datos (filas = observaciones, columnas = variables).  
- Normalmente en formato **one-hot encoding (0/1)**.

### ✔ Centroides
- Una fila por cluster.  
- Se inicializan manualmente o de forma arbitraria.  
- Se recalculan automáticamente en cada iteración.

### ✔ Asignación de clusters
- Una columna donde Excel escribe `C1`, `C2`, `C3`, etc.  
- Representa el cluster asignado a cada observación.

Todo el cálculo de distancias se hace mediante **distancia euclídea**, exactamente igual que en implementaciones estándar.

---

# 🔁 El algoritmo paso a paso (VBA)

El flujo interno del código es el siguiente:

1. Leer el rango de datos y el rango de centroides.  
2. Para cada fila:
   - calcular la distancia a cada centro,  
   - asignar el cluster más cercano.  
3. Comprobar si alguna asignación ha cambiado.  
4. Recalcular los centroides usando medias.  
5. Repetir hasta convergencia o máximo de iteraciones.  

El número de clusters **no está hardcodeado**:  
se deduce automáticamente del número de filas del rango de centroides.

Esto permite probar distintos valores de **K** sin modificar el código.

---

# 📐 La técnica del codo en Excel

Elegir **K** “a ojo” no es una buena práctica.  
Para justificar el número de clusters se aplica la **técnica del codo**.

El proceso es el siguiente:

1. Ejecutar K-Means para distintos valores de K (por ejemplo, de 1 a 10).  
2. Para cada K, calcular la **SSE (Sum of Squared Errors)**.  
3. Representar SSE frente a K en un gráfico.  
4. Identificar el punto donde la mejora empieza a ser marginal: el “codo”.  

Todo este proceso se realiza **dentro de Excel**, sin cálculos externos.

Así, una vez ejecutado el código, obtendremos la visualización correspondiente:

<p align="center">
  <img src="/assets/images/codo.png" alt="Visualización del codo" style="max-width: 800px; width: 100%; border-radius: 6px;">
</p>

<style>
.post-content p {
  text-align: justify;
}
.post-content li {
  text-align: justify;
}
</style>

En la imagen podemos observar que el número óptimo de clústers se encuentra entre 5 y 6.

# 📊 Aplicando K-Means sobre el dataset

Una vez que ejecutamos el código del módulo "clustering.bas" veremos que es le asigna un clúster a cada fila, en la columna "clúster":
<p align="center">
  <img src="/assets/images/dataset.png" alt="Asignación de cluster" style="max-width: 800px; width: 100%; border-radius: 6px;">
</p>

<style>
.post-content p {
  text-align: justify;
}
.post-content li {
  text-align: justify;
}
</style>

Y ya, con esta tabla creada, podemos visualizar la composición de los clústers, lo cual nos permite detectar a simple vista patrones, si estos existiesen:

<p align="center">
  <img src="/assets/images/clusters.png" alt="Visualización de los clusters" style="max-width: 800px; width: 100%; border-radius: 6px;">
</p>

<style>
.post-content p {
  text-align: justify;
}
.post-content li {
  text-align: justify;
}
</style>

Esto resulta especialmente útil en contextos exploratorios, donde el objetivo no es predecir, sino **entender la estructura interna de los datos** antes de tomar decisiones posteriores.

---

# 🖥️ Transparencia total (sin caja negra)

Una de las mayores ventajas de esta aproximación es que:

- puedes ver cada distancia calculada,  
- puedes inspeccionar cada centro,  
- puedes seguir cada iteración,  
- puedes validar manualmente los resultados.  

Esto convierte el proyecto en una herramienta **didáctica**, pero también en una solución perfectamente válida para producción en entornos limitados.

---

# 🧰 ¿Qué aporta este proyecto?

✔ Implementación real de K-Means desde cero  
✔ Comprensión profunda del algoritmo  
✔ Aplicable en entornos corporativos restrictivos  
✔ Justificación matemática del número de clusters  
✔ Reproducible, auditable y modificable  
✔ 100% Excel + VBA  

---

# 🧭 Limitaciones del enfoque

Como cualquier implementación de K-Means, este enfoque tiene algunas limitaciones:

- es sensible a la inicialización de los centroides  
- requiere fijar K a priori  
- funciona mejor con variables numéricas homogéneas  

Aun así, para análisis exploratorios y entornos restringidos, ofrece un equilibrio muy sólido entre interpretabilidad y utilidad práctica.

---

# 🚀 Conclusión

Este proyecto no pretende competir con scikit-learn ni con Spark.

Pretende algo distinto y, en muchos contextos, más valioso:  
**entender realmente qué está pasando cuando hacemos clustering**.

Implementar K-Means en Excel obliga a pensar en cada paso, cada cálculo y cada decisión.  
Y eso, a largo plazo, es lo que marca la diferencia entre *usar* modelos y *comprenderlos*.

Puedes encontrar todo el material, los libros de Excel y el código VBA en el repositorio:

[🔗 Repositorio del proyecto](https://github.com/Gonzati/clustering-kmeans-excel)

---
