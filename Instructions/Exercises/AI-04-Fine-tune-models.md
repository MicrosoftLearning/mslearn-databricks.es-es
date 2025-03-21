---
lab:
  title: Ajuste de modelos de lenguaje grande mediante Azure Databricks y Azure OpenAI
---

# Ajuste de modelos de lenguaje grande mediante Azure Databricks y Azure OpenAI

Con Azure Databricks, los usuarios ahora pueden sacar provecho de la eficacia de los modelos de lenguaje grande para tareas especializadas mediante la optimización de ellos con sus propios datos, lo que mejora el rendimiento específico del dominio. Para optimizar un modelo de lenguaje mediante Azure Databricks, puedes usar la interfaz de entrenamiento de modelos de IA de Mosaic, que simplifica el proceso de optimización completa del modelo. Esta característica permite optimizar un modelo con datos personalizados, con puntos de control guardados en MLflow, lo que garantiza que conserves el control total sobre el modelo optimizado.

Este laboratorio se tarda en completar **60** minutos aproximadamente.

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
        - Este de EE. UU. 2
        - Centro-Norte de EE. UU
        - Centro de Suecia
        - Oeste de Suiza
    - **Nombre**: *nombre único que prefiera*
    - **Plan de tarifa**: estándar S0

> \* Los recursos de Azure OpenAI están restringidos por cuotas regionales. Las regiones enumeradas incluyen la cuota predeterminada para los tipos de modelo usados en este ejercicio. Elegir aleatoriamente una región reduce el riesgo de que una sola región alcance su límite de cuota en escenarios en los que se comparte una suscripción con otros usuarios. En caso de que se alcance un límite de cuota más adelante en el ejercicio, es posible que tenga que crear otro recurso en otra región.

3. Espere a que la implementación finalice. A continuación, vaya al recurso de Azure OpenAI implementado en Azure Portal.

4. En el panel de la izquierda, en **Administración de recursos**, selecciona **Claves y puntos de conexión**.

5. Copia el punto de conexión y una de las claves disponibles, ya que los usarás más adelante en este ejercicio.

6. Inicia Cloud Shell y ejecuta `az account get-access-token` para obtener un token de autorización temporal para las pruebas de API. Mantenlo con el punto de conexión y la clave copiados anteriormente.

## Implementación del modelo necesario

Azure proporciona un portal basado en web denominado **Azure AI Studio** que puedes usar para implementar, administrar y explorar modelos. Para iniciar la exploración de Azure OpenAI, usa Azure AI Studio para implementar un modelo.

> **Nota**: a medida que usas Azure AI Studio, es posible que se muestren cuadros de mensaje que sugieren tareas que se van a realizar. Puede cerrarlos y seguir los pasos descritos en este ejercicio.

1. En Azure Portal, en la página **Información general** del recurso de Azure OpenAI, desplázate hacia abajo hasta la sección **Comenzar** y selecciona el botón para ir a **Inteligencia artificial de Azure Studio**.
   
1. En Azure AI Studio, en el panel de la izquierda, selecciona la página **Implementaciones** y consulta las implementaciones de modelos existentes. Si aún no tienes una, crea una nueva implementación del modelo **gpt-35-turbo** con la siguiente configuración:
    - **Nombre de implementación**: *gpt-35-turbo-0613*
    - **Modelo**: gpt-35-turbo
    - **Versión del modelo**: 0613
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

1. En la página del clúster, selecciona la pestaña **Bibliotecas**.

2. Selecciona **Instalar nueva**.

3. Selecciona **PyPI** como origen de la biblioteca e instala los siguientes paquetes de Python:
   - `numpy==2.1.0`
   - `requests==2.32.3`
   - `openai==1.42.0`
   - `tiktoken==0.7.0`

## Creación de un cuaderno nuevo e ingesta de datos

1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**.
   
1. Asigna un nombre a tu cuaderno y en la lista desplegable **Conectar** selecciona el clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

2. En la primera celda del cuaderno, escribe el siguiente código, que utiliza comandos del *shell* para descargar los archivos de datos de GitHub en el sistema de archivos utilizado por el clúster.

     ```python
    %sh
    rm -r /dbfs/fine_tuning
    mkdir /dbfs/fine_tuning
    wget -O /dbfs/fine_tuning/training_set.jsonl https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/training_set.jsonl
    wget -O /dbfs/fine_tuning/validation_set.jsonl https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/validation_set.jsonl
     ```

3. En una celda nueva, ejecuta el código siguiente con la información de acceso que copiaste al principio de este ejercicio para asignar variables de entorno persistentes para la autenticación al usar recursos de Azure OpenAI:

     ```python
    import os

    os.environ["AZURE_OPENAI_API_KEY"] = "your_openai_api_key"
    os.environ["AZURE_OPENAI_ENDPOINT"] = "your_openai_endpoint"
    os.environ["TEMP_AUTH_TOKEN"] = "your_access_token"
     ```
     
## Valida los recuentos de tokens

Tanto `training_set.jsonl` como `validation_set.jsonl` se componen de diferentes ejemplos de conversación entre `user` y `assistant` que servirán como puntos de datos para el entrenamiento y la validación del modelo optimizado. Los ejemplos individuales deben permanecer bajo el límite de tokens de entrada del modelo `gpt-35-turbo`, que es de 4096 tokens.

1. En una nueva celda, ejecuta el código siguiente para validar los recuentos de tokens para cada fila:

   ```python
    import json
    import tiktoken
    import numpy as np
    from collections import defaultdict

    encoding = tiktoken.get_encoding("cl100k_base")

    def num_tokens_from_messages(messages, tokens_per_message=3, tokens_per_name=1):
        num_tokens = 0
        for message in messages:
            num_tokens += tokens_per_message
            for key, value in message.items():
                num_tokens += len(encoding.encode(value))
                if key == "name":
                    num_tokens += tokens_per_name
        num_tokens += 3
        return num_tokens

    def num_assistant_tokens_from_messages(messages):
        num_tokens = 0
        for message in messages:
            if message["role"] == "assistant":
                num_tokens += len(encoding.encode(message["content"]))
        return num_tokens

    def print_distribution(values, name):
        print(f"\n##### Distribution of {name}:")
        print(f"min / max: {min(values)}, {max(values)}")
        print(f"mean / median: {np.mean(values)}, {np.median(values)}")

    files = ['/dbfs/fine_tuning/training_set.jsonl', '/dbfs/fine_tuning/validation_set.jsonl']

    for file in files:
        print(f"File: {file}")
        with open(file, 'r', encoding='utf-8') as f:
            dataset = [json.loads(line) for line in f]

        total_tokens = []
        assistant_tokens = []

        for ex in dataset:
            messages = ex.get("messages", {})
            total_tokens.append(num_tokens_from_messages(messages))
            assistant_tokens.append(num_assistant_tokens_from_messages(messages))

        print_distribution(total_tokens, "total tokens")
        print_distribution(assistant_tokens, "assistant tokens")
        print('*' * 75)
   ```

## Carga de archivos de optimización en Azure OpenAI

Antes de empezar a optimizar el modelo, debes inicializar un cliente de OpenAI y agregar los archivos de optimización a su entorno, generando identificadores de archivo que se usarán para inicializar el trabajo.

1. En una celda nueva, escribe el código siguiente:

     ```python
    import os
    from openai import AzureOpenAI

    client = AzureOpenAI(
      azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
      api_key = os.getenv("AZURE_OPENAI_API_KEY"),
      api_version = "2024-05-01-preview"  # This API version or later is required to access seed/events/checkpoint features
    )

    training_file_name = '/dbfs/fine_tuning/training_set.jsonl'
    validation_file_name = '/dbfs/fine_tuning/validation_set.jsonl'

    training_response = client.files.create(
        file = open(training_file_name, "rb"), purpose="fine-tune"
    )
    training_file_id = training_response.id

    validation_response = client.files.create(
        file = open(validation_file_name, "rb"), purpose="fine-tune"
    )
    validation_file_id = validation_response.id

    print("Training file ID:", training_file_id)
    print("Validation file ID:", validation_file_id)
     ```

## Envío del trabajo de optimización

Ahora que se han cargado correctamente los archivos de optimización, puedes enviar tu trabajo de entrenamiento de optimización. Puede ocurrir que el entrenamiento tarde más de una hora en completarse. Una vez completado el entrenamiento, puedes ver los resultados en Inteligencia artificial de Azure Studio seleccionando la opción **Optimizar** en el panel izquierdo.

1. En una nueva celda, ejecuta el código siguiente para inicializar el trabajo de entrenamiento de optimización:

     ```python
    response = client.fine_tuning.jobs.create(
        training_file = training_file_id,
        validation_file = validation_file_id,
        model = "gpt-35-turbo-0613",
        seed = 105 # seed parameter controls reproducibility of the fine-tuning job. If no seed is specified one will be generated automatically.
    )

    job_id = response.id
     ```

El parámetro `seed` controla la reproducibilidad del trabajo de optimización. Pasar los mismos parámetros de inicialización y trabajo debe generar los mismos resultados, pero puede diferir en raras ocasiones. Si no se especifica ningún valor de inicialización, se generará uno automáticamente.

2. En una nueva celda, puedes ejecutar el código siguiente para supervisar el estado del trabajo de optimización:

     ```python
    print("Job ID:", response.id)
    print("Status:", response.status)
     ```

3. Una vez que el estado del trabajo cambie a `succeeded`, ejecuta el código siguiente para obtener los resultados finales:

     ```python
    response = client.fine_tuning.jobs.retrieve(job_id)

    print(response.model_dump_json(indent=2))
    fine_tuned_model = response.fine_tuned_model
     ```
   
## Implementación de un modelo con ajuste preciso

Ahora que tienes un modelo optimizado, puedes implementarlo como modelo personalizado y usarlo como cualquier otro modelo implementado en el área de juegos de **chat** de Inteligencia artificial de Azure Studio o a través de la API de finalización de chat.

1. En una nueva celda, ejecuta el código siguiente para implementar el modelo optimizado:
   
     ```python
    import json
    import requests

    token = os.getenv("TEMP_AUTH_TOKEN")
    subscription = "<YOUR_SUBSCRIPTION_ID>"
    resource_group = "<YOUR_RESOURCE_GROUP_NAME>"
    resource_name = "<YOUR_AZURE_OPENAI_RESOURCE_NAME>"
    model_deployment_name = "gpt-35-turbo-ft"

    deploy_params = {'api-version': "2023-05-01"}
    deploy_headers = {'Authorization': 'Bearer {}'.format(token), 'Content-Type': 'application/json'}

    deploy_data = {
        "sku": {"name": "standard", "capacity": 1},
        "properties": {
            "model": {
                "format": "OpenAI",
                "name": "<YOUR_FINE_TUNED_MODEL>",
                "version": "1"
            }
        }
    }
    deploy_data = json.dumps(deploy_data)

    request_url = f'https://management.azure.com/subscriptions/{subscription}/resourceGroups/{resource_group}/providers/Microsoft.CognitiveServices/accounts/{resource_name}/deployments/{model_deployment_name}'

    print('Creating a new deployment...')

    r = requests.put(request_url, params=deploy_params, headers=deploy_headers, data=deploy_data)

    print(r)
    print(r.reason)
    print(r.json())
     ```

2. En una nueva celda, ejecuta el código siguiente para usar el modelo personalizado en una llamada de finalización de chat:
   
     ```python
    import os
    from openai import AzureOpenAI

    client = AzureOpenAI(
      azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
      api_key = os.getenv("AZURE_OPENAI_API_KEY"),
      api_version = "2024-02-01"
    )

    response = client.chat.completions.create(
        model = "gpt-35-turbo-ft", # model = "Custom deployment name you chose for your fine-tuning model"
        messages = [
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Does Azure OpenAI support customer managed keys?"},
            {"role": "assistant", "content": "Yes, customer managed keys are supported by Azure OpenAI."},
            {"role": "user", "content": "Do other Azure AI services support this too?"}
        ]
    )

    print(response.choices[0].message.content)
     ```
 
## Limpieza

Cuando haya terminado de usar el recurso de Azure OpenAI, recuerde eliminar la implementación o todo el recurso en **Azure Portal**, en `https://portal.azure.com`.

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si has terminado de explorar Azure Databricks, puedes eliminar los recursos que has creado para evitar costes innecesarios de Azure y liberar capacidad en tu suscripción.
