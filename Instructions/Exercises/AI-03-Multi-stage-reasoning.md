---
lab:
  title: Razonamiento en varias fases con LangChain mediante Azure Databricks y Azure OpenAI
---

# Razonamiento en varias fases con LangChain mediante Azure Databricks y Azure OpenAI

El razonamiento en varias fases es un enfoque de vanguardia en la IA que implica dividir problemas complejos en fases más pequeñas y manejables. LangChain, un marco de software, facilita la creación de aplicaciones que aprovechan modelos de lenguaje grandes (LLM). Cuando se integra con Azure Databricks, LangChain permite la carga de datos sin problemas, el ajuste del modelo y el desarrollo de agentes sofisticados de IA. Esta combinación es especialmente eficaz para controlar tareas complejas que requieren un profundo conocimiento del contexto y la capacidad de razonar en varios pasos.

Este laboratorio se tarda aproximadamente **30** minutos en completarse.

> **Nota**: la interfaz de usuario de Azure Databricks está sujeta a una mejora continua. Es posible que la interfaz de usuario haya cambiado desde que se escribieron las instrucciones de este ejercicio.

## Antes de empezar

Necesitará una [suscripción de Azure](https://azure.microsoft.com/free) en la que tenga acceso de nivel administrativo.

## Aprovisionamiento de un recurso de Azure OpenAI

Si aún no tiene uno, aprovisione un recurso de Azure OpenAI en la suscripción de Azure.

1. Inicie sesión en **Azure Portal** en `https://portal.azure.com`.
2. Cree un recurso de **Azure OpenAI** con la siguiente configuración:
    - **Suscripción**: *Selección de una suscripción de Azure aprobada para acceder al servicio Azure OpenAI*
    - **Grupo de recursos**: *elija o cree un grupo de recursos*
    - **Región**: *Elija de forma **aleatoria** cualquiera de las siguientes regiones*\*
        - Este de Australia
        - Este de Canadá
        - Este de EE. UU.
        - Este de EE. UU. 2
        - Centro de Francia
        - Japón Oriental
        - Centro-Norte de EE. UU
        - Centro de Suecia
        - Norte de Suiza
        - Sur de Reino Unido 2
    - **Nombre**: *nombre único que prefiera*
    - **Plan de tarifa**: estándar S0

> \* Los recursos de Azure OpenAI están restringidos por cuotas regionales. Las regiones enumeradas incluyen la cuota predeterminada para los tipos de modelo usados en este ejercicio. Elegir aleatoriamente una región reduce el riesgo de que una sola región alcance su límite de cuota en escenarios en los que se comparte una suscripción con otros usuarios. En caso de que se alcance un límite de cuota más adelante en el ejercicio, es posible que tenga que crear otro recurso en otra región.

3. Espere a que la implementación finalice. A continuación, vaya al recurso de Azure OpenAI implementado en Azure Portal.

4. En el panel de la izquierda, en **Administración de recursos**, selecciona **Claves y puntos de conexión**.

5. Copia el punto de conexión y una de las claves disponibles, ya que los usarás más adelante en este ejercicio.

## Implementación de los modelos necesarios

Azure proporciona un portal basado en web denominado **Azure AI Studio** que puedes usar para implementar, administrar y explorar modelos. Para iniciar la exploración de Azure OpenAI, usa Azure AI Studio para implementar un modelo.

> **Nota**: a medida que usas Azure AI Studio, es posible que se muestren cuadros de mensaje que sugieren tareas que se van a realizar. Puede cerrarlos y seguir los pasos descritos en este ejercicio.

1. En Azure Portal, en la página **Información general** del recurso de Azure OpenAI, desplázate hacia abajo hasta la sección **Comenzar** y selecciona el botón para ir a **Inteligencia artificial de Azure Studio**.
   
1. En Azure AI Studio, en el panel de la izquierda, selecciona la página **Implementaciones** y consulta las implementaciones de modelos existentes. Si aún no tienes una, crea una nueva implementación del modelo **gpt-35-turbo-16k** con la siguiente configuración:
    - **Nombre de implementación**: *gpt-35-turbo-16k*
    - **Modelo**: gpt-35-turbo-16k *(si el modelo 16k no estuviera disponible, elige gpt-35-turbo y asigna un nombre a tu implementación según corresponda)*
    - **Versión del modelo**: *usar la versión predeterminada*
    - **Tipo de implementación**: Estándar
    - **Límite de velocidad de tokens por minuto**: 5000\*
    - **Filtro de contenido**: valor predeterminado
    - **Habilitación de la cuota dinámica**: deshabilitada
    
1. Vuelve a la página **Implementaciones** y crea una nueva implementación del modelo **text-embedding-ada-002** con la siguiente configuración:
    - **Nombre de implementación**: *text-embedding-ada-002*
    - **Modelo**: text-embedding-ada-002
    - **Versión del modelo**: *usar la versión predeterminada*
    - **Tipo de implementación**: Estándar
    - **Límite de velocidad de tokens por minuto**: 5000\*
    - **Filtro de contenido**: valor predeterminado
    - **Habilitación de la cuota dinámica**: deshabilitada

> \* Un límite de velocidad de 5000 tokens por minuto es más que adecuado para completar este ejercicio, al tiempo que deja capacidad para otras personas que usan la misma suscripción.

## Aprovisiona un área de trabajo de Azure Databricks.

> **Sugerencia**: si ya tienes un área de trabajo de Azure Databricks, puedes omitir este procedimiento y usar el área de trabajo existente.

1. Inicie sesión en **Azure Portal** en `https://portal.azure.com`.
2. Crea un recurso de **Azure Databricks** con la siguiente configuración:
    - **Suscripción**: *selecciona la misma suscripción a Azure que usaste para crear tu recurso de Azure OpenAI*
    - **Grupo de recursos**: *el mismo grupo de recursos donde creaste tu recurso de Azure OpenAI*
    - **Región**: *la misma región donde creaste tu recurso de Azure OpenAI*
    - **Nombre**: *nombre único que prefiera*
    - **Plan de tarifa**: *Premium* o *Prueba*

3. Selecciona **Revisar y crear** y espera a que se complete la implementación. Después, ve al recurso e inicia el espacio de trabajo.

## Crear un clúster

Azure Databricks es una plataforma de procesamiento distribuido que usa clústeres* de Apache Spark *para procesar datos en paralelo en varios nodos. Cada clúster consta de un nodo de controlador para coordinar el trabajo y nodos de trabajo para hacer tareas de procesamiento. En este ejercicio, crearás un clúster de *nodo único* para minimizar los recursos de proceso usados en el entorno de laboratorio (en los que se pueden restringir los recursos). En un entorno de producción, normalmente crearías un clúster con varios nodos de trabajo.

> **Sugerencia**: Si ya dispone de un clúster con una versión de runtime 13.3 LTS **<u>ML</u>** o superior en su área de trabajo de Azure Databricks, puede utilizarlo para completar este ejercicio y omitir este procedimiento.

1. En Azure Portal, ve al grupo de recursos donde se creó el espacio de trabajo de Azure Databricks.
2. Selecciona tu recurso del servicio Azure Databricks.
3. En la página **Información general** del área de trabajo, usa el botón **Inicio del área de trabajo** para abrir el área de trabajo de Azure Databricks en una nueva pestaña del explorador; inicia sesión si se solicita.

> **Sugerencia**: al usar el portal del área de trabajo de Databricks, se pueden mostrar varias sugerencias y notificaciones. Descártalas y sigue las instrucciones proporcionadas para completar las tareas de este ejercicio.

4. En la barra lateral de la izquierda, selecciona la tarea **(+) Nuevo** y luego selecciona **Clúster**.
5. En la página **Nuevo clúster**, crea un clúster con la siguiente configuración:
    - **Nombre del clúster**: clúster del *Nombre de usuario*  (el nombre del clúster predeterminado)
    - **Directiva**: Unrestricted (Sin restricciones)
    - **Modo de clúster** de un solo nodo
    - **Modo de acceso**: usuario único (*con la cuenta de usuario seleccionada*)
    - **Versión de runtime de Databricks**: *Seleccione la edición de **<u>ML</u>** de la última versión no beta más reciente del runtime (**No** una versión de runtime estándar) que:*
        - ***No** usa una GPU*
        - *Incluye Scala > **2.11***
        - *Incluye Spark > **3.4***
    - **Utilizar la Aceleración de fotones**: <u>No</u> seleccionada
    - **Tipo de nodo**: Standard_D4ds_v5
    - **Finaliza después de** *20* **minutos de inactividad**

6. Espera a que se cree el clúster. Esto puede tardar un par de minutos.

> **Nota**: si el clúster no se inicia, es posible que la suscripción no tenga cuota suficiente en la región donde se aprovisiona el área de trabajo de Azure Databricks. Para obtener más información, consulta [El límite de núcleos de la CPU impide la creación de clústeres](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si esto sucede, puedes intentar eliminar el área de trabajo y crear una nueva en otra región.

## Instalación de bibliotecas necesarias

1. En el área de trabajo de Databricks, ve a la sección **Área de trabajo**.

2. Selecciona **Crear** y, después, selecciona **Cuaderno**.

3. Asigna un nombre al cuaderno y selecciona `Python` como lenguaje.

4. En la primera celda de código, escribe y ejecuta el código siguiente para instalar las bibliotecas necesarias:
   
     ```python
    %pip install langchain openai langchain_openai faiss-cpu
     ```

5. Una vez completada la instalación, reinicia el kernel en una nueva celda:

     ```python
    %restart_python
     ```

6. En una nueva celda, define los parámetros de autenticación que se usarán para inicializar los modelos de OpenAI, reemplazando `your_openai_endpoint` y `your_openai_api_key` por el punto de conexión y la clave copiados anteriormente del recurso de OpenAI:

     ```python
    endpoint = "your_openai_endpoint"
    key = "your_openai_api_key"
     ```
     
## Creación de un índice de vectores y almacenamiento de incrustaciones

Un índice de vectores es una estructura de datos especializada que permite un eficaz almacenamiento y recuperación de datos de vectores de alta dimensión, lo que es fundamental para realizar búsquedas de similitud rápidas y consultas de vecino más próximo. Las inserciones, por otro lado, son representaciones numéricas de objetos que capturan su significado en forma de vector, lo que permite que las máquinas procesen y comprendan varios tipos de datos, incluidos texto e imágenes.

1. En una nueva celda, ejecuta el código siguiente para cargar un conjunto de datos de muestra:

     ```python
    from langchain_core.documents import Document

    documents = [
         Document(page_content="Azure Databricks is a fast, easy, and collaborative Apache Spark-based analytics platform.", metadata={"date_created": "2024-08-22"}),
         Document(page_content="LangChain is a framework designed to simplify the creation of applications using large language models.", metadata={"date_created": "2024-08-22"}),
         Document(page_content="GPT-4 is a powerful language model developed by OpenAI.", metadata={"date_created": "2024-08-22"})
    ]
    ids = ["1", "2", "3"]
     ```
     
1. En una nueva celda, ejecuta el código siguiente para generar incrustaciones usando el modelo `text-embedding-ada-002`:

     ```python
    from langchain_openai import AzureOpenAIEmbeddings
     
    embedding_function = AzureOpenAIEmbeddings(
        deployment="text-embedding-ada-002",
        model="text-embedding-ada-002",
        azure_endpoint=endpoint,
        openai_api_key=key,
        chunk_size=1
    )
     ```
     
1. En una nueva celda, ejecuta el código siguiente para crear un índice de vectores con el primer ejemplo de texto como referencia para la dimensión de vector:

     ```python
    import faiss
      
    index = faiss.IndexFlatL2(len(embedding_function.embed_query("Azure Databricks is a fast, easy, and collaborative Apache Spark-based analytics platform.")))
     ```

## Creación de una cadena basada en recuperador

Un componente recuperador captura los documentos o datos relevantes basados en una consulta. Esto es especialmente útil en las aplicaciones que requieren la integración de grandes cantidades de datos para el análisis, como en sistemas de generación aumentada por recuperación.

1. En una nueva celda, ejecuta el código siguiente para crear un recuperador que pueda buscar el índice de vectores para los textos más similares.

     ```python
    from langchain.vectorstores import FAISS
    from langchain_core.vectorstores import VectorStoreRetriever
    from langchain_community.docstore.in_memory import InMemoryDocstore

    vector_store = FAISS(
        embedding_function=embedding_function,
        index=index,
        docstore=InMemoryDocstore(),
        index_to_docstore_id={}
    )
    vector_store.add_documents(documents=documents, ids=ids)
    retriever = VectorStoreRetriever(vectorstore=vector_store)
     ```

1. En una nueva celda, ejecuta el código siguiente para crear un sistema de QA mediante el recuperador y el modelo `gpt-35-turbo-16k`:
    
     ```python
    from langchain_openai import AzureChatOpenAI
    from langchain_core.prompts import ChatPromptTemplate
    from langchain.chains.combine_documents import create_stuff_documents_chain
    from langchain.chains import create_retrieval_chain
     
    llm = AzureChatOpenAI(
        deployment_name="gpt-35-turbo-16k",
        model_name="gpt-35-turbo-16k",
        azure_endpoint=endpoint,
        api_version="2023-03-15-preview",
        openai_api_key=key,
    )

    system_prompt = (
        "Use the given context to answer the question. "
        "If you don't know the answer, say you don't know. "
        "Use three sentences maximum and keep the answer concise. "
        "Context: {context}"
    )

    prompt1 = ChatPromptTemplate.from_messages([
        ("system", system_prompt),
        ("human", "{input}")
    ])

    chain = create_stuff_documents_chain(llm, prompt)

    qa_chain1 = create_retrieval_chain(retriever, chain)
     ```

1. En una nueva celda, ejecuta el código siguiente para probar el sistema de QA:

     ```python
    result = qa_chain1.invoke({"input": "What is Azure Databricks?"})
    print(result)
     ```

La salida del resultado debe mostrar una respuesta basada en el documento relevante presente en el conjunto de datos de ejemplo, además del texto generativo generado por LLM.

## Combinación de cadenas en un sistema de varias cadenas

Langchain es una herramienta versátil que permite la combinación de varias cadenas en un sistema de varias cadenas, lo que mejora las funcionalidades de los modelos de lenguaje. Este proceso implica encadenar varios componentes que pueden procesar entradas en paralelo o en secuencia, sintetizando en última instancia una respuesta final.

1. En una nueva celda, ejecuta el código siguiente para crear una segunda cadena.

     ```python
    from langchain_core.prompts import ChatPromptTemplate
    from langchain_core.output_parsers import StrOutputParser

    prompt2 = ChatPromptTemplate.from_template("Create a social media post based on this summary: {summary}")

    qa_chain2 = ({"summary": qa_chain1} | prompt2 | llm | StrOutputParser())
     ```

1. En una nueva celda, ejecuta el código siguiente para invocar una cadena de varias fases con una entrada determinada:

     ```python
    result = qa_chain2.invoke({"input": "How can we use LangChain?"})
    print(result)
     ```

La primera cadena proporciona una respuesta a la entrada basada en el conjunto de datos de ejemplo proporcionado, mientras que la segunda cadena crea una publicación en redes sociales basada en la salida de la primera cadena. Este enfoque permite controlar tareas de procesamiento de texto más complejas encadenando varios pasos juntos.

## Limpieza

Cuando haya terminado de usar el recurso de Azure OpenAI, recuerde eliminar la implementación o todo el recurso en **Azure Portal**, en `https://portal.azure.com`.

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si has terminado de explorar Azure Databricks, puedes eliminar los recursos que has creado para evitar costes innecesarios de Azure y liberar capacidad en tu suscripción.
