---
lab:
  title: Introducción al aprendizaje automático en Azure Databricks
---

# Introducción al aprendizaje automático en Azure Databricks

En este ejercicio, explorará técnicas para preparar datos y entrenar modelos de aprendizaje automático en Azure Databricks.

Este ejercicio debería tardar en completarse **45** minutos aproximadamente.

## Antes de empezar

Necesitará una [suscripción de Azure](https://azure.microsoft.com/free) en la que tenga acceso de nivel administrativo.

## Aprovisiona un área de trabajo de Azure Databricks.

> **Sugerencia**: si ya tienes un área de trabajo de Azure Databricks, puedes omitir este procedimiento y usar el área de trabajo existente.

En este ejercicio, se incluye un script para aprovisionar una nueva área de trabajo de Azure Databricks. El script intenta crear un recurso de área de trabajo de Azure Databricks de nivel *Premium* en una región en la que la suscripción de Azure tiene cuota suficiente para los núcleos de proceso necesarios en este ejercicio, y da por hecho que la cuenta de usuario tiene permisos suficientes en la suscripción para crear un recurso de área de trabajo de Azure Databricks. Si se produjese un error en el script debido a cuota o permisos insuficientes, intenta [crear un área de trabajo de Azure Databricks de forma interactiva en Azure Portal](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. En un explorador web, inicia sesión en [Azure Portal](https://portal.azure.com) en `https://portal.azure.com`.
2. Usa el botón **[\>_]** a la derecha de la barra de búsqueda en la parte superior de la página para crear un nuevo Cloud Shell en Azure Portal, selecciona un entorno de ***PowerShell*** y crea almacenamiento si se te solicita. Cloud Shell proporciona una interfaz de línea de comandos en un panel situado en la parte inferior de Azure Portal, como se muestra a continuación:

    ![Azure Portal con un panel de Cloud Shell](./images/cloud-shell.png)

    > **Nota**: si creaste anteriormente un Cloud Shell que usa un entorno de *Bash*, usa el menú desplegable situado en la parte superior izquierda del panel de Cloud Shell para cambiarlo a ***PowerShell***.

3. Ten en cuenta que puedes cambiar el tamaño de Cloud Shell arrastrando la barra de separación en la parte superior del panel, o usando los iconos **&#8212;** , **&#9723;** y **X** en la parte superior derecha para minimizar, maximizar y cerrar el panel. Para obtener más información sobre el uso de Azure Cloud Shell, consulta la [documentación de Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

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
7. Espera a que se complete el script: normalmente puede tardar entre 5 y 10 minutos, pero en algunos casos puede tardar más. Mientras espera, revise el artículo [¿Qué es Databricks Machine Learning?](https://learn.microsoft.com/azure/databricks/machine-learning/) de la documentación de Azure Databricks.

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
1. Cambie el nombre predeterminado del cuaderno (**Cuaderno sin título *[fecha]***) por **Machine Learning** y en la lista desplegable **Conectar**, seleccione su clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

## Ingerir datos

El escenario de este ejercicio se basa en observaciones de pingüinos en la Antártida, con el objetivo de entrenar un modelo de Machine Learning para predecir la especie de un pingüino observado basándose en su ubicación y en las medidas de su cuerpo.

> **Cita**: El conjunto de datos sobre pingüinos que se usa en este ejercicio es un subconjunto de datos que han recopilado y hecho público el [Dr. Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) y la [Palmer Station, Antarctica LTER](https://pal.lternet.edu/), miembro de la [Long Term Ecological Research Network](https://lternet.edu/).

1. En la primera celda del cuaderno, escriba el siguiente código, que utiliza comandos de *shell* para descargar los datos de pingüinos de GitHub en el sistema de archivos utilizado por el clúster.

    ```bash
    %sh
    rm -r /dbfs/ml_lab
    mkdir /dbfs/ml_lab
    wget -O /dbfs/ml_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. Use la opción de menú **&#9656; Ejecutar celda** situado a la izquierda de la celda para ejecutarla. A continuación, espere a que se complete el trabajo de Spark ejecutado por el código.

## Exploración y limpieza de los datos
  
Ahora que ha ingerido el archivo de datos, puede cargarlo en un dataframe y verlo.

1. Debajo de la celda de código existente, usa el icono **+** para agregar una nueva celda de código. A continuación, en la nueva celda, escriba y ejecute el código siguiente para cargar los datos de los archivos y mostrarlos.

    ```python
   df = spark.read.format("csv").option("header", "true").load("/ml_lab/penguins.csv")
   display(df)
    ```

    El código inicia los *Trabajos de Spark* necesarios para cargar los datos, y la salida es un objeto *pyspark.sql.dataframe.DataFrame* llamado *df*. Verá esta información directamente debajo del código, y podrá usar el botón de alternancia **&#9656;** para expandir la salida **df: pyspark.sql.dataframe.DataFrame** y ver los detalles de las columnas que contiene y sus tipos de datos. Dado que estos datos se cargaron desde un archivo de texto y contenían algunos valores en blanco, Spark ha asignado un tipo de datos **cadena** a todas las columnas.
    
    Los datos en sí constan de medidas de los siguientes detalles de los pingüinos que se han observado en la Antártica:
    
    - **Isla**: La isla en la Antártica donde se observó el pingüino.
    - **CulmenLength**: La longitud en mm del culmen (pico) del pingüino.
    - **CulmenDepth**: La profundidad en mm del culmen del pingüino.
    - **FlipperLength**: La longitud en mm de la aleta del pingüino.
    - **BodyMass**: La masa corporal del pingüino en gramos.
    - **Especie**: Valor entero que representa la especie del pingüino:
      - **0**: *Adelia*
      - **1**: *Papúa*
      - **2**: *Barbijo*
      
    Nuestro objetivo en este proyecto es usar las características observadas de un pingüino (sus *características*) para predecir su especie (lo que en terminología de aprendizaje automático llamamos la *etiqueta*).
      
    Tenga en cuenta que algunas observaciones contienen valores de datos *nulos* o "ausentes" para algunas características. No es infrecuente que los datos sin procesar de origen que ingiere tengan problemas de este tipo, por lo que normalmente la primera etapa de un proyecto de aprendizaje automático consiste en explorar los datos a fondo y limpiarlos para hacerlos más adecuados para el entrenamiento de un modelo de Machine Learning.
    
1. Agregue una celda y úsela para ejecutar la siguiente celda para eliminar las filas con datos incompletos usando el método **dropna**, y para aplicar los tipos de datos apropiados a los datos usando el método **select** con las funciones **col** y **astype**.

    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   
   data = df.dropna().select(col("Island").astype("string"),
                              col("CulmenLength").astype("float"),
                             col("CulmenDepth").astype("float"),
                             col("FlipperLength").astype("float"),
                             col("BodyMass").astype("float"),
                             col("Species").astype("int")
                             )
   display(data)
    ```
    
    Una vez más, puede alternar los detalles del dataframe que se devuelve (esta vez denominado *data*) para comprobar que se han aplicado los tipos de datos, y puede revisar los datos para verificar que se han eliminado las filas que contenían datos incompletos.
    
    En un proyecto real, probablemente necesitaría realizar más exploración y limpieza de datos para corregir (o eliminar) errores en los datos, identificar y eliminar valores atípicos (valores atípicamente grandes o pequeños), o equilibrar los datos para que haya un número razonablemente igual de filas para cada etiqueta que esté intentando predecir.

    > **Sugerencia**: Puede obtener más información sobre los métodos y funciones que puede usar con los dataframes en la [Referencia de Spark SQL](https://spark.apache.org/docs/latest/sql-programming-guide.html).

## División de los datos

Para los fines de este ejercicio, supondremos que los datos están ahora convenientemente depurados y listos para que los usemos para entrenar un modelo de Machine Learning. El etiquetado que vamos a intentar predecir es una categoría específica o *clase* (la especie de un pingüino), por lo que el tipo de modelo de Machine Learning que necesitamos entrenar es un modelo de *clasificación*. La clasificación (junto con la *regresión*, que se usa para predecir un valor numérico) es una forma o *aprendizaje automático supervisado* en el que usamos datos de entrenamiento que incluyen valores conocidos para el etiquetado que queremos predecir. El proceso de entrenamiento de un modelo consiste en realidad en ajustar un algoritmo a los datos para calcular cómo se correlacionan los valores de las características con el valor conocido del etiquetado. Después podemos aplicar el modelo entrenado a una nueva observación de la que solo conocemos los valores de las características y hacer que prediga el valor de la etiqueta.

Para asegurarnos de que podemos tener confianza en nuestro modelo entrenado, el enfoque típico es entrenar el modelo con solo *algunos* de los datos, y reservar algunos datos con valores de etiquetado conocidos que podamos usar para probar el modelo entrenado y ver la precisión de sus predicciones. Para lograr este objetivo, dividiremos el conjunto completo de datos en dos subconjuntos aleatorios. Usaremos el 70 % de los datos para el entrenamiento y reservaremos el 30 % para las pruebas.

1. Agregue y ejecute una celda de código con el código siguiente para dividir los datos.

    ```python
   splits = data.randomSplit([0.7, 0.3])
   train = splits[0]
   test = splits[1]
   print ("Training Rows:", train.count(), " Testing Rows:", test.count())
    ```

## Realizar la ingeniería de características

Después de limpiar los datos sin procesar, los científicos de datos normalmente realizan algún trabajo adicional para prepararlos para el entrenamiento del modelo. Este proceso se conoce comúnmente como *ingeniería de características* e implica la optimización iterativa de las características en el conjunto de datos de entrenamiento para producir el mejor modelo posible. Las modificaciones de características específicas necesarias dependen de los datos y del modelo deseado, pero hay algunas tareas comunes de ingeniería de características con las que debe familiarizarse.

### Codificación de características de categorías

Normalmente, los algoritmos de aprendizaje automático se basan en la búsqueda de relaciones matemáticas entre características y etiquetas. Esto significa que suele ser mejor definir las características de los datos de entrenamiento como valores *numéricos*. En algunos casos, puede tener algunas características que son *categóricas* en lugar de numéricas y que se expresan como cadenas, por ejemplo, el nombre de la isla donde se produjo la observación de pingüinos en nuestro conjunto de datos. Sin embargo, la mayoría de los algoritmos esperan características numéricas, por lo que estos valores categóricos basados en cadenas deben *codificarse* como números. En este caso, usaremos un **StringIndexer** de la biblioteca **Spark MLLib** para codificar el nombre de la isla como un valor numérico asignando un índice entero único para cada nombre discreto de isla.

1. Ejecute el siguiente código para codificar los valores de la columna categórica **Isla** como índices numéricos.

    ```python
   from pyspark.ml.feature import StringIndexer

   indexer = StringIndexer(inputCol="Island", outputCol="IslandIdx")
   indexedData = indexer.fit(train).transform(train).drop("Island")
   display(indexedData)
    ```

    En los resultados, debería estar viendo que en lugar de un nombre de isla, cada fila tiene ahora una columna **IslandIdx** con un valor entero que representa la isla en la que se registró la observación.

### Normalización (escalado) de características numéricas

Ahora vamos a poner nuestra atención en los valores numéricos de nuestros datos. Estos valores (**LongitudCulmen**, **ProfundidadCulmen**, **LongitudAleta** y **MasaCorporal**) representan medidas de un tipo u otro, pero están en diferentes escalas. A la hora de entrenar un modelo, las unidades de medida no son tan importantes como las diferencias relativas entre las distintas observaciones, y las características representadas por números más grandes pueden dominar a menudo el algoritmo de entrenamiento del modelo, sesgando la importancia de la característica a la hora de calcular una predicción. Para mitigar esto, es habitual *normalizar* los valores numéricos de las características para que estén todos en la misma escala relativa (por ejemplo, un valor decimal entre 0,0 y 1,0).

El código que usaremos para hacer esto es un poco más complicado que la codificación categórica que hicimos anteriormente. Necesitamos escalar varios valores de columna al mismo tiempo, por lo que la técnica que usamos es crear una sola columna que contenga un *vector* (esencialmente una matriz) de todas las características numéricas, y después aplicar un escalador para producir una nueva columna vectorial con los valores normalizados equivalentes.

1. Use el siguiente código para normalizar las características numéricas y ver una comparación de las columnas de vectores prenormalizadas y normalizadas.

    ```python
   from pyspark.ml.feature import VectorAssembler, MinMaxScaler

   # Create a vector column containing all numeric features
   numericFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   numericColVector = VectorAssembler(inputCols=numericFeatures, outputCol="numericFeatures")
   vectorizedData = numericColVector.transform(indexedData)
   
   # Use a MinMax scaler to normalize the numeric values in the vector
   minMax = MinMaxScaler(inputCol = numericColVector.getOutputCol(), outputCol="normalizedFeatures")
   scaledData = minMax.fit(vectorizedData).transform(vectorizedData)
   
   # Display the data with numeric feature vectors (before and after scaling)
   compareNumerics = scaledData.select("numericFeatures", "normalizedFeatures")
   display(compareNumerics)
    ```

    La columna **numericFeatures** de los resultados contiene un vector para cada fila. El vector incluye cuatro valores numéricos sin escalar (las medidas originales del pingüino). Puede usar el botón de alternancia **&#9656;** para ver los valores discretos con mayor claridad.
    
    La columna **normalizedFeatures** también contiene un vector para cada observación de pingüinos, pero esta vez los valores del vector están normalizados a una escala relativa basada en los valores mínimo y máximo de cada medición.

### Preparación de características y etiquetas para el entrenamiento

Ahora, unamos todo y creemos una única columna que contenga todas las características (el nombre categórico codificado de la isla y las medidas normalizadas de los pingüinos), y otra columna que contenga la etiqueta de clase para la que queremos entrenar un modelo de predicción (la especie de pingüino).

1. Ejecute el código siguiente:

    ```python
   featVect = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="featuresVector")
   preppedData = featVect.transform(scaledData)[col("featuresVector").alias("features"), col("Species").alias("label")]
   display(preppedData)
    ```

    El vector de **características** contiene cinco valores (la isla codificada y la longitud del culmen, la profundidad del culmen, la longitud de las aletas y la masa corporal normalizadas). La etiqueta contiene un código entero simple que indica la clase de especie de pingüino.

## Entrenar un modelo de Machine Learning

Ahora que los datos de entrenamiento están preparados, puede usarlos para entrenar un modelo. Los modelos se entrenan mediante un *algoritmo* que intenta establecer una relación entre las características y las etiquetas. Como en este caso usted quiere entrenar un modelo que prediga una categoría de *clase* , necesita usar un algoritmo de *clasificación*. Existen muchos algoritmos para la clasificación; empecemos por uno bien establecido: la regresión logística, que intenta encontrar de forma iterativa los coeficientes óptimos que pueden aplicarse a los datos de las características en un cálculo logístico que predice la probabilidad de cada valor de etiquetado de clase. Para entrenar el modelo, ajustará el algoritmo de regresión logística a los datos de entrenamiento.

1. Ejecute el código siguiente para entrenar un modelo.

    ```python
   from pyspark.ml.classification import LogisticRegression

   lr = LogisticRegression(labelCol="label", featuresCol="features", maxIter=10, regParam=0.3)
   model = lr.fit(preppedData)
   print ("Model trained!")
    ```

    La mayoría de los algoritmos admiten parámetros que proporcionan algún control sobre la forma en que se entrena el modelo. En este caso, el algoritmo de regresión logística requiere que identifique la columna que contiene el vector de características y la columna que contiene la etiqueta conocida; y también le permite especificar el número máximo de iteraciones realizadas para encontrar coeficientes óptimos para el cálculo logístico, y un parámetro de regularización que se usa para evitar que el modelo se *sobreajuste* (en otras palabras, que establezca un cálculo logístico que funcione bien con los datos de entrenamiento, pero que no generalice bien cuando se aplica a datos nuevos).

## Probar el modelo

Ahora que tiene un modelo entrenado, puede probarlo con los datos que ha retenido. Para poder hacerlo, debe realizar las mismas transformaciones de ingeniería de características en los datos de prueba que aplicó a los datos de entrenamiento (en este caso, codificar el nombre de la isla y normalizar las medidas). A continuación, puede usar el modelo para predecir etiquetas para las características de los datos de prueba y comparar las etiquetas predichas con las etiquetas conocidas reales.

1. Use el código siguiente para preparar los datos de prueba y, a continuación, generar predicciones:

    ```python
   # Prepare the test data
   indexedTestData = indexer.fit(test).transform(test).drop("Island")
   vectorizedTestData = numericColVector.transform(indexedTestData)
   scaledTestData = minMax.fit(vectorizedTestData).transform(vectorizedTestData)
   preppedTestData = featVect.transform(scaledTestData)[col("featuresVector").alias("features"), col("Species").alias("label")]
   
   # Get predictions
   prediction = model.transform(preppedTestData)
   predicted = prediction.select("features", "probability", col("prediction").astype("Int"), col("label").alias("trueLabel"))
   display(predicted)
    ```

    Los resultados incluyen las columnas siguientes:
    
    - **características**: Los datos de características preparadas del conjunto de datos de prueba.
    - **probabilidad**: Probabilidad calculada por el modelo para cada clase. Consiste en un vector que contiene tres valores de probabilidad (porque hay tres clases) que suman un total de 1,0 (se supone que hay un 100 % de probabilidad de que el pingüino pertenezca a *una* de las tres clases de especies).
    - **predicción**: Etiqueta de clase predicha (la que tiene la probabilidad más alta).
    - **trueLabel**: Valor de etiqueta conocido real de los datos de prueba.
    
    Para evaluar la eficacia del modelo, basta con comparar las etiquetas predichas y verdaderas en estos resultados. Sin embargo, puede obtener métricas más significativas mediante un evaluador de modelos; en este caso, un evaluador de clasificación multiclase (porque hay varias etiquetas de clase posibles).

1. Use el código siguiente para obtener métricas de evaluación para un modelo de clasificación en función de los resultados de los datos de prueba:

    ```python
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   evaluator = MulticlassClassificationEvaluator(labelCol="label", predictionCol="prediction")
   
   # Simple accuracy
   accuracy = evaluator.evaluate(prediction, {evaluator.metricName:"accuracy"})
   print("Accuracy:", accuracy)
   
   # Individual class metrics
   labels = [0,1,2]
   print("\nIndividual class metrics:")
   for label in sorted(labels):
       print ("Class %s" % (label))
   
       # Precision
       precision = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                   evaluator.metricName:"precisionByLabel"})
       print("\tPrecision:", precision)
   
       # Recall
       recall = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                evaluator.metricName:"recallByLabel"})
       print("\tRecall:", recall)
   
       # F1 score
       f1 = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                            evaluator.metricName:"fMeasureByLabel"})
       print("\tF1 Score:", f1)
   
   # Weighted (overall) metrics
   overallPrecision = evaluator.evaluate(prediction, {evaluator.metricName:"weightedPrecision"})
   print("Overall Precision:", overallPrecision)
   overallRecall = evaluator.evaluate(prediction, {evaluator.metricName:"weightedRecall"})
   print("Overall Recall:", overallRecall)
   overallF1 = evaluator.evaluate(prediction, {evaluator.metricName:"weightedFMeasure"})
   print("Overall F1 Score:", overallF1)
    ```

    Las métricas de evaluación que se calculan para la clasificación multiclase incluyen:
    
    - **Exactitud**: Proporción de predicciones generales correctas.
    - Métricas por clase:
      - **Precisión**: Proporción de predicciones de esta clase que eran correctas.
      - **Coincidencia**: Proporción de instancias reales de esta clase que se predijeron correctamente.
      - **Puntuación F1**: Métrica combinada para precisión y coincidencia
    - Métricas de precisión, coincidencia y F1 combinadas (ponderadas) para todas las clases.
    
    > **Nota**: Inicialmente puede parecer que la métrica de precisión general proporciona la mejor manera de evaluar el rendimiento predictivo de un modelo. Sin embargo, tenga en cuenta esto. Suponga que los pingüinos papúa constituyen el 95 % de la población de pingüinos de su ubicación de estudio. Un modelo que siempre predice la etiqueta **1** (la clase para el papúa) tendrá una precisión de 0,95. Eso no significa que sea un excelente modelo para predecir una especie de pingüino en función de las características. Por eso los científicos de datos tienden a explorar métricas adicionales para comprender mejor cómo predice un modelo de clasificación para cada etiqueta de clase posible.

## Uso de una canalización

Ha entrenado el modelo realizando los pasos de ingeniería de características necesarios y, a continuación, ajustando un algoritmo a los datos. Para usar el modelo con algunos datos de prueba para generar predicciones (denominadas *inferencias*), tenía que aplicar los mismos pasos de ingeniería de características a los datos de prueba. Una manera más eficaz de compilar y usar modelos es encapsular los transformadores usados para preparar los datos y el modelo usado para entrenarlos en una *canalización*.

1. Use el código siguiente para crear una canalización que encapsula los pasos de preparación de datos y entrenamiento del modelo:

    ```python
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import LogisticRegression
   
   catFeature = "Island"
   numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   
   # Define the feature engineering and model training algorithm steps
   catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
   numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
   numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
   featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
   algo = LogisticRegression(labelCol="Species", featuresCol="Features", maxIter=10, regParam=0.3)
   
   # Chain the steps as stages in a pipeline
   pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
   # Use the pipeline to prepare data and fit the model algorithm
   model = pipeline.fit(train)
   print ("Model trained!")
    ```

    Dado que los pasos de ingeniería de características están ahora encapsulados en el modelo entrenado por la canalización, puede usar el modelo con los datos de prueba sin necesidad de aplicar cada transformación (serán aplicadas automáticamente por el modelo).

1. Use el código siguiente para aplicar la canalización a los datos de prueba:

    ```python
   prediction = model.transform(test)
   predicted = prediction.select("Features", "probability", col("prediction").astype("Int"), col("Species").alias("trueLabel"))
   display(predicted)
    ```

## Prueba de un algoritmo diferente

Hasta ahora ha entrenado un modelo de clasificación mediante el algoritmo de regresión logística. Vamos a cambiar esa fase de la canalización para probar un algoritmo diferente.

1. Ejecute el código siguiente para crear una canalización que use un algoritmo de árbol de decisión:

    ```python
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import DecisionTreeClassifier
   
   catFeature = "Island"
   numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   
   # Define the feature engineering and model steps
   catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
   numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
   numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
   featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
   algo = DecisionTreeClassifier(labelCol="Species", featuresCol="Features", maxDepth=10)
   
   # Chain the steps as stages in a pipeline
   pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
   # Use the pipeline to prepare data and fit the model algorithm
   model = pipeline.fit(train)
   print ("Model trained!")
    ```

    Esta vez, la canalización incluye las mismas etapas de preparación de características que antes, pero usa un algoritmo de *Árbol de decisión* para entrenar el modelo.
    
   1. Ejecute el código siguiente para usar la nueva canalización con los datos de prueba:

    ```python
   # Get predictions
   prediction = model.transform(test)
   predicted = prediction.select("Features", "probability", col("prediction").astype("Int"), col("Species").alias("trueLabel"))
   
   # Generate evaluation metrics
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   evaluator = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction")
   
   # Simple accuracy
   accuracy = evaluator.evaluate(prediction, {evaluator.metricName:"accuracy"})
   print("Accuracy:", accuracy)
   
   # Class metrics
   labels = [0,1,2]
   print("\nIndividual class metrics:")
   for label in sorted(labels):
       print ("Class %s" % (label))
   
       # Precision
       precision = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                       evaluator.metricName:"precisionByLabel"})
       print("\tPrecision:", precision)
   
       # Recall
       recall = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                evaluator.metricName:"recallByLabel"})
       print("\tRecall:", recall)
   
       # F1 score
       f1 = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                            evaluator.metricName:"fMeasureByLabel"})
       print("\tF1 Score:", f1)
   
   # Weighed (overall) metrics
   overallPrecision = evaluator.evaluate(prediction, {evaluator.metricName:"weightedPrecision"})
   print("Overall Precision:", overallPrecision)
   overallRecall = evaluator.evaluate(prediction, {evaluator.metricName:"weightedRecall"})
   print("Overall Recall:", overallRecall)
   overallF1 = evaluator.evaluate(prediction, {evaluator.metricName:"weightedFMeasure"})
   print("Overall F1 Score:", overallF1)
    ```

## Guardar el modelo

En realidad, probaría iterativamente el entrenamiento del modelo con distintos algoritmos (y parámetros) para encontrar el mejor modelo para los datos. Por ahora, nos quedaremos con el modelo de árbol de decisión que hemos entrenado. Vamos a guardarlo para poder usarlo más adelante con algunas observaciones de pingüinos nuevas.

1. Use el siguiente código para guardar el modelo:

    ```python
   model.save("/models/penguin.model")
    ```

    Ahora, cuando haya salido y haya avistado un nuevo pingüino, podrá cargar el modelo guardado y usarlo para predecir la especie del pingüino basándose en las mediciones que haya hecho de sus características. El uso de un modelo para generar predicciones a partir de nuevos datos se denomina *inferencia*.

1. Ejecute el código siguiente para cargar el modelo y usarlo para predecir la especie para una nueva observación de pingüinos:

    ```python
   from pyspark.ml.pipeline import PipelineModel

   persistedModel = PipelineModel.load("/models/penguin.model")
   
   newData = spark.createDataFrame ([{"Island": "Biscoe",
                                     "CulmenLength": 47.6,
                                     "CulmenDepth": 14.5,
                                     "FlipperLength": 215,
                                     "BodyMass": 5400}])
   
   
   predictions = persistedModel.transform(newData)
   display(predictions.select("Island", "CulmenDepth", "CulmenLength", "FlipperLength", "BodyMass", col("prediction").alias("PredictedSpecies")))
    ```

## Limpiar

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si ha terminado de explorar Azure Databricks, puede eliminar los recursos que ha creado para evitar costos innecesarios de Azure y liberar capacidad en la suscripción.

> **Más información**: Para más información, consulte la [Documentación de Spark MLLib](https://spark.apache.org/docs/latest/ml-guide.html).
