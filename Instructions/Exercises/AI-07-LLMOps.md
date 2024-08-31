# Ejercicio 07: Implementación de LLMOps con Azure Databricks

## Objetivo
Este ejercicio te guiará por el proceso de implementación de operaciones de modelos de lenguaje grande (LLMOps) mediante Azure Databricks. Al final de este laboratorio, comprenderás cómo administrar, implementar y supervisar modelos de lenguaje grandes (LLM) en un entorno de producción mediante procedimientos recomendados.

## Requisitos
Una suscripción a Azure activa. Si no tiene una, puede registrarse para obtener una versión de [evaluación gratuita](https://azure.microsoft.com/en-us/free/).

## Paso 1: Aprovisionar Azure Databricks
- Inicie sesión en Azure Portal.
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

- Instalación de las bibliotecas necesarias
    1. Una vez que el clúster se esté ejecutando, navega a la pestaña "Bibliotecas".
    2. Instale las siguientes bibliotecas:
        - azure-ai-openai (para conectarse a Azure OpenAI)
        - mlflow (para la administración de modelos)
        - scikit-learn (para obtener una evaluación del modelo adicional, si es necesario)

## Paso 3: Administración de modelos
- Carga o acceso al LLM
    1. Si tienes un modelo entrenado, cárgalo en el sistema de archivos de Databricks (DBFS) o usa Azure OpenAI para acceder a un modelo entrenado previamente.
    2. Si se utiliza Azure OpenAI

    ```python
    from azure.ai.openai import OpenAIClient

    client = OpenAIClient(api_key="<Your_API_Key>")
    model = client.get_model("gpt-3.5-turbo")

    ```
- Control de versiones del modelo mediante MLflow
    1. Inicialización del seguimiento de MLflow

    ```python
    import mlflow

    mlflow.set_tracking_uri("databricks")
    mlflow.start_run()
    ```

- Registro del modelo

```python
mlflow.pyfunc.log_model("model", python_model=model)
mlflow.end_run()

```

## Paso 4: Implementación del modelo
- Creación de una API de REST para el modelo
    1. Crea un cuaderno de Databricks para la API.
    2. Definición de los puntos de conexión de la API mediante Flask o FastAPI

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
- Guarda y ejecuta este cuaderno para iniciar la API.

## Paso 5: Supervisión de modelos
- Configuración del registro y la supervisión mediante MLflow
    1. Habilitación del registro automático de MLflow en el cuaderno

    ```python
    mlflow.autolog()
    ```

    2. Realiza un seguimiento de las predicciones y los datos de entrada.

    ```python
    mlflow.log_param("input", data["input"])
    mlflow.log_metric("prediction", prediction)
    ```

- Implementación de alertas de problemas de desfase o rendimiento del modelo
    1. Usa Azure Databricks o Azure Monitor para configurar alertas de cambios significativos en el rendimiento del modelo.

## Paso 6: Reentrenamiento y automatización de modelos
- Configuración de canalizaciones de reentrenamiento automatizado
    1. Crea un nuevo cuaderno de Databricks para el reentrenamiento.
    2. Programa el trabajo de reentrenamiento mediante trabajos de Databricks o Azure Data Factory.
    3. Automatiza el proceso de reentrenamiento en función del desfase de datos o los intervalos de tiempo.

- Implementación automática del modelo reentrenado
    1. Usa model_registry de MLflow para actualizar el modelo implementado automáticamente.
    2. Implemente el modelo reentrenado con el mismo proceso que en el paso 3.

## Paso 7: Prácticas de IA responsable
- Integración de la detección y mitigación de sesgo
    1. Usa Fairlearn o los scripts personalizados de Azure para evaluar el sesgo del modelo.
    2. Implementa estrategias de mitigación y registra los resultados mediante MLflow.

- Aplicación de directrices éticas para la implementación de LLM
    1. Garantiza la transparencia en las predicciones del modelo mediante el registro de datos de entrada y las predicciones.
    2. Establece directrices para el uso del modelo y garantiza el cumplimiento de los estándares éticos.

En este ejercicio se proporciona una guía completa para implementar LLMOps con Azure Databricks, que abarca la administración de modelos, la implementación, la supervisión, el reentrenamiento y las prácticas de IA responsable. Seguir estos pasos te ayudará a administrar y operar LLM de forma eficaz en un entorno de producción.    