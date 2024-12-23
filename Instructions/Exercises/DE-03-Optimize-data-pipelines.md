---
lab:
  title: "Optimización de canalizaciones de datos para mejorar el rendimiento en Azure\_Databricks"
---

# Optimización de canalizaciones de datos para mejorar el rendimiento en Azure Databricks

La optimización de canalizaciones de datos en Azure Databricks puede mejorar significativamente el rendimiento y la eficacia. El uso de Auto Loader para la ingesta incremental de datos, junto con la capa de almacenamiento de Delta Lake, garantiza la confiabilidad y las transacciones ACID. La implementación de sal puede evitar la asimetría de datos, mientras que la agrupación en clústeres de orden Z optimiza las lecturas de archivos mediante la colocación de información relacionada. Las funcionalidades de ajuste automático de Azure Databricks y el optimizador basado en costos pueden mejorar aún más el rendimiento ajustando la configuración en función de los requisitos de la carga de trabajo.

Este laboratorio se tarda aproximadamente **30** minutos en completarse.

> **Nota**: la interfaz de usuario de Azure Databricks está sujeta a una mejora continua. Es posible que la interfaz de usuario haya cambiado desde que se escribieron las instrucciones de este ejercicio.

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

7. Espera a que se complete el script: normalmente tarda unos 5 minutos, pero en algunos casos puede tardar más. Mientras esperas, revisa los artículos [¿Qué es el cargador automático?](https://learn.microsoft.com/azure/databricks/ingestion/cloud-object-storage/auto-loader/) y [Optimización del diseño del archivo de datos](https://learn.microsoft.com/azure/databricks/delta/optimize) en la documentación de Azure Databricks.

## Crear un clúster

Azure Databricks es una plataforma de procesamiento distribuido que usa clústeres* de Apache Spark *para procesar datos en paralelo en varios nodos. Cada clúster consta de un nodo de controlador para coordinar el trabajo y nodos de trabajo para hacer tareas de procesamiento. En este ejercicio, crearás un clúster de *nodo único* para minimizar los recursos de proceso usados en el entorno de laboratorio (en los que se pueden restringir los recursos). En un entorno de producción, normalmente crearías un clúster con varios nodos de trabajo.

> **Sugerencia**: si ya dispones de un clúster con una versión de runtime 13.3 LTS o superior en tu área de trabajo de Azure Databricks, puedes utilizarlo para completar este ejercicio y omitir este procedimiento.

1. En Azure Portal, ve al grupo de recursos **msl-*xxxxxxx*** que se creó con el script (o al grupo de recursos que contiene el área de trabajo de Azure Databricks existente)

1. Selecciona el recurso Azure Databricks Service (llamado **databricks-*xxxxxxx*** si usaste el script de instalación para crearlo).

1. En la página **Información general** del área de trabajo, usa el botón **Inicio del área de trabajo** para abrir el área de trabajo de Azure Databricks en una nueva pestaña del explorador; inicia sesión si se solicita.

    > **Sugerencia**: al usar el portal del área de trabajo de Databricks, se pueden mostrar varias sugerencias y notificaciones. Descarta estos elementos y sigue las instrucciones proporcionadas para completar las tareas de este ejercicio.

1. En la barra lateral de la izquierda, selecciona la tarea **(+) Nuevo** y luego selecciona **Clúster** (es posible que debas buscar en el submenú **Más**).

1. En la página **Nuevo clúster**, crea un clúster con la siguiente configuración:
    - **Nombre del clúster**: clúster del *Nombre de usuario*  (el nombre del clúster predeterminado)
    - **Directiva**: Unrestricted (Sin restricciones)
    - **Modo de clúster** de un solo nodo
    - **Modo de acceso**: usuario único (*con la cuenta de usuario seleccionada*)
    - **Versión de runtime de Databricks**: 13.3 LTS (Spark 3.4.1, Scala 2.12) o posterior
    - **Usar aceleración de Photon**: seleccionado
    - **Tipo de nodo**: Standard_D4ds_v5
    - **Finaliza después de** *20* **minutos de inactividad**

1. Espera a que se cree el clúster. Esto puede tardar un par de minutos.

    > **Nota**: si el clúster no se inicia, es posible que la suscripción no tenga cuota suficiente en la región donde se aprovisiona el área de trabajo de Azure Databricks. Para obtener más información, consulta [El límite de núcleos de la CPU impide la creación de clústeres](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si esto sucede, puedes intentar eliminar el área de trabajo y crear una nueva en otra región. Puedes especificar una región como parámetro para el script de configuración de la siguiente manera: `./mslearn-databricks/setup.ps1 eastus`

## Creación de un cuaderno e ingesta de datos

1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **Cuaderno** y cambia el nombre predeterminado del cuaderno (**Cuaderno sin título *[fecha]***) por **Optimize Data Ingestion**. A continuación, en la lista desplegable **Conectar**, selecciona el clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

2. En la primera celda del cuaderno, escribe el siguiente código, que utiliza comandos del *shell* para descargar los archivos de datos de GitHub en el sistema de archivos utilizado por el clúster.

     ```python
    %sh
    rm -r /dbfs/nyc_taxi_trips
    mkdir /dbfs/nyc_taxi_trips
    wget -O /dbfs/nyc_taxi_trips/yellow_tripdata_2021-01.parquet https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/yellow_tripdata_2021-01.parquet
     ```

3. En la salida de la primera celda, usa el icono **+ Código** para agregar una nueva celda y ejecuta el código siguiente en ella para cargar el conjunto de datos en un DataFrame:
   
     ```python
    # Load the dataset into a DataFrame
    df = spark.read.parquet("/nyc_taxi_trips/yellow_tripdata_2021-01.parquet")
    display(df)
     ```

4. Usa la opción del menú **&#9656; Ejecutar celda** situado a la izquierda de la celda para ejecutarla. A continuación, espera a que se complete el trabajo de Spark ejecutado por el código.

## Optimización de la ingesta de datos con Auto Loader:

La optimización de la ingesta de datos es fundamental para controlar grandes conjuntos de datos de forma eficaz. Auto Loader está diseñado para procesar nuevos archivos de datos a medida que llegan al almacenamiento en la nube y es compatible con varios formatos de archivo y servicios de almacenamiento en la nube. 

El cargador automático proporciona un origen de streaming estructurado denominado `cloudFiles`. Dada una ruta de acceso del directorio de entrada en el almacenamiento de archivos en la nube, el origen `cloudFiles` procesa automáticamente los nuevos archivos a medida que llegan, con la opción de procesar también los archivos existentes en ese directorio. 

1. En una nueva celda, ejecuta el siguiente código para crear una secuencia basada en la carpeta que contiene los datos de ejemplo:

     ```python
     df = (spark.readStream
             .format("cloudFiles")
             .option("cloudFiles.format", "parquet")
             .option("cloudFiles.schemaLocation", "/stream_data/nyc_taxi_trips/schema")
             .load("/nyc_taxi_trips/"))
     df.writeStream.format("delta") \
         .option("checkpointLocation", "/stream_data/nyc_taxi_trips/checkpoints") \
         .option("mergeSchema", "true") \
         .start("/delta/nyc_taxi_trips")
     display(df)
     ```

2. En una nueva celda, ejecuta el código siguiente para agregar un nuevo archivo Parquet a la secuencia:

     ```python
    %sh
    rm -r /dbfs/nyc_taxi_trips
    mkdir /dbfs/nyc_taxi_trips
    wget -O /dbfs/nyc_taxi_trips/yellow_tripdata_2021-02_edited.parquet https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/yellow_tripdata_2021-02_edited.parquet
     ```
   
    El nuevo archivo tiene una nueva columna, por lo que la secuencia se detiene con un error `UnknownFieldException`. Antes de que la secuencia genere este error, Auto Loader realiza la inferencia de esquemas en el microlote de datos más reciente y actualiza la ubicación del esquema con el esquema más reciente combinando las columnas nuevas al final del esquema. Los tipos de datos de las columnas existentes permanecen sin cambios.

3. Vuelve a ejecutar la celda de código de streaming y comprueba que se agregaron dos columnas nuevas (**new_column** y *_rescued_data**) a la tabla. La columna **_rescued_data** contiene los datos que no se analizan debido al error de coincidencia de tipos, a la falta de coincidencia entre mayúsculas y minúsculas, o a la ausencia de una columna en el esquema.

4. Selecciona **Interrumpir** para detener el streaming de datos.
   
    Los datos de streaming se escriben en tablas Delta. Delta Lake proporciona un conjunto de mejoras para los archivos Parquet tradicionales, incluidas las transacciones ACID, la evolución de esquemas y el viaje en el tiempo, y unifica el procesamiento de datos por lotes y streaming, por lo que resulta una solución eficaz para administrar cargas de trabajo de macrodatos.

## Optimización de la transformación de datos

La asimetría de datos es un desafío importante en la informática distribuida, especialmente en el procesamiento de macrodatos con marcos como Apache Spark. El cifrado con sal es una técnica eficaz para optimizar la asimetría de datos mediante la adición de un componente aleatorio, o “sal”, a las claves antes de la creación de particiones. Este proceso ayuda a distribuir datos de forma más uniforme entre particiones, lo que conduce a una carga de trabajo más equilibrada y a un rendimiento mejorado.

1. En una nueva celda, ejecuta el código siguiente para dividir una partición sesgada grande en particiones más pequeñas anexando una columna de *cifrado con sal* con enteros aleatorios:

     ```python
    from pyspark.sql.functions import lit, rand

    # Convert streaming DataFrame back to batch DataFrame
    df = spark.read.parquet("/nyc_taxi_trips/*.parquet")
     
    # Add a salt column
    df_salted = df.withColumn("salt", (rand() * 100).cast("int"))

    # Repartition based on the salted column
    df_salted.repartition("salt").write.format("delta").mode("overwrite").save("/delta/nyc_taxi_trips_salted")

    display(df_salted)
     ```   

## Optimización del almacenamiento

Delta Lake ofrece un conjunto de comandos de optimización que pueden mejorar significativamente el rendimiento y la administración del almacenamiento de datos. El comando `optimize` está diseñado para mejorar la velocidad de las consultas mediante la organización de datos de forma más eficaz a través de técnicas como la compactación y el orden Z.

La compactación consolida archivos más pequeños en archivos más grandes, lo que puede ser especialmente beneficioso para las consultas de lectura. El orden Z implica organizar puntos de datos para que la información relacionada se almacene cerca, lo que reduce el tiempo necesario para acceder a estos datos durante las consultas.

1. En una nueva celda, ejecuta el código siguiente para realizar la compactación en la tabla Delta:

     ```python
    from delta.tables import DeltaTable

    delta_table = DeltaTable.forPath(spark, "/delta/nyc_taxi_trips")
    delta_table.optimize().executeCompaction()
     ```

2. En una nueva celda, ejecuta el código siguiente para realizar la agrupación en clústeres de orden Z:

     ```python
    delta_table.optimize().executeZOrderBy("tpep_pickup_datetime")
     ```

Esta técnica buscará la información relacionada en el mismo conjunto de archivos, lo que mejorará el rendimiento de las consultas.

## Limpieza

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si has terminado de explorar Azure Databricks, puedes eliminar los recursos que has creado para evitar costes innecesarios de Azure y liberar capacidad en tu suscripción.
