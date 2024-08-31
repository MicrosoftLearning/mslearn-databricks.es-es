# Ejercicio 04: Ajuste de modelos de lenguaje grande mediante Azure Databricks y Azure OpenAI

## Objetivo
Este ejercicio te guiará a través del proceso de ajuste de un modelo de lenguaje grande (LLM) mediante Azure Databricks y Azure OpenAI. Aprenderás a configurar el entorno, preprocesar datos y ajustar un LLM con datos personalizados para lograr tareas específicas de NLP.

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
- En la pestaña "Bibliotecas" del clúster, haz clic en "Instalar nueva".
- Instale los siguientes paquetes de Python:
    1. transformers
    2. conjuntos de datos
    3. azure-ai-openai
- Opcionalmente, también puedes instalar cualquier otro paquete necesario, como torch.

### Creación de un nuevo cuaderno
- Ve a la sección "Área de trabajo" y haz clic en "Crear" > "Cuaderno".
- Asigna un nombre al cuaderno (por ejemplo, Fine-Tuning-GPT4) y elige Python como lenguaje predeterminado.
- Adjunta el cuaderno a tu clúster.

## Paso 4: Preparación del conjunto de datos

- Carga del conjunto de datos
    1. Puedes usar cualquier conjunto de datos de texto adecuado para la tarea de ajuste. Por ejemplo, vamos a usar el conjunto de datos de IMDB para el análisis de sentimiento.
    2. En el cuaderno, ejecuta el código siguiente para cargar el conjunto de datos.

    ```python
    from datasets import load_dataset

    dataset = load_dataset("imdb")
    ```

- Preprocesamiento del conjunto de datos
    1. Tokeniza los datos de texto mediante el tokenizador de la biblioteca de transformadores.
    2. Agrega el código siguiente en tu cuaderno:

    ```python
    from transformers import GPT2Tokenizer

    tokenizer = GPT2Tokenizer.from_pretrained("gpt2")

    def tokenize_function(examples):
        return tokenizer(examples["text"], padding="max_length", truncation=True)

    tokenized_datasets = dataset.map(tokenize_function, batched=True)
    ```

- Preparar datos para el ajuste preciso
    1. Divide los datos en conjuntos de entrenamiento y validación.
    2. En el cuaderno, agrega:

    ```python
    small_train_dataset = tokenized_datasets["train"].shuffle(seed=42).select(range(1000))
    small_eval_dataset = tokenized_datasets["test"].shuffle(seed=42).select(range(500))
    ```

## Paso 5: Ajuste del modelo GPT-4

- Configuración de la API de OpenAI
    1. Necesitarás la clave de API y el punto de conexión de Azure OpenAI.
    2. En el cuaderno, configura las credenciales de la API:

    ```python
    import openai

    openai.api_type = "azure"
    openai.api_key = "YOUR_AZURE_OPENAI_API_KEY"
    openai.api_base = "YOUR_AZURE_OPENAI_ENDPOINT"
    openai.api_version = "2023-05-15"
    ```
- Ajuste de un modelo
    1. El ajuste de GPT-4 se realiza mediante la configuración de hiperparámetros y la continuación del proceso de entrenamiento en el conjunto de datos específico.
    2. El ajuste puede ser más complejo y puede requerir datos de procesamiento por lotes, personalizar bucles de entrenamiento, etc.
    3. Utiliza la siguiente plantilla básica:

    ```python
    from transformers import GPT2LMHeadModel, Trainer, TrainingArguments

    model = GPT2LMHeadModel.from_pretrained("gpt2")

    training_args = TrainingArguments(
        output_dir="./results",
        evaluation_strategy="epoch",
        learning_rate=2e-5,
        per_device_train_batch_size=2,
        per_device_eval_batch_size=2,
        num_train_epochs=3,
        weight_decay=0.01,
    )

    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=small_train_dataset,
        eval_dataset=small_eval_dataset,
    )

    trainer.train()
    ```
    4. Este código proporciona un marco básico para el entrenamiento. Los parámetros y conjuntos de datos deben adaptarse para casos concretos.

- Supervisión del proceso de entrenamiento
    1. Databricks permite supervisar el proceso de entrenamiento a través de la interfaz del cuaderno y herramientas integradas como MLflow para el seguimiento.

## Paso 6: Evaluación del modelo ajustado

- Generación de predicciones
    1. Después del ajuste, genera predicciones en el conjunto de datos de evaluación.
    2. En el cuaderno, agrega:

    ```python
    predictions = trainer.predict(small_eval_dataset)
    print(predictions)
    ```

- Evaluación del rendimiento del modelo
    1. Puedes usar métricas como la exactitud, precisión, coincidencia y puntuación F1 para evaluar el rendimiento del modelo.
    2. Ejemplo:

    ```python
    from sklearn.metrics import accuracy_score

    preds = predictions.predictions.argmax(-1)
    labels = predictions.label_ids
    accuracy = accuracy_score(labels, preds)
    print(f"Accuracy: {accuracy}")
    ```

- Guardar el modelo ajustado
    1. Guarda el modelo ajustado en el entorno de Azure Databricks o Azure Storage para su uso futuro.
    2. Ejemplo:

    ```python
    model.save_pretrained("/dbfs/mnt/fine-tuned-gpt4/")
    ```

## Paso 7: Implementación del modelo ajustado
- Empaquetado de un modelo para la implementación
    1. Convierte el modelo en un formato compatible con Azure OpenAI u otro servicio de implementación.

- Implementación del modelo
    1. Usa Azure OpenAI para la implementación mediante el registro del modelo a través de Azure Machine Learning o directamente con el punto de conexión de OpenAI.

- Prueba del modelo implementado
    1. Ejecuta pruebas para asegurarte de que el modelo implementado se comporta según lo previsto y se integra sin problemas con las aplicaciones.

## Paso 8: Limpieza de recursos
- Finalización del clúster:
    1. Vuelve a la página "Proceso", selecciona el clúster y haz clic en "Finalizar" para detener el clúster.

- Opcional: Eliminación del servicio de Databricks:
    1. Para evitar incurrir en cargos adicionales, considera la posibilidad de eliminar el área de trabajo de Databricks si este laboratorio no forma parte de un proyecto o una ruta de aprendizaje más grande.

En este ejercicio se proporciona una guía completa sobre el ajuste de modelos de lenguaje grandes como GPT-4 mediante Azure Databricks y Azure OpenAI. Siguiendo estos pasos, podrás ajustar los modelos para tareas específicas, evaluar su rendimiento e implementarlos para aplicaciones reales.

