---
lab:
  title: Optimización de hiperparámetros para el aprendizaje automático en Azure Databricks
---

# Optimización de hiperparámetros para el aprendizaje automático en Azure Databricks

En este ejercicio, usarás la biblioteca **Optuna** para optimizar los hiperparámetros para el entrenamiento del modelo de Machine Learning en Azure Databricks.

Este ejercicio debería tardar en completarse **30** minutos aproximadamente.

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

    ```
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

5. Una vez clonado el repositorio, escribe el siguiente comando para ejecutar el script **setup.ps1**, que aprovisiona un área de trabajo de Azure Databricks en una región disponible:

    ```
    ./mslearn-databricks/setup.ps1
    ```

6. Si se solicita, elige la suscripción que quieres usar (esto solo ocurrirá si tienes acceso a varias suscripciones de Azure).
7. Espera a que se complete el script: normalmente puede tardar entre 5 y 10 minutos, pero en algunos casos puede tardar más. Mientras espera, revise el artículo [Ajuste de hiperparámetros](https://learn.microsoft.com/azure/databricks/machine-learning/automl-hyperparam-tuning/) en la documentación de Azure Databricks.

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
1. Cambie el nombre predeterminado del cuaderno (**Cuaderno sin título *[fecha]***) por **Ajuste de hiperparámetros** y en la lista desplegable **Conectar**, seleccione su clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

## Ingerir datos

El escenario de este ejercicio se basa en observaciones de pingüinos en la Antártida, con el objetivo de entrenar un modelo de Machine Learning para predecir la especie de un pingüino observado basándose en su ubicación y en las medidas de su cuerpo.

> **Cita**: El conjunto de datos sobre pingüinos que se usa en este ejercicio es un subconjunto de datos que han recopilado y hecho público el [Dr. Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) y la [Palmer Station, Antarctica LTER](https://pal.lternet.edu/), miembro de la [Long Term Ecological Research Network](https://lternet.edu/).

1. En la primera celda del cuaderno, escriba el siguiente código, que utiliza comandos de *shell* para descargar los datos de pingüinos de GitHub en el sistema de archivos utilizado por el clúster.

    ```bash
    %sh
    rm -r /dbfs/hyperparam_tune_lab
    mkdir /dbfs/hyperparam_tune_lab
    wget -O /dbfs/hyperparam_tune_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
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
   
   data = spark.read.format("csv").option("header", "true").load("/hyperparam_tune_lab/penguins.csv")
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

Para ayudarte a determinar los valores óptimos de hiperparámetros, Azure Databricks incluye compatibilidad con [**Optuna**](https://optuna.readthedocs.io/en/stable/index.html), una biblioteca que te permite probar varios valores de hiperparámetros y encontrar la mejor combinación para los datos.

El primer paso para usar Optuna es crear una función que:

- Entrene un modelo mediante uno o varios valores de hiperparámetros que se pasan a la función como parámetros.
- Calcule una métrica de rendimiento que se puede usar para medir la *pérdida* (cuánto se aleja el modelo del rendimiento de predicción perfecto).
- Devuelva el valor de pérdida para que se pueda optimizar (minimizar) iterativamente mediante la prueba de valores de hiperparámetros diferentes.

1. Agrega una nueva celda y usa el código siguiente para crear una función que define el rango de los valores que se usan para los hiperámetros y usa los datos de los pingüinos para entrenar un modelo de clasificación que prediga la especie de un pingüino en función de su ubicación y medidas:

    ```python
   import optuna
   import mlflow # if you wish to log your experiments
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import DecisionTreeClassifier
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   def objective(trial):
       # Suggest hyperparameter values (maxDepth and maxBins):
       max_depth = trial.suggest_int("MaxDepth", 0, 9)
       max_bins = trial.suggest_categorical("MaxBins", [10, 20, 30])

       # Define pipeline components
       cat_feature = "Island"
       num_features = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
       catIndexer = StringIndexer(inputCol=cat_feature, outputCol=cat_feature + "Idx")
       numVector = VectorAssembler(inputCols=num_features, outputCol="numericFeatures")
       numScaler = MinMaxScaler(inputCol=numVector.getOutputCol(), outputCol="normalizedFeatures")
       featureVector = VectorAssembler(inputCols=[cat_feature + "Idx", "normalizedFeatures"], outputCol="Features")

       dt = DecisionTreeClassifier(
           labelCol="Species",
           featuresCol="Features",
           maxDepth=max_depth,
           maxBins=max_bins
       )

       pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, dt])
       model = pipeline.fit(train)

       # Evaluate the model using accuracy.
       predictions = model.transform(test)
       evaluator = MulticlassClassificationEvaluator(
           labelCol="Species",
           predictionCol="prediction",
           metricName="accuracy"
       )
       accuracy = evaluator.evaluate(predictions)

       # Since Optuna minimizes the objective, return negative accuracy.
       return -accuracy
    ```

1. Agrega una nueva celda y usa el código siguiente para ejecutar el experimento de optimización:

    ```python
   # Optimization run with 5 trials:
   study = optuna.create_study()
   study.optimize(objective, n_trials=5)

   print("Best param values from the optimization run:")
   print(study.best_params)
    ```

1. Observa que el código ejecuta iterativamente la función de entrenamiento 5 veces al tratar de minimizar la pérdida (en función de la configuración de **n_trials**). MLflow registra cada prueba y puedes usar la tecla de alternancia **&#9656;** para expandir la salida de la **ejecución de MLflow** en la celda de código y seleccionar el hipervínculo **experimento** para verlas. A cada ejecución se le asigna un nombre aleatorio y puede ver cada uno de ellos en el visor de ejecución de MLflow para consultar los detalles de los parámetros y las métricas que se registraron.
1. Cuando finalicen todas las ejecuciones, observe que el código muestra los detalles de los mejores valores de hiperparámetros que se encontraron (es decir, la combinación que dio lugar a la menor pérdida). En este caso, el parámetro **MaxBins** se define como una opción de una lista de tres valores posibles (10, 20 y 30): el mejor valor indica el elemento de base cero de la lista (por lo que 0=10, 1=20 y 2=30). El parámetro **MaxDepth** se define como un entero aleatorio entre 0 y 10, y se muestra el valor entero que dio el mejor resultado. 

## Limpieza

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si ha terminado de explorar Azure Databricks, puede eliminar los recursos que ha creado para evitar costos innecesarios de Azure y liberar capacidad en la suscripción.

> **Más información**: Para más información, consulte [Ajuste de hiperparámetros](https://learn.microsoft.com/azure/databricks/machine-learning/automl-hyperparam-tuning/) en la documentación de Azure Databricks.
