---
title: Instrucciones hospedadas en línea
permalink: index.html
layout: home
---

# Ejercicios de Azure Databricks

Estos ejercicios están diseñados para admitir el siguiente contenido de formación de Microsoft Learn:

- [Implementación de una solución de análisis de datos con Azure Databricks](https://learn.microsoft.com/training/paths/data-engineer-azure-databricks/)
- [Implementación de una solución de Azure Machine Learning con Azure Databricks](https://learn.microsoft.com/training/paths/build-operate-machine-learning-solutions-azure-databricks/)

Para realizar estos ejercicios, necesitará una suscripción a Azure con acceso de administrador.

{% assign exercises = site.pages | where_exp:"page", "page.url contains '/Instructions/Exercises'" %} {% for activity in exercises  %}
- [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) | {% endfor %}