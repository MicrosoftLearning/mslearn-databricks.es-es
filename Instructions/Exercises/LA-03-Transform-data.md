---
lab:
  title: Transformación de datos de Azure Databricks con Apache Spark
---

# Transformación de datos de Azure Databricks con Apache Spark

Azure Databricks es una versión basada en Microsoft Azure de la conocida plataforma de código abierto Databricks. 

Azure Databricks se basa en Apache Spark y ofrece una solución altamente escalable para tareas de ingeniería y análisis de datos que implican trabajar con datos en archivos.

Las tareas comunes de transformación de datos en Azure Databricks incluyen limpieza de datos, realización de agregaciones y conversión de tipos. Estas transformaciones son esenciales para preparar los datos para el análisis y forman parte del proceso más amplio de ETL (extracción, transformación, carga).

Este ejercicio debería tardar en completarse **30** minutos aproximadamente.

## Aprovisiona un área de trabajo de Azure Databricks.

> **Sugerencia**: si ya tienes un área de trabajo de Azure Databricks, puedes omitir este procedimiento y usar el área de trabajo existente.

En este ejercicio, se incluye un script para aprovisionar una nueva área de trabajo de Azure Databricks. El script intenta crear un recurso de área de trabajo de Azure Databricks de nivel *Premium* en una región en la que la suscripción de Azure tiene cuota suficiente para los núcleos de proceso necesarios en este ejercicio, y da por hecho que la cuenta de usuario tiene permisos suficientes en la suscripción para crear un recurso de área de trabajo de Azure Databricks. Si se produjese un error en el script debido a cuota o permisos insuficientes, intenta [crear un área de trabajo de Azure Databricks de forma interactiva en Azure Portal](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. En un explorador web, inicia sesión en [Azure Portal](https://portal.azure.com) en `https://portal.azure.com`.
2. Usa el botón **[\>_]** a la derecha de la barra de búsqueda en la parte superior de la página para crear un nuevo Cloud Shell en Azure Portal, selecciona un entorno de ***PowerShell*** y crea almacenamiento si se te solicita. Cloud Shell proporciona una interfaz de línea de comandos en un panel situado en la parte inferior de Azure Portal, como se muestra a continuación:

    ![Azure Portal con un panel de Cloud Shell](./images/cloud-shell.png)

    > **Nota**: Si ha creado previamente un cloud shell que usa un entorno de *Bash*, use el menú desplegable de la parte superior izquierda del panel de cloud shell para cambiarlo a ***PowerShell***.

3. Tenga en cuenta que puede cambiar el tamaño de Cloud Shell arrastrando la barra de separación en la parte superior del panel, o usando los iconos **&#8212;** , **&#9723;** y **X** en la parte superior derecha para minimizar, maximizar y cerrar el panel. Para obtener más información sobre el uso de Azure Cloud Shell, consulta la [documentación de Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

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
7. Espera a que se complete el script: normalmente tarda unos 5 minutos, pero en algunos casos puede tardar más. Mientras esperas, revisa el artículo [Análisis de datos exploratorios en Azure Databricks](https://learn.microsoft.com/azure/databricks/exploratory-data-analysis/) en la documentación de Azure Databricks.

## Crear un clúster

Azure Databricks es una plataforma de procesamiento distribuido que usa clústeres* de Apache Spark *para procesar datos en paralelo en varios nodos. Cada clúster consta de un nodo de controlador para coordinar el trabajo y nodos de trabajo para hacer tareas de procesamiento. En este ejercicio, crearás un clúster de *nodo único* para minimizar los recursos de proceso usados en el entorno de laboratorio (en los que se pueden restringir los recursos). En un entorno de producción, normalmente crearías un clúster con varios nodos de trabajo.

> **Sugerencia**: si ya dispones de un clúster con una versión de runtime 13.3 LTS o superior en tu área de trabajo de Azure Databricks, puedes utilizarlo para completar este ejercicio y omitir este procedimiento.

1. En Azure Portal, ve al grupo de recursos **msl-*xxxxxxx*** que se creó con el script (o al grupo de recursos que contiene el área de trabajo de Azure Databricks existente)
2. Selecciona el recurso Azure Databricks Service (llamado **databricks-*xxxxxxx*** si usaste el script de instalación para crearlo).
3. En la página **Información general** del área de trabajo, usa el botón **Inicio del área de trabajo** para abrir el área de trabajo de Azure Databricks en una nueva pestaña del explorador; inicia sesión si se solicita.

    > **Sugerencia**: al usar el portal del área de trabajo de Databricks, se pueden mostrar varias sugerencias y notificaciones. Descártalas y sigue las instrucciones proporcionadas para completar las tareas de este ejercicio.

4. En la barra lateral de la izquierda, selecciona la tarea **(+) Nuevo** y luego selecciona **Clúster**.
5. En la página **Nuevo clúster**, crea un clúster con la siguiente configuración:
    - **Nombre del clúster**: clúster del *Nombre de usuario*  (el nombre del clúster predeterminado)
    - **Directiva**: Unrestricted (Sin restricciones)
    - **Modo de clúster** de un solo nodo
    - **Modo de acceso**: usuario único (*con la cuenta de usuario seleccionada*)
    - **Versión de runtime de Databricks**: 13.3 LTS (Spark 3.4.1, Scala 2.12) o posterior
    - **Usar aceleración de Photon**: seleccionado
    - **Tipo de nodo**: Standard_D4ds_v5
    - **Finaliza después de** *20* **minutos de inactividad**

6. Espera a que se cree el clúster. Esto puede tardar un par de minutos.

> **Nota**: si el clúster no se inicia, es posible que la suscripción no tenga cuota suficiente en la región donde se aprovisiona el área de trabajo de Azure Databricks. Para obtener más información, consulta [El límite de núcleos de la CPU impide la creación de clústeres](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si esto sucede, puedes intentar eliminar el área de trabajo y crear una nueva en otra región. Puedes especificar una región como parámetro para el script de configuración de la siguiente manera: `./mslearn-databricks/setup.ps1 eastus`

## Crear un cuaderno

1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**.

2. Cambia el nombre por defecto del cuaderno (**Cuaderno sin título *[fecha]***) por `Transform data with Spark` y en la lista desplegable **Conectar**, selecciona tu clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

## Ingerir datos

1. En la primera celda del cuaderno, escribe el siguiente código, que utiliza comandos del *shell* para descargar los archivos de datos de GitHub en el sistema de archivos utilizado por el clúster.

     ```python
    %sh
    rm -r /dbfs/spark_lab
    mkdir /dbfs/spark_lab
    wget -O /dbfs/spark_lab/2019.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2019_edited.csv
    wget -O /dbfs/spark_lab/2020.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2020_edited.csv
    wget -O /dbfs/spark_lab/2021.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2021_edited.csv
     ```

2. Usa la opción del menú **&#9656; Ejecutar celda** situado a la izquierda de la celda para ejecutarla. A continuación, espera a que se complete el trabajo de Spark ejecutado por el código.
3. Agrega una nueva celda de código y úsala para ejecutar el código siguiente, que define un esquema para los datos:

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
   df = spark.read.load('/spark_lab/*.csv', format='csv', schema=orderSchema)
   display(df.limit(100))
    ```

## Limpieza de los datos

Observa que este conjunto de datos tiene algunas filas y `null` valores duplicados en la columna **Tax**. Por lo tanto, se requiere un paso de limpieza antes de realizar cualquier procesamiento y análisis adicionales con los datos.

![Tabla con datos que se van a limpiar.](./images/data-cleaning.png)

1. Debajo de la celda de código existente, usa el icono **+** para agregar una nueva celda de código. A continuación, en la nueva celda, escribe y ejecuta el código siguiente para quitar filas duplicadas de la tabla y reemplazar las entradas `null` por los valores correctos:

    ```python
    from pyspark.sql.functions import col
    df = df.dropDuplicates()
    df = df.withColumn('Tax', col('UnitPrice') * 0.08)
    df = df.withColumn('Tax', col('Tax').cast("float"))
    ```

Observa que después de actualizar los valores de la columna **Tax**, su tipo de datos se establece en `float` de nuevo. Esto se debe a que su tipo de datos cambia a `double` después de realizar el cálculo. Dado que `double` tiene un uso de memoria mayor que `float`, es mejor para el rendimiento escribir la conversión de la columna a `float`.

## Filtrado de un objeto DataFrame

1. Agrega una nueva celda de código y úsala para ejecutar el código siguiente para las tareas que se indican a continuación:
    - Filtrar las columnas del DataFrame de pedidos de ventas para que incluyan solo el nombre del cliente y la dirección de correo electrónico.
    - Contar el número total de registros de pedidos
    - Contar el número de clientes distintos
    - Mostrar los clientes distintos

    ```python
   customers = df['CustomerName', 'Email']
   print(customers.count())
   print(customers.distinct().count())
   display(customers.distinct())
    ```

    Observe los siguientes detalles:

    - Cuando se realiza una operación en un objeto DataFrame, el resultado es un nuevo DataFrame (en este caso, se crea un nuevo DataFrame customers seleccionando un subconjunto específico de columnas del DataFrame df).
    - Los objetos DataFrame proporcionan funciones como count y distinct que se pueden usar para resumir y filtrar los datos que contienen.
    - La sintaxis `dataframe['Field1', 'Field2', ...]` es una forma abreviada de definir un subconjunto de columna. También puedes usar el método **select**, por lo que la primera línea del código anterior se podría escribir como `customers = df.select("CustomerName", "Email")`.

1. Ahora vamos a aplicar un filtro para incluir solo los clientes que han realizado un pedido para un producto específico ejecutando el código siguiente en una nueva celda de código:

    ```python
   customers = df.select("CustomerName", "Email").where(df['Item']=='Road-250 Red, 52')
   print(customers.count())
   print(customers.distinct().count())
   display(customers.distinct())
    ```

    Ten en cuenta que puedes "concatenar" varias funciones para que la salida de una función se convierta en la entrada de la siguiente; en este caso, el DataFrame creado por el método “select” es el DataFrame de origen para el método “where” que se usa para aplicar criterios de filtrado.

## Agregación y agrupación de datos en un objeto DataFrame

1. Ejecuta el código siguiente en una nueva celda de código para agregar y agrupar los datos de pedidos:

    ```python
   productSales = df.select("Item", "Quantity").groupBy("Item").sum()
   display(productSales)
    ```

    Observa que los resultados muestran la suma de las cantidades de pedidos agrupadas por producto. El método **groupBy** agrupa las filas por *Item* y la función de agregado **sum** subsiguiente se aplica a todas las columnas numéricas restantes (en este caso, *Quantity*).

1. En una nueva celda de código, vamos a probar otra agregación:

    ```python
   yearlySales = df.select(year("OrderDate").alias("Year")).groupBy("Year").count().orderBy("Year")
   display(yearlySales)
    ```

    Esta vez, los resultados muestran el número de pedidos de ventas por año. Ten en cuenta que el método “select” incluye una función **year** de SQL para extraer el componente del año del campo *OrderDate* y, a continuación, se usa un método **alias** para asignar un nombre de columna al valor del año extraído. A continuación, los datos se agrupan por la columna *Year* derivada, y el **recuento** de filas en cada grupo se calcula antes de que finalmente el método **orderBy** se usa para ordenar el DataFrame resultante.

> **Nota**: para obtener más información sobre cómo trabajar con DataFrames en Azure Databricks, consulta [Introducción a DataFrames: Python](https://docs.microsoft.com/azure/databricks/spark/latest/dataframes-datasets/introduction-to-dataframes-python) en la documentación de Azure Databricks.

## Ejecución de código SQL en una celda

1. Aunque resulta útil poder insertar instrucciones SQL en una celda que contenga código de PySpark, los analistas de datos suelen preferir trabajar directamente en SQL. Agrega una nueva celda de código y úsala para ejecutar el código siguiente.

    ```python
   df.createOrReplaceTempView("salesorders")
    ```

Esta línea de código creará una vista temporal que se puede usar directamente con instrucciones SQL.

2. En una celda nueva, escribe el código siguiente:
   
    ```python
   %sql
    
   SELECT YEAR(OrderDate) AS OrderYear,
          SUM((UnitPrice * Quantity) + Tax) AS GrossRevenue
   FROM salesorders
   GROUP BY YEAR(OrderDate)
   ORDER BY OrderYear;
    ```

    Observa lo siguiente:
    
    - La línea **%sql** al principio de la celda (denominada magic) indica que se debe usar el tiempo de ejecución del lenguaje Spark SQL para ejecutar el código en esta celda en lugar de PySpark.
    - El código SQL hace referencia a la vista de **salesorder** que creaste anteriormente.
    - La salida de la consulta SQL se muestra automáticamente como resultado en la celda.
    
> **Nota**: para más información sobre Spark SQL y los objetos DataFrame, consulta la [documentación de Spark SQL](https://spark.apache.org/docs/2.2.0/sql-programming-guide.html).

## Limpiar

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si has terminado de explorar Azure Databricks, puedes eliminar los recursos que has creado para evitar costes innecesarios de Azure y liberar capacidad en tu suscripción.
