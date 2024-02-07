---
lab:
  title: Optimización de hiperparámetros para el aprendizaje automático en Azure Databricks
---

# Optimización de hiperparámetros para el aprendizaje automático en Azure Databricks

En este ejercicio, usará la biblioteca **Hyperopt** para optimizar los hiperparámetros para el entrenamiento del modelo de Machine Learning en Azure Databricks.

Este ejercicio debería tardar en completarse **30** minutos aproximadamente.

## Antes de empezar

Necesitará una [suscripción de Azure](https://azure.microsoft.com/free) en la que tenga acceso de nivel administrativo.

## Aprovisiona un área de trabajo de Azure Databricks.

> **Sugerencia**: Si ya tiene un área de trabajo de Azure Databricks, puede omitir este procedimiento y usar el área de trabajo existente.

En este ejercicio, se incluye un script para aprovisionar una nueva área de trabajo de Azure Databricks. El script intenta crear un recurso de área de trabajo de Azure Databricks de nivel *Premium* en una región en la que la suscripción de Azure tiene cuota suficiente para los núcleos de proceso necesarios en este ejercicio, y da por hecho que la cuenta de usuario tiene permisos suficientes en la suscripción para crear un recurso de área de trabajo de Azure Databricks. Si se produce un error en el script debido a una cuota o permisos insuficientes, puede intentar crear un área de trabajo de Azure Databricks de forma interactiva en Azure Portal.

1. En un explorador, inicia sesión en [Azure Portal](https://portal.azure.com) en `https://portal.azure.com`.
2. Usa el botón **[\>_]** a la derecha de la barra de búsqueda en la parte superior de la página para crear un nuevo Cloud Shell en Azure Portal, selecciona un entorno de ***PowerShell*** y crea almacenamiento si se te solicita. Cloud Shell proporciona una interfaz de línea de comandos en un panel situado en la parte inferior de Azure Portal, como se muestra a continuación:

    ![Azure Portal con un panel de Cloud Shell](./images/cloud-shell.png)

    > **Nota**: Si creaste anteriormente un Cloud Shell que usa un entorno de *Bash*, usa el menú desplegable situado en la parte superior izquierda del panel de Cloud Shell para cambiarlo a ***PowerShell***.

3. Tenga en cuenta que puede cambiar el tamaño de Cloud Shell arrastrando la barra de separación en la parte superior del panel, o usando los iconos **&#8212;** , **&#9723;** y **X** en la parte superior derecha para minimizar, maximizar y cerrar el panel. Para obtener más información sobre el uso de Azure Cloud Shell, consulte la [documentación de Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

4. En el panel de PowerShell, introduce los siguientes comandos para clonar este repositorio:

    ```
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

5. Una vez clonado el repositorio, escriba el siguiente comando para ejecutar el script **setup.ps1**, que aprovisiona un área de trabajo de Azure Databricks en una región disponible:

    ```
    ./mslearn-databricks/setup.ps1
    ```

6. Si se solicita, elige la suscripción que quieres usar (esto solo ocurrirá si tienes acceso a varias suscripciones de Azure).
7. Espera a que se complete el script: normalmente puede tardar entre 5 y 10 minutos, pero en algunos casos puede tardar más. Mientras espera, revise el artículo [Ajuste de hiperparámetros](https://learn.microsoft.com/azure/databricks/machine-learning/automl-hyperparam-tuning/) en la documentación de Azure Databricks.

## Crear un clúster

Azure Databricks es una plataforma de procesamiento distribuido que usa clústeres* de Apache Spark *para procesar datos en paralelo en varios nodos. Cada clúster consta de un nodo de controlador para coordinar el trabajo y nodos de trabajo para hacer tareas de procesamiento. En este ejercicio, crearás un clúster de *nodo único* para minimizar los recursos de proceso usados en el entorno de laboratorio (en los que se pueden restringir los recursos). En un entorno de producción, normalmente crearías un clúster con varios nodos de trabajo.

> **Sugerencia**: Si ya dispone de un clúster con una versión de runtime 13.3 LTS **<u>ML</u>** o superior en su área de trabajo de Azure Databricks, puede utilizarlo para completar este ejercicio y omitir este procedimiento.

1. En Azure Portal, vaya al grupo de recursos **msl-*xxxxxxx*** que se creó con el script (o al grupo de recursos que contiene el área de trabajo de Azure Databricks existente)
1. Seleccione el recurso Azure Databricks Service (llamado **databricks-*xxxxxxx*** si usó el script de instalación para crearlo).
1. En la página **Información general** del área de trabajo, usa el botón **Inicio del área de trabajo** para abrir el área de trabajo de Azure Databricks en una nueva pestaña del explorador; inicia sesión si se solicita.

    > **Sugerencia**: al usar el portal del área de trabajo de Databricks, se pueden mostrar varias sugerencias y notificaciones. Descártalas y sigue las instrucciones proporcionadas para completar las tareas de este ejercicio.

1. En la barra lateral de la izquierda, seleccione la tarea **(+) Nuevo** y luego seleccione **Clúster**.
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
    - **Tipo de nodo**: Standard_DS3_v2.
    - **Finaliza después de** *20* **minutos de inactividad**

1. Espera a que se cree el clúster. Esto puede tardar un par de minutos.

> **Nota**: si el clúster no se inicia, es posible que la suscripción no tenga cuota suficiente en la región donde se aprovisiona el área de trabajo de Azure Databricks. Para más información consulta [El límite de núcleos de la CPU impide la creación de clústeres](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si esto sucede, puedes intentar eliminar el área de trabajo y crear una nueva en otra región. Puedes especificar una región como parámetro para el script de configuración de la siguiente manera: `./mslearn-databricks/setup.ps1 eastus`

## Crear un cuaderno

Va a ejecutar código que use la biblioteca MLLib de Spark para entrenar un modelo de Machine Learning, por lo que el primer paso es crear un cuaderno en el área de trabajo.

1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**.
1. Cambie el nombre predeterminado del cuaderno (**Cuaderno sin título *[fecha]***) por **Ajuste de hiperparámetros** y en la lista desplegable **Conectar**, seleccione su clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

## Ingerir datos

El escenario de este ejercicio se basa en observaciones de pingüinos en la Antártida, con el objetivo de entrenar un modelo de Machine Learning para predecir la especie de un pingüino observado basándose en su ubicación y en las medidas de su cuerpo.

> **Cita**: El conjunto de datos sobre pingüinos que se usa en este ejercicio es un subconjunto de datos que han recopilado y hecho público el [Dr. Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) y la [Palmer Station, Antarctica LTER](https://pal.lternet.edu/), miembro de la [Long Term Ecological Research Network](https://lternet.edu/).

1. En la primera celda del cuaderno, escriba el siguiente código, que utiliza comandos de *shell* para descargar los datos de pingüinos de GitHub en el sistema de archivos Databricks (DBFS) utilizado por su clúster.

    ```bash
    %sh
    rm -r /dbfs/hyperopt_lab
    mkdir /dbfs/hyperopt_lab
    wget -O /dbfs/hyperopt_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. Utilice la opción **&#9656; Ejecutar celda** del menú situado en la parte superior derecha de la siguiente celda para ejecutarla. A continuación, espera a que se complete el trabajo Spark ejecutado por el código.
1. Ahora prepare los datos para el aprendizaje automático. Debajo de la celda de código existente, usa el icono **+** para agregar una nueva celda de código. A continuación, en la nueva celda, escriba y ejecute el siguiente código para:
    - Quitar las filas incompletas
    - Aplicar tipos de datos adecuados
    - Ver un ejemplo aleatorio de los datos
    - Dividir los datos en dos conjuntos: uno de entrenamiento y otro de prueba.


    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   
   data = spark.read.format("csv").option("header", "true").load("/hyperopt_lab/penguins.csv")
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

## Optimización de los valores de hiperparámetros para entrenar un modelo

Para entrenar un modelo de Machine Learning, ajuste las características a un algoritmo que calcule la etiqueta más probable. Los algoritmos toman los datos de entrenamiento como parámetro e intentan calcular una relación matemática entre las características y las etiquetas. Además de los datos, la mayoría de los algoritmos usan uno o varios *hiperparámetros* para influir en la forma en que se calcula la relación. Determinar los valores óptimos de hiperparámetros es una parte importante del proceso de entrenamiento del modelo iterativo.

Para ayudarle a determinar los valores óptimos de hiperparámetros, Azure Databricks incluye compatibilidad con **Hyperopt**, una biblioteca que le permite probar varios valores de hiperparámetros y encontrar la mejor combinación para los datos.

El primer paso para usar Hyperopt es crear una función que:

- Entrene un modelo mediante uno o varios valores de hiperparámetros que se pasan a la función como parámetros.
- Calcule una métrica de rendimiento que se puede usar para medir la *pérdida* (cuánto se aleja el modelo del rendimiento de predicción perfecto).
- Devuelva el valor de pérdida para que se pueda optimizar (minimizar) iterativamente mediante la prueba de valores de hiperparámetros diferentes.

1. Agregue una nueva celda y use el código siguiente para crear una función que use los datos de los pingüinos para crear un modelo de clasificación que prediga la especie de un pingüino en función de su ubicación y medidas:

    ```python
   from hyperopt import STATUS_OK
   import mlflow
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import DecisionTreeClassifier
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   def objective(params):
       # Train a model using the provided hyperparameter value
       catFeature = "Island"
       numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
       catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
       numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
       numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
       featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
       mlAlgo = DecisionTreeClassifier(labelCol="Species",    
                                       featuresCol="Features",
                                       maxDepth=params['MaxDepth'], maxBins=params['MaxBins'])
       pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, mlAlgo])
       model = pipeline.fit(train)
       
       # Evaluate the model to get the target metric
       prediction = model.transform(test)
       eval = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction", metricName="accuracy")
       accuracy = eval.evaluate(prediction)
       
       # Hyperopt tries to minimize the objective function, so you must return the negative accuracy.
       return {'loss': -accuracy, 'status': STATUS_OK}
    ```

1. Agregue una nueva celda y use el código siguiente para:
    - Definir un espacio de búsqueda que especifique el intervalo de valores que se va a usar para uno o varios hiperparámetros (consulte [Definición de un espacio de búsqueda](http://hyperopt.github.io/hyperopt/getting-started/search_spaces/) en la documentación de Hyperopt para más información).
    - Especifique el algoritmo de Hyperopt que desea usar (consulte [Algoritmos](http://hyperopt.github.io/hyperopt/#algorithms) en la documentación de Hyperopt para más información).
    - Use la función **hyperopt.fmin** para llamar a la función de entrenamiento repetidamente e intentar minimizar la pérdida.

    ```python
   from hyperopt import fmin, tpe, hp
   
   # Define a search space for two hyperparameters (maxDepth and maxBins)
   search_space = {
       'MaxDepth': hp.randint('MaxDepth', 10),
       'MaxBins': hp.choice('MaxBins', [10, 20, 30])
   }
   
   # Specify an algorithm for the hyperparameter optimization process
   algo=tpe.suggest
   
   # Call the training function iteratively to find the optimal hyperparameter values
   argmin = fmin(
     fn=objective,
     space=search_space,
     algo=algo,
     max_evals=6)
   
   print("Best param values: ", argmin)
    ```

1. Observe que el código ejecuta iterativamente la función de entrenamiento 6 veces (en función de la configuración de **max_evals**). MLflow registra cada ejecución y puede usar el botón de alternancia **&#9656;** para expandir la salida de la **ejecución de MLflow** en la celda de código y seleccionar el hipervínculo **experimento** para verlas. A cada ejecución se le asigna un nombre aleatorio y puede ver cada uno de ellos en el visor de ejecución de MLflow para consultar los detalles de los parámetros y las métricas que se registraron.
1. Cuando finalicen todas las ejecuciones, observe que el código muestra los detalles de los mejores valores de hiperparámetros que se encontraron (es decir, la combinación que dio lugar a la menor pérdida). En este caso, el parámetro **MaxBins** se define como una opción de una lista de tres valores posibles (10, 20 y 30): el mejor valor indica el elemento de base cero de la lista (por lo que 0=10, 1=20 y 2=30). El parámetro **MaxDepth** se define como un entero aleatorio entre 0 y 10, y se muestra el valor entero que dio el mejor resultado. Para más información sobre cómo especificar ámbitos de valor de hiperparámetros para espacios de búsqueda, consulte [Expresiones de parámetros](http://hyperopt.github.io/hyperopt/getting-started/search_spaces/#parameter-expressions) en la documentación de Hyperopt.

## Uso de la clase Trials para registrar los detalles de la ejecución

Además de usar las ejecuciones del experimento de MLflow para registrar los detalles de cada iteración, también puede usar la clase**hyperopt.Trials** para registrar y ver los detalles de cada ejecución.

1. Agregue una nueva celda y use el código siguiente para ver los detalles de cada ejecución registrada por la clase **Trials**:

    ```python
   from hyperopt import Trials
   
   # Create a Trials object to track each run
   trial_runs = Trials()
   
   argmin = fmin(
     fn=objective,
     space=search_space,
     algo=algo,
     max_evals=3,
     trials=trial_runs)
   
   print("Best param values: ", argmin)
   
   # Get details from each trial run
   print ("trials:")
   for trial in trial_runs.trials:
       print ("\n", trial)
    ```

## Limpiar

En el portal de Azure Databricks, en la página **Proceso**, seleccione el clúster y **&#9632; Finalizar** para apagarlo.

Si ha terminado de explorar Azure Databricks, puede eliminar los recursos que ha creado para evitar costos innecesarios de Azure y liberar capacidad en la suscripción.

> **Más información**: Para más información, consulte [Ajuste de hiperparámetros](https://learn.microsoft.com/azure/databricks/machine-learning/automl-hyperparam-tuning/) en la documentación de Azure Databricks.