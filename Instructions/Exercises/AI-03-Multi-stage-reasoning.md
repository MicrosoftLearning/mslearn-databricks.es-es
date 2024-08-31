# Ejercicio 03: Razonamiento en varias fases con LangChain mediante Azure Databricks y GPT-4

## Objetivo
Este ejercicio tiene como objetivo guiarte a través de la compilación de un sistema de razonamiento de varias fases mediante LangChain en Azure Databricks. Aprenderás a crear un índice vectorial, almacenar incrustaciones, compilar una cadena basada en un recuperador, construir una cadena de generación de imágenes y, por último, combinarlas en un sistema de varias cadenas mediante el modelo GPT-4 de OpenAI.

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

- Abre un nuevo cuaderno en el área de trabajo.
- Instala las bibliotecas necesarias mediante los siguientes comandos:

```python
%pip install langchain openai faiss-cpu
```

- Configuración de Azure OpenAI

```python
import os
os.environ["OPENAI_API_KEY"] = "your-openai-api-key"
```

## Paso 4: crear un índice vectorial y almacenar incrustaciones

- Carga del conjunto de datos
    1. Carga un conjunto de datos de muestra para el que quieras generar incrustaciones. Para este laboratorio, utilizaremos un conjunto de datos de texto pequeño.

    ```python
    sample_texts = [
        "Azure Databricks is a fast, easy, and collaborative Apache Spark-based analytics platform.",
        "LangChain is a framework designed to simplify the creation of applications using large language models.",
        "GPT-4 is a powerful language model developed by OpenAI."
    ]
    ```
- Generación de incrustaciones
    1. Usa el modelo GPT-4 de OpenAI para generar incrustaciones para estos textos.

    ```python
    from langchain.embeddings.openai import OpenAIEmbeddings

    embeddings_model = OpenAIEmbeddings()
    embeddings = embeddings_model.embed_documents(sample_texts)
    ``` 

- Almacenamiento de las incrustaciones mediante FAISS
    1. Usa FAISS para crear un índice vectorial para una recuperación eficaz.

    ```python
    import faiss
    import numpy as np

    dimension = len(embeddings[0])
    index = faiss.IndexFlatL2(dimension)
    index.add(np.array(embeddings))
    ```

## Paso 5: Creación de una cadena basada en un recuperador
- Definición del recuperador
    1. Crea un recuperador que pueda buscar en el índice vectorial los textos más similares.

    ```python
    from langchain.chains import RetrievalQA
    from langchain.vectorstores.faiss import FAISS

    vector_store = FAISS(index, embeddings_model)
    retriever = vector_store.as_retriever()  
    ```

- Compilación de la cadena de RetrievalQA
    1. Crea un sistema de QA mediante el recuperador y el modelo GPT-4.
    
    ```python
    from langchain.llms import OpenAI
    from langchain.chains.question_answering import load_qa_chain

    llm = OpenAI(model_name="gpt-4")
    qa_chain = load_qa_chain(llm, retriever)
    ```

- Prueba del sistema de QA
    1. Haz una pregunta relacionada con los textos que has incrustado

    ```python
    result = qa_chain.run("What is Azure Databricks?")
    print(result)
    ```

## Paso 6: Compilación de una cadena de generación de imágenes

- Configuración del modelo de generación de imágenes
    1. Configura las funcionalidades de generación de imágenes mediante GPT-4.

    ```python
    from langchain.chains import SimpleChain

    def generate_image(prompt):
        # Assuming you have an endpoint or a tool to generate images from text.
        return f"Generated image for prompt: {prompt}"

    image_generation_chain = SimpleChain(input_variables=["prompt"], output_variables=["image"], transform=generate_image)
    ```

- Prueba de la cadena de generación de imágenes
    1. Genera una imagen basada en una solicitud de texto.

    ```python
    prompt = "A futuristic city with flying cars"
    image_result = image_generation_chain.run(prompt=prompt)
    print(image_result)
    ```

## Paso 7: Combinación de cadenas en un sistema de varias cadenas
- Combinación de cadenas
    1. Integre la cadena de QA basada en un recuperador y la cadena de generación de imágenes en un sistema de varias cadenas.

    ```python
    from langchain.chains import MultiChain

    multi_chain = MultiChain(
        chains=[
            {"name": "qa", "chain": qa_chain},
            {"name": "image_generation", "chain": image_generation_chain}
        ]
    )
    ```

- Ejecución del sistema de varias cadenas
    1. Pasa una tarea que implique la recuperación de texto y la generación de imágenes.

    ```python
    multi_task_input = {
        "qa": {"question": "Tell me about LangChain."},
        "image_generation": {"prompt": "A conceptual diagram of LangChain in use"}
    }

    multi_task_output = multi_chain.run(multi_task_input)
    print(multi_task_output)
    ```

## Paso 8: Limpieza de recursos
- Finalización del clúster:
    1. Vuelve a la página "Proceso", selecciona el clúster y haz clic en "Finalizar" para detener el clúster.

- Opcional: Eliminación del servicio de Databricks:
    1. Para evitar incurrir en cargos adicionales, considera la posibilidad de eliminar el área de trabajo de Databricks si este laboratorio no forma parte de un proyecto o una ruta de aprendizaje más grande.

Esto concluye el ejercicio sobre el razonamiento en varias fases con LangChain mediante Azure Databricks.