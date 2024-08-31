# Ejercicio 05: Evaluación de modelos de lenguaje grande mediante Azure Databricks y Azure OpenAI

## Objetivo
En este ejercicio, aprenderás a evaluar modelos de lenguaje grande (LLM) mediante Azure Databricks y el modelo GPT-4 de OpenAI. Esto incluye la configuración del entorno, la definición de métricas de evaluación y el análisis del rendimiento del modelo en tareas específicas.

## Requisitos
Una suscripción a Azure activa. Si no tiene una, puede registrarse para obtener una versión de [evaluación gratuita](https://azure.microsoft.com/en-us/free/).

## Paso 1: Aprovisionar Azure Databricks
- Inicia sesión en Azure Portal:
    1. Ve a Azure Portal e inicia sesión con tus credenciales.
- Creación de un servicio de Databricks:
    1. Ve a "Crear un recurso" > "Análisis" > "Azure Databricks".
    2. Escribe los detalles necesarios, como el nombre del área de trabajo, la suscripción, el grupo de recursos (crear nuevo o seleccionar existente) y la ubicación.
    3. Selecciona el plan de tarifa (elige estándar para este laboratorio).
    4. Selecciona "Revisar y crear" y, después, "Crear" una vez que se supere la validación.

## Paso 2: Iniciar área de trabajo y crear un clúster
- Iniciar el área de trabajo de Databricks
    1. Cuando se complete la implementación, ve al recurso y haz clic en "Iniciar área de trabajo".
- Crea un clúster de Spark:
    1. En el área de trabajo de Databricks, haz clic en "Proceso" en la barra lateral y, después, en "Crear proceso".
    2. Especifica el nombre del clúster y selecciona una versión en tiempo de ejecución de Spark.
    3. Elige el tipo de trabajo como "Estándar" y tipo de nodo en función de las opciones disponibles (elige nodos más pequeños para rentabilidad).
    4. Haz clic en "Crear proceso".

## Paso 3: Instalación de las bibliotecas necesarias

- Inicie sesión en su área de trabajo de Azure Databricks.
- Crea un nuevo cuaderno y selecciona el clúster predeterminado.
- Ejecuta los siguientes comandos para instalar las bibliotecas Python necesarias:

```python
%pip install openai
%pip install transformers
%pip install datasets
```

- Configura la clave de API de OpenAI:
    1. Agrega la clave de API de Azure OpenAI al cuaderno:

    ```python
    import openai
    openai.api_key = "your-openai-api-key"
    ```

## Paso 4: Definición de métricas de evaluación
- Definición de métricas de evaluación comunes:
    1. En este paso, definirás métricas de evaluación como Perplexity, la puntuación BLEU, la puntuación ROUGE y la precisión dependiendo de la tarea.

    ```python
    from datasets import load_metric

    # Example: Load BLEU metric
    bleu_metric = load_metric("bleu")
    rouge_metric = load_metric("rouge")

    def compute_bleu(predictions, references):
        return bleu_metric.compute(predictions=predictions, references=references)

    def compute_rouge(predictions, references):
        return rouge_metric.compute(predictions=predictions, references=references)
    ```

- Definición de métricas específicas de la tarea:
    1. En función del caso de uso, define otras métricas pertinentes. Por ejemplo, para el análisis de sentimiento, define la precisión:

    ```python
    from sklearn.metrics import accuracy_score

    def compute_accuracy(predictions, references):
        return accuracy_score(references, predictions)
    ```

## Paso 5: Preparación del conjunto de datos
- Carga de un conjunto de datos
    1. Usa la biblioteca de conjuntos de datos para cargar un conjunto de datos predefinido. Para este laboratorio, puedes usar un conjunto de datos básico, como el conjunto de datos de críticas de películas de IMDB para el análisis de sentimiento:

    ```python
    from datasets import load_dataset

    dataset = load_dataset("imdb")
    test_data = dataset["test"]
    ```

- Preprocesamiento de los datos
    1. Tokeniza y preprocesa el conjunto de datos para que sea compatible con el modelo GPT-4:

    ```python
    from transformers import GPT2Tokenizer

    tokenizer = GPT2Tokenizer.from_pretrained("gpt2")

    def preprocess_function(examples):
        return tokenizer(examples["text"], truncation=True, padding=True)

    tokenized_data = test_data.map(preprocess_function, batched=True)
    ```

##  Paso 6: Evaluación del modelo GPT-4
- Generación de predicciones:
    1. Usa el modelo GPT-4 para generar predicciones sobre el conjunto de datos de prueba

    ```python
    def generate_predictions(input_texts):
    predictions = []
    for text in input_texts:
        response = openai.Completion.create(
            model="gpt-4",
            prompt=text,
            max_tokens=50
        )
        predictions.append(response.choices[0].text.strip())
    return predictions

    input_texts = tokenized_data["text"]
    predictions = generate_predictions(input_texts)
    ```

- Cálculo de las métricas de evaluación
    1. Calcula las métricas de evaluación en función de las predicciones generadas por el modelo GPT-4

    ```python
    # Example: Compute BLEU and ROUGE scores
    bleu_score = compute_bleu(predictions, tokenized_data["text"])
    rouge_score = compute_rouge(predictions, tokenized_data["text"])

    print("BLEU Score:", bleu_score)
    print("ROUGE Score:", rouge_score)
    ```

    2. Si vas a evaluar una tarea específica, como el análisis de sentimiento, calcula la precisión

    ```python
    # Assuming binary sentiment labels (positive/negative)
    actual_labels = test_data["label"]
    predicted_labels = [1 if "positive" in pred else 0 for pred in predictions]

    accuracy = compute_accuracy(predicted_labels, actual_labels)
    print("Accuracy:", accuracy)
    ```

## Paso 7: Análisis e interpretación de los resultados

- Interpretación de los resultados
    1. Analiza las puntuaciones de precisión BLEU o ROUGE para determinar el rendimiento del modelo GPT-4 en la tarea.
    2. Discute las posibles razones de las discrepancias y considera formas de mejorar el rendimiento del modelo (por ejemplo, mediante el ajuste o con más preprocesamiento de datos).

- Visualización de los resultados
    1. Opcionalmente, puedes visualizar los resultados utilizando Matplotlib o cualquier otra herramienta de visualización.

    ```python
    import matplotlib.pyplot as plt

    # Example: Plot accuracy scores
    plt.bar(["Accuracy"], [accuracy])
    plt.ylabel("Score")
    plt.title("Model Evaluation Metrics")
    plt.show()
    ```

## Paso 8: Experimentación con diferentes escenarios

- Experimentación con solicitudes diferentes
    1. Modifica la estructura de las solicitudes para ver cómo afecta al rendimiento del modelo.

- Evaluación en diferentes conjuntos de datos
    1. Prueba a usar un conjunto de datos diferente para evaluar la versatilidad del modelo GPT-4 en varias tareas.

- Optimización de métricas de evaluación
    1. Experimenta con hiperparámetros, como la temperatura, tokens máximos, etc., para optimizar las métricas de evaluación.

## Paso 9: Limpieza de recursos
- Finalización del clúster:
    1. Vuelve a la página "Proceso", selecciona el clúster y haz clic en "Finalizar" para detener el clúster.

- Opcional: Eliminación del servicio de Databricks:
    1. Para evitar incurrir en cargos adicionales, considera la posibilidad de eliminar el área de trabajo de Databricks si este laboratorio no forma parte de un proyecto o una ruta de aprendizaje más grande.

Este ejercicio te guía por el proceso de evaluación de un modelo de lenguaje grande mediante Azure Databricks y el modelo GPT-4 de OpenAI. Al completar este ejercicio, obtendrás información sobre el rendimiento del modelo y comprenderás cómo mejorar y ajustar el modelo para tareas específicas.