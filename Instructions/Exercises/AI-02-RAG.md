---
lab:
  title: Generación aumentada de recuperación con Azure Databricks
---

# Generación aumentada de recuperación con Azure Databricks

La generación aumentada de recuperación (RAG) es un enfoque de vanguardia en la IA que mejora los modelos de lenguaje grande mediante la integración de orígenes de conocimiento externos. Azure Databricks ofrece una plataforma sólida para desarrollar aplicaciones RAG, lo que permite la transformación de datos no estructurados en un formato adecuado para la recuperación y la generación de respuestas. Este proceso implica una serie de pasos, incluida la comprensión de la consulta del usuario, la recuperación de datos pertinentes y la generación de una respuesta mediante un modelo de lenguaje. El marco proporcionado por Azure Databricks admite la iteración e implementación rápida de aplicaciones RAG, lo que garantiza respuestas específicas de dominio de alta calidad que pueden incluir información actualizada y conocimientos de propiedad exclusiva.

Se tardan aproximadamente **40** minutos en completar este laboratorio.

> **Nota**: la interfaz de usuario de Azure Databricks está sujeta a una mejora continua. Es posible que la interfaz de usuario haya cambiado desde que se escribieron las instrucciones de este ejercicio.

## Antes de empezar

Necesitará una [suscripción de Azure](https://azure.microsoft.com/free) en la que tenga acceso de nivel administrativo.

## Aprovisiona un área de trabajo de Azure Databricks.

> **Sugerencia**: si ya tienes un área de trabajo de Azure Databricks, puedes omitir este procedimiento y usar el área de trabajo existente.

En este ejercicio, se incluye un script para aprovisionar una nueva área de trabajo de Azure Databricks. El script intenta crear un recurso de área de trabajo de Azure Databricks de nivel *Premium* en una región en la que la suscripción de Azure tiene cuota suficiente para los núcleos de proceso necesarios en este ejercicio, y da por hecho que la cuenta de usuario tiene permisos suficientes en la suscripción para crear un recurso de área de trabajo de Azure Databricks. Si se produjese un error en el script debido a cuota o permisos insuficientes, intenta [crear un área de trabajo de Azure Databricks de forma interactiva en Azure Portal](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. En un explorador web, inicia sesión en [Azure Portal](https://portal.azure.com) en `https://portal.azure.com`.
2. Usa el botón **[\>_]** situado a la derecha de la barra de búsqueda en la parte superior de la página para crear una nueva instancia de Cloud Shell en Azure Portal, para lo que deberás seleccionar un entorno de ***PowerShell***. Cloud Shell proporciona una interfaz de línea de comandos en un panel situado en la parte inferior de Azure Portal, como se muestra a continuación:

    ![Azure Portal con un panel de Cloud Shell](./images/cloud-shell.png)

    > **Nota**: si has creado anteriormente una instancia de Cloud Shell que usa un entorno de *Bash*, cámbiala a ***PowerShell***.

3. Ten en cuenta que puedes cambiar el tamaño de la instancia de Cloud Shell. Para ello, arrastra la barra de separación de la parte superior del panel o utiliza los iconos **&#8212;**, **&#10530;** y **X** de la parte superior derecha del panel para minimizar, maximizar y cerrar el panel. Para obtener más información sobre el uso de Azure Cloud Shell, consulta la [documentación de Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

4. En el panel de PowerShell, introduce los siguientes comandos para clonar este repositorio:

    ```powershell
   rm -r mslearn-databricks -f
   git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

5. Una vez clonado el repositorio, escribe el siguiente comando para ejecutar el script **setup.ps1**, que aprovisiona un área de trabajo de Azure Databricks en una región disponible:

    ```powershell
   ./mslearn-databricks/setup.ps1
    ```

6. Si se solicita, elige la suscripción que quieres usar (esto solo ocurrirá si tienes acceso a varias suscripciones de Azure).

7. Espera a que se complete el script: normalmente tarda unos 5 minutos, pero en algunos casos puede tardar más.

## Crear un clúster

Azure Databricks es una plataforma de procesamiento distribuido que usa clústeres* de Apache Spark *para procesar datos en paralelo en varios nodos. Cada clúster consta de un nodo de controlador para coordinar el trabajo y nodos de trabajo para hacer tareas de procesamiento. En este ejercicio, crearás un clúster de *nodo único* para minimizar los recursos de proceso usados en el entorno de laboratorio (en los que se pueden restringir los recursos). En un entorno de producción, normalmente crearías un clúster con varios nodos de trabajo.

> **Sugerencia**: Si ya dispones de un clúster con una versión de runtime 15.4 LTS **<u>ML</u>** o superior en su área de trabajo de Azure Databricks, puedes utilizarlo para completar este ejercicio y omitir este procedimiento.

1. En Azure Portal, ve al grupo de recursos **msl-*xxxxxxx*** que se creó con el script (o al grupo de recursos que contiene el área de trabajo de Azure Databricks existente)
1. Selecciona el recurso Azure Databricks Service (llamado **databricks-*xxxxxxx*** si usaste el script de instalación para crearlo).
1. En la página **Información general** del área de trabajo, usa el botón **Inicio del área de trabajo** para abrir el área de trabajo de Azure Databricks en una nueva pestaña del explorador; inicia sesión si se solicita.

    > **Sugerencia**: al usar el portal del área de trabajo de Databricks, se pueden mostrar varias sugerencias y notificaciones. Descártalas y sigue las instrucciones proporcionadas para completar las tareas de este ejercicio.

1. En la barra lateral de la izquierda, selecciona la tarea **(+) Nuevo** y luego selecciona **Clúster**.
1. En la página **Nuevo clúster**, crea un clúster con la siguiente configuración:
    - **Nombre del clúster**: clúster del *Nombre de usuario*  (el nombre del clúster predeterminado)
    - **Directiva**: Unrestricted (Sin restricciones)
    - **Aprendizaje automático**: Habilitado
    - **Databricks Runtime**: 15.4 LTS
    - **Utilizar la Aceleración de fotones**: <u>No</u> seleccionada
    - **Tipo de trabajo**: Standard_D4ds_v5
    - **Nodo único**: Activado

1. Espera a que se cree el clúster. Esto puede tardar un par de minutos.

> **Nota**: si el clúster no se inicia, es posible que la suscripción no tenga cuota suficiente en la región donde se aprovisiona el área de trabajo de Azure Databricks. Para obtener más información, consulta [El límite de núcleos de la CPU impide la creación de clústeres](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si esto sucede, puedes intentar eliminar el área de trabajo y crear una nueva en otra región. Puedes especificar una región como parámetro para el script de configuración de la siguiente manera: `./mslearn-databricks/setup.ps1 eastus`

## Instalación de bibliotecas necesarias

1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**. En la lista desplegable **Conectar**, selecciona el clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.
1. En la primera celda de código, escribe y ejecuta el código siguiente para instalar las bibliotecas necesarias:
   
    ```python
   %pip install faiss-cpu
   dbutils.library.restartPython()
    ```
   
## Ingerir datos

1. En una nueva pestaña del explorador, descargue el [archivo de ejemplo](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/enwiki-latest-pages-articles.xml) que se usará como datos en este ejercicio: `https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/enwiki-latest-pages-articles.xml`
1. De nuevo en la pestaña Área de trabajo de Databricks, con el cuaderno abierto, seleccione el explorador **Catálogo (CTRL + Alt + C)** y el icono ➕ para **Agregar datos**.
1. En la página **Agregar datos**, seleccione **Cargar archivos en DBFS**.
1. En la página **DBFS**, asigne el nombre `RAG_lab` al directorio de destino y cargue el archivo .xml que ha guardado antes.
1. En la barra lateral, seleccione **Área de trabajo** y vuelva a abrir el cuaderno.
1. En una nueva celda de código, ejecuta el siguiente código para crear un marco de datos a partir de los datos sin procesar:

    ```python
   from pyspark.sql import SparkSession

   # Create a Spark session
   spark = SparkSession.builder \
       .appName("RAG-DataPrep") \
       .getOrCreate()

   # Read the XML file
   raw_df = spark.read.format("xml") \
       .option("rowTag", "page") \
       .load("/FileStore/tables/RAG_lab/enwiki_latest_pages_articles.xml")

   # Show the DataFrame
   raw_df.show(5)

   # Print the schema of the DataFrame
   raw_df.printSchema()
    ```

1. Usa la opción del menú **&#9656; Ejecutar celda** situado a la izquierda de la celda para ejecutarla. A continuación, espera a que se complete el trabajo de Spark ejecutado por el código.
1. En una nueva celda, ejecute el código siguiente para limpiar y preprocesar los datos a fin de extraer los campos de texto pertinentes:

    ```python
   from pyspark.sql.functions import col

   clean_df = raw_df.select(col("title"), col("revision.text._VALUE").alias("text"))
   clean_df = clean_df.na.drop()
   clean_df.show(5)
    ```

## Generación de incrustaciones e implementación del vector de búsqueda

FAISS (Facebook AI Similarity Search) es una biblioteca de bases de datos vectoriales de código abierto desarrollada por Meta AI, diseñada para una búsqueda eficaz de similitud y agrupación en clústeres de vectores densos. FAISS permite búsquedas de vecino más próximo rápidas y escalables, y se puede integrar con sistemas de búsqueda híbridos para combinar la similitud basada en vectores con técnicas tradicionales basadas en palabras clave, lo que mejora la relevancia de los resultados de la búsqueda.

1. En una nueva celda, ejecute el código siguiente para cargar el modelo `all-MiniLM-L6-v2` entrenado previamente y convertir texto en inserciones:

    ```python
   from sentence_transformers import SentenceTransformer
   import numpy as np
    
   # Load pre-trained model
   model = SentenceTransformer('all-MiniLM-L6-v2')
    
   # Function to convert text to embeddings
   def text_to_embedding(text):
       embeddings = model.encode([text])
       return embeddings[0]
    
   # Convert the DataFrame to a Pandas DataFrame
   pandas_df = clean_df.toPandas()
    
   # Apply the function to get embeddings
   pandas_df['embedding'] = pandas_df['text'].apply(text_to_embedding)
   embeddings = np.vstack(pandas_df['embedding'].values)
    ```

1. En una nueva celda, ejecute el código siguiente para crear y consultar el índice de FAISS:

    ```python
   import faiss
    
   # Create a FAISS index
   d = embeddings.shape[1]  # dimension
   index = faiss.IndexFlatL2(d)  # L2 distance
   index.add(embeddings)  # add vectors to the index
    
   # Perform a search
   query_embedding = text_to_embedding("Anthropology fields")
   k = 1  # number of nearest neighbors
   distances, indices = index.search(np.array([query_embedding]), k)
    
   # Get the results
   results = pandas_df.iloc[indices[0]]
   display(results)
    ```

Comprueba que la salida encuentra la página Wiki correspondiente relacionada con la solicitud de consulta.

## Aumento de avisos con datos recuperados

Ahora podemos mejorar las capacidades de los modelos de lenguaje de grande proporcionándoles un contexto adicional a partir de orígenes de datos externos. De este modo, los modelos pueden generar respuestas más precisas y adaptadas al contexto.

1. En una nueva celda, ejecuta el siguiente código para combinar los datos recuperados con la consulta del usuario y así crear una solicitud enriquecida para el LLM.

    ```python
   from transformers import pipeline
    
   # Load the summarization model
   summarizer = pipeline("summarization", model="facebook/bart-large-cnn", framework="pt")
    
   # Extract the string values from the DataFrame column
   text_data = results["text"].tolist()
    
   # Pass the extracted text data to the summarizer function
   summary = summarizer(text_data, max_length=512, min_length=100, do_sample=True)
    
   def augment_prompt(query_text):
       context = " ".join([item['summary_text'] for item in summary])
       return f"{context}\n\nQuestion: {query_text}\nAnswer:"
    
   prompt = augment_prompt("Explain the significance of Anthropology")
   print(prompt)
    ```

1. En una nueva celda, ejecuta el código siguiente para usar un LLM para generar respuestas.

    ```python
   from transformers import GPT2LMHeadModel, GPT2Tokenizer

   tokenizer = GPT2Tokenizer.from_pretrained("gpt2")
   model = GPT2LMHeadModel.from_pretrained("gpt2")

   inputs = tokenizer(prompt, return_tensors="pt")
   outputs = model.generate(
       inputs["input_ids"], 
       max_length=300, 
       num_return_sequences=1, 
       repetition_penalty=2.0, 
       top_k=50, 
       top_p=0.95, 
       temperature=0.7,
       do_sample=True
   )
   response = tokenizer.decode(outputs[0], skip_special_tokens=True)

   print(response)
    ```

## Limpieza

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si has terminado de explorar Azure Databricks, puedes eliminar los recursos que has creado para evitar costes innecesarios de Azure y liberar capacidad en tu suscripción.
