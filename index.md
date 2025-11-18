---
layout: default
title: "Inicio"
---

# 👋 Hola, soy Ángel

Abogado en el sector bancario y Data Engineer certificado en Google Cloud.  
Aquí recopilo mis proyectos de:

- Pipelines ETL en Google Cloud (Dataflow, Data Fusion, Composer)
- IA generativa y Vertex AI aplicada a resoluciones judiciales
- Modelos de predicción de costas judiciales
- Automatización procesal y LegalTech

---

## 📝 Últimos post
{% for post in site.posts limit:5 %}
- [{{ post.title }}]({{ post.url }}) — <small>{{ post.date | date: "%d-%m-%Y" }}</small>
{% endfor %}

---

## 🔗 Enlaces

- [Sobre mí](/about)
- [GitHub](https://github.com/Gonzati)
