---
lab:
  title: Implementación de cargas de trabajo con flujos de trabajo de Azure Databricks
---

# Implementación de cargas de trabajo con flujos de trabajo de Azure Databricks

Los flujos de trabajo de Azure Databricks proporcionan una plataforma sólida para implementar cargas de trabajo de forma eficaz. Con características como Azure Databricks Jobs y Delta Live Tables, los usuarios pueden orquestar canalizaciones complejas de procesamiento de datos, aprendizaje automático y análisis.

Se tardan aproximadamente **40** minutos en completar este laboratorio.

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

7. Espera a que se complete el script: normalmente tarda unos 5 minutos, pero en algunos casos puede tardar más. Mientras espera, revisa el artículo [Programación y orquestación de flujos de trabajo](https://learn.microsoft.com/azure/databricks/jobs/) en la documentación de Azure Databricks.

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

1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**.

2. En la lista desplegable **Conectar**, selecciona el clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

3. En la primera celda del cuaderno, escribe el siguiente código, que utiliza comandos del *shell* para descargar los archivos de datos de GitHub en el sistema de archivos utilizado por el clúster.

     ```python
    %sh
    rm -r /dbfs/workflow_lab
    mkdir /dbfs/workflow_lab
    wget -O /dbfs/workflow_lab/2019.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/2019_edited.csv
    wget -O /dbfs/workflow_lab/2020.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/2020_edited.csv
    wget -O /dbfs/workflow_lab/2021.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/2021_edited.csv
     ```

4. Usa la opción del menú **&#9656; Ejecutar celda** situado a la izquierda de la celda para ejecutarla. A continuación, espera a que se complete el trabajo de Spark ejecutado por el código.

## Creación de una tarea de trabajo

Implementa el flujo de trabajo de procesamiento y análisis de datos mediante tareas. Un trabajo se compone de una o varias tareas. Puedes crear las tareas de los trabajos que ejecutan cuadernos, JARS, canalizaciones de Delta Live Tables o aplicaciones de Python, Scala, Spark submit y Java. En este ejercicio, crearás una tarea como cuaderno que extrae, transforma y carga datos en gráficos de visualización. 

1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**.

2. Cambia el nombre por defecto del cuaderno (**Cuaderno sin título *[fecha]***) por `ETL task` y en la lista desplegable **Conectar**, selecciona tu clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

    Asegúrate de que el idioma predeterminado para el cuaderno se ha establecido en **Python**.

3. En la primera celda del cuaderno, escribe y ejecuta el código siguiente, que define un esquema para los datos y carga los conjuntos de datos en un dataframe:

    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   orderSchema = StructType([
        StructField("SalesOrderNumber", StringType()),
        StructField("SalesOrderLineNumber", IntegerType()),
        StructField("OrderDate", DateType()),
        StructField("CustomerName", StringType()),
        StructField("Email", StringType()),
        StructField("Item", StringType()),
        StructField("Quantity", IntegerType()),
        StructField("UnitPrice", FloatType()),
        StructField("Tax", FloatType())
   ])
   df = spark.read.load('/workflow_lab/*.csv', format='csv', schema=orderSchema)
   display(df.limit(100))
    ```

4. Debajo de la celda de código existente, usa el icono **+ Código** para agregar una nueva celda de código. A continuación, en la nueva celda, escribe y ejecuta el código siguiente para quitar filas duplicadas y reemplazar las entradas `null` por los valores correctos:

     ```python
    from pyspark.sql.functions import col
    df = df.dropDuplicates()
    df = df.withColumn('Tax', col('UnitPrice') * 0.08)
    df = df.withColumn('Tax', col('Tax').cast("float"))
     ```
    > **Nota**: después de actualizar los valores de la columna **Tax**, su tipo de datos se establece en `float` de nuevo. Esto se debe a que su tipo de datos cambia a `double` después de realizar el cálculo. Dado que `double` tiene un uso de memoria mayor que `float`, es mejor para el rendimiento escribir la conversión de la columna a `float`.

5. En la nueva celda de código, ejecuta el código siguiente para agregar y agrupar los datos de pedidos:

    ```python
   yearlySales = df.select(year("OrderDate").alias("Year")).groupBy("Year").count().orderBy("Year")
   display(yearlySales)
    ```

## Compilación del flujo de trabajo

Azure Databricks administra la orquestación de tareas, la administración de clústeres, la supervisión y la generación de informes de errores en todos los trabajos. Puedes ejecutar los trabajos inmediatamente, periódicamente a través de un sistema de programación fácil de usar, siempre que los nuevos archivos lleguen a una ubicación externa o de forma continua para asegurarse de que una instancia del trabajo siempre se está ejecutando.

1. En la barra de menús de la izquierda, selecciona **Flujos de trabajo**.

2. En el panel Flujos de trabajo, selecciona **Crear trabajo**.

3. Cambia el nombre de trabajo predeterminado (**Nuevo trabajo *[fecha]***) a `ETL job`

4. Configura el trabajo con la siguiente configuración:
    - **Nombre de la tarea**: `Run ETL task notebook`
    - **Tipo**: Cuaderno
    - **Origen**: Área de trabajo
    - **Ruta**: *selecciona tu* *cuaderno* de tareas ETL
    - **Clúster**: *Seleccione el clúster*

5. Selecciona **Crear tarea**.

6. Selecciona **Ejecutar ahora**.

7. Después de que el trabajo empiece a ejecutarse, puedes supervisar su ejecución mediante la selección de **Ejecuciones de trabajo** en la barra lateral izquierda.

8. Una vez que la ejecución del trabajo se realice correctamente, puedes seleccionarla y comprobar su salida.

Además, puedes ejecutar trabajos de forma desencadenada, por ejemplo, ejecutando un flujo de trabajo según una programación. Para programar una ejecución de trabajo periódica, puedes abrir la tarea de trabajo y agregar un desencadenador.

## Limpieza

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si has terminado de explorar Azure Databricks, puedes eliminar los recursos que has creado para evitar costes innecesarios de Azure y liberar capacidad en la suscripción.
