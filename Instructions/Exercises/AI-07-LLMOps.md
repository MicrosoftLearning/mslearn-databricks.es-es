---
lab:
  title: Implementación de LLMOps con Azure Databricks
---

# Implementación de LLMOps con Azure Databricks

Azure Databricks proporciona una plataforma unificada que simplifica el ciclo de vida de la inteligencia artificial, desde la preparación de datos hasta la supervisión y el servicio de modelos, optimizando el rendimiento y la eficacia de los sistemas de aprendizaje automático. Admite el desarrollo de aplicaciones de IA generativa, aprovechando características como Unity Catalog para la gobernanza de datos, MLflow para el seguimiento de modelos y Mosaic AI Model Serving para implementar LLM.

Este laboratorio se tarda aproximadamente **20** minutos en completarse.

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

## Implementación del modelo necesario

Azure proporciona un portal basado en web denominado **Azure AI Studio** que puedes usar para implementar, administrar y explorar modelos. Para iniciar la exploración de Azure OpenAI, usa Azure AI Studio para implementar un modelo.

> **Nota**: a medida que usas Azure AI Studio, es posible que se muestren cuadros de mensaje que sugieren tareas que se van a realizar. Puede cerrarlos y seguir los pasos descritos en este ejercicio.

1. En Azure Portal, en la página **Información general** del recurso de Azure OpenAI, desplázate hacia abajo hasta la sección **Comenzar** y selecciona el botón para ir a **Inteligencia artificial de Azure Studio**.
   
1. En Azure AI Studio, en el panel de la izquierda, selecciona la página **Implementaciones** y consulta las implementaciones de modelos existentes. Si aún no tienes una, crea una nueva implementación del modelo **gpt-35-turbo** con la siguiente configuración:
    - **Nombre de implementación**: *gpt-35-turbo*
    - **Modelo**: gpt-35-turbo
    - **Versión del modelo**: Valor predeterminado
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
    %pip install azure-ai-openai flask
     ```

5. Una vez completada la instalación, reinicia el kernel en una nueva celda:

     ```python
    %restart_python
     ```

## Registra el LLM mediante MLflow

1. En una nueva celda, ejecuta el código siguiente para inicializar el cliente de Azure OpenAI:

     ```python
    from azure.ai.openai import OpenAIClient

    client = OpenAIClient(api_key="<Your_API_Key>")
    model = client.get_model("gpt-3.5-turbo")
     ```

1. En una nueva celda, ejecuta el código siguiente para inicializar el seguimiento de MLflow:     

     ```python
    import mlflow

    mlflow.set_tracking_uri("databricks")
    mlflow.start_run()
     ```

1. En una nueva celda, ejecuta el código siguiente para registrar el modelo:

     ```python
    mlflow.pyfunc.log_model("model", python_model=model)
    mlflow.end_run()
     ```

## Implementación del modelo

1. Crea un nuevo cuaderno y, en su primera celda, ejecuta el código siguiente para crear una API de REST para el modelo:

     ```python
    from flask import Flask, request, jsonify
    import mlflow.pyfunc

    app = Flask(__name__)

    @app.route('/predict', methods=['POST'])
    def predict():
        data = request.json
        model = mlflow.pyfunc.load_model("model")
        prediction = model.predict(data["input"])
        return jsonify(prediction)

    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=5000)
     ```

## Supervisión del modelo

1. En el primer cuaderno, crea una nueva celda y ejecuta el código siguiente para habilitar el registro automático de MLflow:

     ```python
    mlflow.autolog()
     ```

1. En una nueva celda, ejecuta el código siguiente para realizar un seguimiento de las predicciones y los datos de entrada.

     ```python
    mlflow.log_param("input", data["input"])
    mlflow.log_metric("prediction", prediction)
     ```

1. En una nueva celda, ejecuta el código siguiente para supervisar el desfase de datos:

     ```python
    import pandas as pd
    from evidently.dashboard import Dashboard
    from evidently.tabs import DataDriftTab

    report = Dashboard(tabs=[DataDriftTab()])
    report.calculate(reference_data=historical_data, current_data=current_data)
    report.show()
     ```

Una vez que empieces a supervisar el modelo, puedes configurar canalizaciones de reentrenamiento automatizadas basadas en la detección de desfase de datos.

## Limpiar

Cuando haya terminado de usar el recurso de Azure OpenAI, recuerde eliminar la implementación o todo el recurso en **Azure Portal**, en `https://portal.azure.com`.

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si has terminado de explorar Azure Databricks, puedes eliminar los recursos que has creado para evitar costes innecesarios de Azure y liberar capacidad en tu suscripción.
