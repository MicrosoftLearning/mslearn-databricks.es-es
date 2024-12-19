---
lab:
  title: "Uso de MLflow en Azure\_Databricks"
---

# Uso de MLflow en Azure Databricks

En este ejercicio, explorará el uso de MLflow para entrenar y servir modelos de Machine Learning en Azure Databricks.

Este ejercicio debería tardar en completarse **45** minutos aproximadamente.

> **Nota**: la interfaz de usuario de Azure Databricks está sujeta a una mejora continua. Es posible que la interfaz de usuario haya cambiado desde que se escribieron las instrucciones de este ejercicio.

## Antes de empezar

Necesitará una [suscripción de Azure](https://azure.microsoft.com/free) en la que tenga acceso de nivel administrativo.

## Aprovisiona un área de trabajo de Azure Databricks.

> **Nota**: Para este ejercicio, necesita un área de trabajo de Azure Databricks **Premium** en una región en la que se admita el *servicio de modelo*. Consulte las [Regiones de Azure Databricks](https://learn.microsoft.com/azure/databricks/resources/supported-regions) para obtener más información sobre sus funcionalidades regionales. Si ya tiene un área de trabajo de Azure Databricks *Premium* o *de prueba* en una región adecuada, puede omitir este procedimiento y usar el área de trabajo existente.

En este ejercicio se incluye un script para aprovisionar una nueva área de trabajo de Azure Databricks. El script intenta crear un recurso de área de trabajo de Azure Databricks de nivel *Premium* en una región en la que la suscripción de Azure tiene cuota suficiente para los núcleos de proceso necesarios en este ejercicio, y da por hecho que la cuenta de usuario tiene permisos suficientes en la suscripción para crear un recurso de área de trabajo de Azure Databricks. Si se produjese un error en el script debido a cuota o permisos insuficientes, intenta [crear un área de trabajo de Azure Databricks de forma interactiva en Azure Portal](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. En un explorador web, inicia sesión en [Azure Portal](https://portal.azure.com) en `https://portal.azure.com`.
2. Usa el botón **[\>_]** situado a la derecha de la barra de búsqueda en la parte superior de la página para crear una nueva instancia de Cloud Shell en Azure Portal, para lo que deberás seleccionar un entorno de ***PowerShell***. Cloud Shell proporciona una interfaz de línea de comandos en un panel situado en la parte inferior de Azure Portal, como se muestra a continuación:

    ![Azure Portal con un panel de Cloud Shell](./images/cloud-shell.png)

    > **Nota**: si has creado anteriormente una instancia de Cloud Shell que usa un entorno de *Bash*, cámbiala a ***PowerShell***.

3. Ten en cuenta que puedes cambiar el tamaño de la instancia de Cloud Shell. Para ello, arrastra la barra de separación de la parte superior del panel o utiliza los iconos **&#8212;**, **&#10530;** y **X** de la parte superior derecha del panel para minimizar, maximizar y cerrar el panel. Para obtener más información sobre el uso de Azure Cloud Shell, consulta la [documentación de Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

4. En el panel de PowerShell, introduce los siguientes comandos para clonar este repositorio:

    ```
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

5. Una vez clonado el repositorio, escribe el siguiente comando para ejecutar el script **setup.ps1**, que aprovisiona un área de trabajo de Azure Databricks en una región disponible:

    ```
    ./mslearn-databricks/setup.ps1
    ```

6. Si se solicita, elige la suscripción que quieres usar (esto solo ocurrirá si tienes acceso a varias suscripciones de Azure).
7. Espera a que se complete el script: normalmente puede tardar entre 5 y 10 minutos, pero en algunos casos puede tardar más. Mientras espera, revise el artículo [Guía de MLflow](https://learn.microsoft.com/azure/databricks/mlflow/) en la documentación de Azure Databricks.

## Crear un clúster

Azure Databricks es una plataforma de procesamiento distribuido que usa clústeres* de Apache Spark *para procesar datos en paralelo en varios nodos. Cada clúster consta de un nodo de controlador para coordinar el trabajo y nodos de trabajo para hacer tareas de procesamiento. En este ejercicio, crearás un clúster de *nodo único* para minimizar los recursos de proceso usados en el entorno de laboratorio (en los que se pueden restringir los recursos). En un entorno de producción, normalmente crearías un clúster con varios nodos de trabajo.

> **Sugerencia**: Si ya dispone de un clúster con una versión de runtime 13.3 LTS **<u>ML</u>** o superior en su área de trabajo de Azure Databricks, puede utilizarlo para completar este ejercicio y omitir este procedimiento.

1. En Azure Portal, vaya al grupo de recursos **msl-*xxxxxxx*** que se creó con el script (o al grupo de recursos que contiene el área de trabajo de Azure Databricks existente)
1. Selecciona el recurso Azure Databricks Service (llamado **databricks-*xxxxxxx*** si usaste el script de instalación para crearlo).
1. En la página **Información general** del área de trabajo, usa el botón **Inicio del área de trabajo** para abrir el área de trabajo de Azure Databricks en una nueva pestaña del explorador; inicia sesión si se solicita.

    > **Sugerencia**: al usar el portal del área de trabajo de Databricks, se pueden mostrar varias sugerencias y notificaciones. Descártalas y sigue las instrucciones proporcionadas para completar las tareas de este ejercicio.

1. En la barra lateral de la izquierda, selecciona la tarea **(+) Nuevo** y luego selecciona **Clúster**.
1. En la página **Nuevo clúster**, crea un clúster con la siguiente configuración:
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

1. Espera a que se cree el clúster. Esto puede tardar un par de minutos.

> **Nota**: si el clúster no se inicia, es posible que la suscripción no tenga cuota suficiente en la región donde se aprovisiona el área de trabajo de Azure Databricks. Para obtener más información, consulta [El límite de núcleos de la CPU impide la creación de clústeres](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si esto sucede, puedes intentar eliminar el área de trabajo y crear una nueva en otra región. Puedes especificar una región como parámetro para el script de configuración de la siguiente manera: `./mslearn-databricks/setup.ps1 eastus`

## Crear un cuaderno

Va a ejecutar código que use la biblioteca MLLib de Spark para entrenar un modelo de Machine Learning, por lo que el primer paso es crear un cuaderno en el área de trabajo.

1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**.
1. Cambie el nombre predeterminado del cuaderno (**Cuaderno sin título *[fecha]***) a **MLflow** y, en la lista desplegable **Conectar**, seleccione su clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

## Ingesta y preparación de datos

El escenario de este ejercicio se basa en observaciones de pingüinos en la Antártida, con el objetivo de entrenar un modelo de Machine Learning para predecir la especie de un pingüino observado basándose en su ubicación y en las medidas de su cuerpo.

> **Cita**: El conjunto de datos sobre pingüinos que se usa en este ejercicio es un subconjunto de datos que han recopilado y hecho público el [Dr. Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) y la [Palmer Station, Antarctica LTER](https://pal.lternet.edu/), miembro de la [Long Term Ecological Research Network](https://lternet.edu/).

1. En la primera celda del cuaderno, escriba el siguiente código, que utiliza comandos de *shell* para descargar los datos de pingüinos de GitHub en el sistema de archivos utilizado por el clúster.

    ```bash
    %sh
    rm -r /dbfs/mlflow_lab
    mkdir /dbfs/mlflow_lab
    wget -O /dbfs/mlflow_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. Use la opción de menú **&#9656; Ejecutar celda** situado a la izquierda de la celda para ejecutarla. A continuación, espera a que se complete el trabajo Spark ejecutado por el código.

1. Ahora prepare los datos para el aprendizaje automático. Debajo de la celda de código existente, usa el icono **+** para agregar una nueva celda de código. A continuación, en la nueva celda, escriba y ejecute el siguiente código para:
    - Quitar las filas incompletas
    - Aplicar tipos de datos adecuados
    - Ver un ejemplo aleatorio de los datos
    - Dividir los datos en dos conjuntos: uno de entrenamiento y otro de prueba.


    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   
   data = spark.read.format("csv").option("header", "true").load("/mlflow_lab/penguins.csv")
   data = data.dropna().select(col("Island").astype("string"),
                               col("CulmenLength").astype("float"),
                               col("CulmenDepth").astype("float"),
                               col("FlipperLength").astype("float"),
                               col("BodyMass").astype("float"),
                               col("Species").astype("int")
                             )
   display(data.sample(0.2))
   
   splits = data.randomSplit([0.7, 0.3])
   train = splits[0]
   test = splits[1]
   print ("Training Rows:", train.count(), " Testing Rows:", test.count())
    ```

## Ejecutar un experimento de MLflow

MLflow permite ejecutar experimentos que realizan un seguimiento del proceso de entrenamiento del modelo y las métricas de evaluación de registros. Esta capacidad para registrar detalles de las ejecuciones de entrenamiento del modelo puede ser extremadamente útil en el proceso iterativo de creación de un modelo de Machine Learning eficaz.

Puede usar las mismas bibliotecas y técnicas que normalmente se usan para entrenar y evaluar un modelo (en este caso, usaremos la biblioteca MLLib de Spark), pero lo haremos en el contexto de un experimento de MLflow que incluya comandos adicionales para registrar métricas e información importantes durante el proceso.

1. Agregue una nueva celda y escriba el código siguiente en ella:

    ```python
   import mlflow
   import mlflow.spark
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import LogisticRegression
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   import time
   
   # Start an MLflow run
   with mlflow.start_run():
       catFeature = "Island"
       numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
     
       # parameters
       maxIterations = 5
       regularization = 0.5
   
       # Define the feature engineering and model steps
       catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
       numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
       numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
       featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
       algo = LogisticRegression(labelCol="Species", featuresCol="Features", maxIter=maxIterations, regParam=regularization)
   
       # Chain the steps as stages in a pipeline
       pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
       # Log training parameter values
       print ("Training Logistic Regression model...")
       mlflow.log_param('maxIter', algo.getMaxIter())
       mlflow.log_param('regParam', algo.getRegParam())
       model = pipeline.fit(train)
      
       # Evaluate the model and log metrics
       prediction = model.transform(test)
       metrics = ["accuracy", "weightedRecall", "weightedPrecision"]
       for metric in metrics:
           evaluator = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction", metricName=metric)
           metricValue = evaluator.evaluate(prediction)
           print("%s: %s" % (metric, metricValue))
           mlflow.log_metric(metric, metricValue)
   
           
       # Log the model itself
       unique_model_name = "classifier-" + str(time.time())
       mlflow.spark.log_model(model, unique_model_name, mlflow.spark.get_default_conda_env())
       modelpath = "/model/%s" % (unique_model_name)
       mlflow.spark.save_model(model, modelpath)
       
       print("Experiment run complete.")
    ```

1. Cuando finalice la ejecución del experimento, en la celda de código, si es necesario, use el botón de alternancia **&#9656;** para expandir los detalles de la **ejecución de MLflow**. Use el hipervínculo **experimento** que aparece allí para abrir la página MLflow que muestra las ejecuciones del experimento. A cada ejecución se le asigna un nombre único.
1. Seleccione la ejecución más reciente y vea sus detalles. Tenga en cuenta que puede expandir secciones para ver los **Parámetros** y **Métricas** que se registraron y puede ver los detalles del modelo que se entrenó y guardó.

    > **Sugerencia**: También puede usar el icono de **experimentos de MLflow** en el menú de la barra lateral de la derecha de este cuaderno para ver los detalles de las ejecuciones de experimentos.

## Creación de una función

En los proyectos de aprendizaje automático, los científicos de datos suelen probar modelos de entrenamiento con distintos parámetros, registrando los resultados cada vez. Para ello, es habitual crear una función que encapsula el proceso de entrenamiento y lo llama con los parámetros que desea probar.

1. En una nueva celda, ejecute el código siguiente para crear una función basada en el código de entrenamiento que usó anteriormente:

    ```python
   def train_penguin_model(training_data, test_data, maxIterations, regularization):
       import mlflow
       import mlflow.spark
       from pyspark.ml import Pipeline
       from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
       from pyspark.ml.classification import LogisticRegression
       from pyspark.ml.evaluation import MulticlassClassificationEvaluator
       import time
   
       # Start an MLflow run
       with mlflow.start_run():
   
           catFeature = "Island"
           numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   
           # Define the feature engineering and model steps
           catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
           numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
           numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
           featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
           algo = LogisticRegression(labelCol="Species", featuresCol="Features", maxIter=maxIterations, regParam=regularization)
   
           # Chain the steps as stages in a pipeline
           pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
           # Log training parameter values
           print ("Training Logistic Regression model...")
           mlflow.log_param('maxIter', algo.getMaxIter())
           mlflow.log_param('regParam', algo.getRegParam())
           model = pipeline.fit(training_data)
   
           # Evaluate the model and log metrics
           prediction = model.transform(test_data)
           metrics = ["accuracy", "weightedRecall", "weightedPrecision"]
           for metric in metrics:
               evaluator = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction", metricName=metric)
               metricValue = evaluator.evaluate(prediction)
               print("%s: %s" % (metric, metricValue))
               mlflow.log_metric(metric, metricValue)
   
   
           # Log the model itself
           unique_model_name = "classifier-" + str(time.time())
           mlflow.spark.log_model(model, unique_model_name, mlflow.spark.get_default_conda_env())
           modelpath = "/model/%s" % (unique_model_name)
           mlflow.spark.save_model(model, modelpath)
   
           print("Experiment run complete.")
    ```

1. En una nueva celda, use el siguiente código para llamar a la función:

    ```python
   train_penguin_model(train, test, 10, 0.2)
    ```

1. Vea los detalles del experimento de MLflow para la segunda ejecución.

## Registro e implementación de un modelo con MLflow

Además de realizar un seguimiento de los detalles de las ejecuciones de experimentos de entrenamiento, puede usar MLflow para administrar los modelos de aprendizaje automático que ha entrenado. Ya ha registrado el modelo entrenado por cada ejecución de experimento. También puede *registrar* modelos e implementarlos para que se puedan utilizar en las aplicaciones cliente.

> **Nota**: El servicio de modelos solo se admite en áreas de trabajo de Azure Databricks *Premium* y está restringido a [determinadas regiones](https://learn.microsoft.com/azure/databricks/resources/supported-regions).

1. Vea la página de detalles de la ejecución de experimento más reciente.
1. Use el botón **Registrar modelo** para registrar el modelo que se registró en ese experimento y, cuando se le solicite, cree un nuevo modelo denominado **Pronosticador de pingüinos**.
1. Cuando se haya registrado el modelo, vea la página **Modelos** (en la barra de navegación de la izquierda) y seleccione el modelo **Pronosticador de pingüinos**.
1. En la página del modelo **Pronosticador de pingüinos**, use el botón **Usar modelo para la inferencia** para crear un nuevo punto de conexión en tiempo real con los siguientes valores:
    - **Modelo**: Pronosticador de pingüinos
    - **Versión del modelo**: 1
    - **Punto de conexión**: predict-penguin
    - **Tamaño de proceso**: Pequeña

    El punto de conexión de servicio se hospeda en un nuevo clúster, que puede tardar varios minutos en crearse.
  
1. Cuando se haya creado el punto de conexión, use el botón **Consultar punto de conexión** situado en la parte superior derecha para abrir una interfaz desde la que puede probarlo. A continuación, en la interfaz de prueba, en la pestaña **Explorador**, escriba la siguiente solicitud JSON y use el botón **Enviar solicitud** para llamar al punto de conexión y generar una predicción.

    ```json
    {
      "dataframe_records": [
      {
         "Island": "Biscoe",
         "CulmenLength": 48.7,
         "CulmenDepth": 14.1,
         "FlipperLength": 210,
         "BodyMass": 4450
      }
      ]
    }
    ```

1. Experimente con algunos valores diferentes para las características de los pingüinos y observe los resultados que se devuelven. A continuación, cierre la interfaz de prueba.

## Eliminación del punto de conexión

Cuando el punto de conexión ya no sea necesario, debe eliminarlo para evitar costos innecesarios.

En la página de punto de conexión **predict-penguin**, en el menú **&#8285;**, seleccione **Eliminar**.

## Limpiar

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si ha terminado de explorar Azure Databricks, puede eliminar los recursos que ha creado para evitar costos innecesarios de Azure y liberar capacidad en la suscripción.

> **Más información**: Para más información, consulte la [Documentación de Spark MLLib](https://spark.apache.org/docs/latest/ml-guide.html).
