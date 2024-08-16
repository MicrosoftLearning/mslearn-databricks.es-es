# Ejercicio 02: Generación aumentada de recuperación con Azure Databricks

## Objetivo
Este ejercicio te guía a través de la configuración de un flujo de trabajo de generación aumentada de recuperación (RAG) en Azure Databricks. El proceso implica la ingesta de datos, la creación de incrustaciones de vectores, el almacenamiento de estas incrustaciones en una base de datos vectorial y su uso para aumentar la entrada para un modelo generativo.

## Requisitos
Una suscripción a Azure activa. Si no tiene una, puede registrarse para obtener una versión de [evaluación gratuita](https://azure.microsoft.com/en-us/free/).

## Tiempo estimado: 40 minutos.

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

## Paso 3: Preparación de datos
- Ingerir datos
    1. Descarga un conjunto de datos de ejemplo de artículos de [aquí](https://dumps.wikimedia.org/enwiki/latest/).
    2. Carga el conjunto de datos en Azure Data Lake Storage o directamente en el sistema de archivos de Azure Databricks.

- Carga de datos en Azure Databricks
```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("RAG-DataPrep").getOrCreate()
raw_data_path = "/mnt/data/wiki_sample.json"  # Adjust the path as necessary

raw_df = spark.read.json(raw_data_path)
raw_df.show(5)
```

- Limpieza y preprocesamiento de datos
    1. Limpia y preprocesa los datos para extraer los campos de texto pertinentes.

    ```python
    from pyspark.sql.functions import col

    clean_df = raw_df.select(col("title"), col("text"))
    clean_df = clean_df.na.drop()
    clean_df.show(5)
    ```
## Paso 4: Generación de incrustaciones
- Instalar las bibliotecas necesarias
    1. Asegúrate de que tienes instalados los transformadores y las bibliotecas de transformadores de frases.

    ```python
    %pip install transformers sentence-transformers
    ```
- Generación de incrustaciones
    1. Usa un modelo entrenado previamente para generar incrustaciones para el texto.

    ```python
    from sentence_transformers import SentenceTransformer

    model = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')

    def embed_text(text):
        return model.encode(text).tolist()

    # Apply the embedding function to the dataset
    from pyspark.sql.functions import udf
    from pyspark.sql.types import ArrayType, FloatType

    embed_udf = udf(embed_text, ArrayType(FloatType()))
    embedded_df = clean_df.withColumn("embeddings", embed_udf(col("text")))
    embedded_df.show(5)
    ```

## Paso 5: Almacenamiento de incrustaciones
- Almacenamiento de incrustaciones en tablas Delta
    1. Guarda los datos incrustados en una tabla Delta para una recuperación eficaz.

    ```python
    embedded_df.write.format("delta").mode("overwrite").save("/mnt/delta/wiki_embeddings")
    ```

    2. Creación de una tabla Delta

    ```python
    CREATE TABLE IF NOT EXISTS wiki_embeddings
     LOCATION '/mnt/delta/wiki_embeddings'
    ```
## Paso 6: Implementación del vector de búsqueda
- Configuración de la búsqueda de vectores
    1. Usa las funcionalidades de vector de búsqueda de Databricks o integra con una base de datos vectorial como Milvus o Pinecone.

    ```python
    from databricks.feature_store import FeatureStoreClient

    fs = FeatureStoreClient()

    fs.create_table(
        name="wiki_embeddings_vector_store",
        primary_keys=["title"],
        df=embedded_df,
        description="Vector embeddings for Wikipedia articles."
    )
    ```
- Ejecución del vector de búsqueda
    1. Implementa una función para buscar documentos relevantes basados en un vector de consulta.

    ```python
    def search_vectors(query_text, top_k=5):
        query_embedding = model.encode([query_text]).tolist()
        query_df = spark.createDataFrame([(query_text, query_embedding)], ["query_text", "query_embedding"])
        
        results = fs.search_table(
            name="wiki_embeddings_vector_store",
            vector_column="embeddings",
            query=query_df,
            top_k=top_k
        )
        return results

    query = "Machine learning applications"
    search_results = search_vectors(query)
    search_results.show()
    ```

## Paso 7: Aumento generativo
- Aumento de solicitudes con datos recuperados:
    1. Combina los datos recuperados con la consulta del usuario para crear una solicitud enriquecida para el LLM.

    ```python
    def augment_prompt(query_text):
        search_results = search_vectors(query_text)
        context = " ".join(search_results.select("text").rdd.flatMap(lambda x: x).collect())
        return f"Query: {query_text}\nContext: {context}"

    prompt = augment_prompt("Explain the significance of the Turing test")
    print(prompt)
    ```

- Generación de respuestas con LLM:
    2. Usa un LLM como GPT-3 o modelos similares de Hugging Face para generar respuestas.

    ```python
    from transformers import GPT2LMHeadModel, GPT2Tokenizer

    tokenizer = GPT2Tokenizer.from_pretrained("gpt2")
    model = GPT2LMHeadModel.from_pretrained("gpt2")

    inputs = tokenizer(prompt, return_tensors="pt")
    outputs = model.generate(inputs["input_ids"], max_length=500, num_return_sequences=1)
    response = tokenizer.decode(outputs[0], skip_special_tokens=True)

    print(response)
    ```

## Paso 8: Evaluación y optimización
- Evaluación de la calidad de las respuestas generadas:
    1. Evalúa la relevancia, la coherencia y la precisión de las respuestas generadas.
    2. Recopila los comentarios de los usuarios e itera el proceso de aumento de solicitudes.

- Optimización del flujo de trabajo RAG:
    1. Experimenta con diferentes modelos de incrustación, tamaños de fragmentos y parámetros de recuperación para optimizar el rendimiento.
    2. Supervisa el rendimiento del sistema y realiza ajustes para mejorar la precisión y la eficacia.

## Paso 9: Limpieza de recursos
- Finalización del clúster:
    1. Vuelve a la página "Proceso", selecciona el clúster y haz clic en "Finalizar" para detener el clúster.

- Opcional: Eliminación del servicio de Databricks:
    1. Para evitar incurrir en cargos adicionales, considera la posibilidad de eliminar el área de trabajo de Databricks si este laboratorio no forma parte de un proyecto o una ruta de aprendizaje más grande.

Siguiendo estos pasos, habrás implementado un sistema de generación aumentada de recuperación (RAG) con Azure Databricks. En este laboratorio se muestra cómo preprocesar datos, generar incrustaciones, almacenarlos de forma eficaz, realizar búsquedas vectoriales y usar modelos generativos para crear respuestas enriquecidas. El enfoque se puede adaptar a varios dominios y conjuntos de datos para mejorar las funcionalidades de las aplicaciones controladas por IA.