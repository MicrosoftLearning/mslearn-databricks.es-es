---
lab:
  title: Uso de Apache Spark en Azure Databricks
---

# Uso de Apache Spark en Azure Databricks

Azure Databricks es una versión basada en Microsoft Azure de la conocida plataforma de código abierto Databricks. Azure Databricks se basa en Apache Spark y ofrece una solución altamente escalable para tareas de ingeniería y análisis de datos que implican trabajar con datos en archivos. Una de las ventajas de Spark es la compatibilidad con una amplia variedad de lenguajes de programación, como Java, Scala, Python y SQL, lo que lo convierte en una solución muy flexible para cargas de trabajo de procesamiento de datos, incluida la limpieza y manipulación de datos, el análisis estadístico y el aprendizaje automático, y el análisis y la visualización de datos.

Este ejercicio debería tardar en completarse **45** minutos aproximadamente.

## Aprovisiona un área de trabajo de Azure Databricks.

> **Sugerencia**: Si ya tiene un área de trabajo de Azure Databricks, puede omitir este procedimiento y usar el área de trabajo existente.

En este ejercicio, se incluye un script para aprovisionar una nueva área de trabajo de Azure Databricks. El script intenta crear un recurso de área de trabajo de Azure Databricks de nivel *Premium* en una región en la que la suscripción de Azure tiene cuota suficiente para los núcleos de proceso necesarios en este ejercicio, y da por hecho que la cuenta de usuario tiene permisos suficientes en la suscripción para crear un recurso de área de trabajo de Azure Databricks. Si se produjese un error en el script debido a cuota o permisos insuficientes, intente [crear un área de trabajo de Azure Databricks de forma interactiva en Azure Portal](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. En un explorador web, inicia sesión en [Azure Portal](https://portal.azure.com) en `https://portal.azure.com`.
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
7. Espera a que se complete el script: normalmente tarda unos 5 minutos, pero en algunos casos puede tardar más. Mientras esperas, revisa el artículo [Análisis de datos exploratorios en Azure Databricks](https://learn.microsoft.com/azure/databricks/exploratory-data-analysis/) en la documentación de Azure Databricks.

## Crear un clúster

Azure Databricks es una plataforma de procesamiento distribuido que usa clústeres* de Apache Spark *para procesar datos en paralelo en varios nodos. Cada clúster consta de un nodo de controlador para coordinar el trabajo y nodos de trabajo para hacer tareas de procesamiento. En este ejercicio, crearás un clúster de *nodo único* para minimizar los recursos de proceso usados en el entorno de laboratorio (en los que se pueden restringir los recursos). En un entorno de producción, normalmente crearías un clúster con varios nodos de trabajo.

> **Sugerencia**: Si ya dispone de un clúster con una versión de runtime 13.3 LTS o superior en su área de trabajo de Azure Databricks, puede utilizarlo para completar este ejercicio y omitir este procedimiento.

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
    - **Versión de runtime de Databricks**: 13.3 LTS (Spark 3.4.1, Scala 2.12) o posterior
    - **Usar aceleración de Photon**: seleccionado
    - **Tipo de nodo**: Standard_DS3_v2.
    - **Finaliza después de** *20* **minutos de inactividad**

1. Espera a que se cree el clúster. Esto puede tardar un par de minutos.

> **Nota**: si el clúster no se inicia, es posible que la suscripción no tenga cuota suficiente en la región donde se aprovisiona el área de trabajo de Azure Databricks. Para más información consulta [El límite de núcleos de la CPU impide la creación de clústeres](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si esto sucede, puedes intentar eliminar el área de trabajo y crear una nueva en otra región. Puedes especificar una región como parámetro para el script de configuración de la siguiente manera: `./mslearn-databricks/setup.ps1 eastus`

## Exploración de datos con Spark

Como en muchos entornos de Spark, Databricks es compatible con el uso de cuadernos para combinar notas y celdas de código interactivo que puedes usar para explorar los datos.

### Crear un cuaderno

1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**.
1. Cambie el nombre predeterminado del cuaderno (**Cuaderno sin título *[fecha]***) a **Explorar datos con Spark** y en la lista desplegable**Conectar**, seleccione el clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

### Ingerir datos

1. En la primera celda del cuaderno, escriba el siguiente código, que utiliza comandos del *shell* para descargar los archivos de datos de GitHub en el sistema de archivos utilizado por el clúster.

    ```python
    %sh
    rm -r /dbfs/spark_lab
    mkdir /dbfs/spark_lab
    wget -O /dbfs/spark_lab/2019.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2019.csv
    wget -O /dbfs/spark_lab/2020.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2020.csv
    wget -O /dbfs/spark_lab/2021.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2021.csv
    ```

1. Use la opción del menú **&#9656; Ejecutar celda** situado a la izquierda de la celda para ejecutarla. A continuación, espere a que se complete el trabajo de Spark ejecutado por el código.

### Consulta de datos en archivos

1. Debajo de la celda de código existente, usa el icono **+** para agregar una nueva celda de código. A continuación, en la nueva celda, escriba y ejecute el siguiente código para cargar los datos de los archivos y ver las primeras 100 filas.

    ```python
   df = spark.read.load('spark_lab/*.csv', format='csv')
   display(df.limit(100))
    ```

1. Revise la salida y observe que los datos del archivo se relacionan con pedidos de ventas, pero no incluye los encabezados de columna ni la información sobre los tipos de datos. Para que los datos tengan más sentido, puede definir un *esquema* para el DataFrame.

1. Agregue una nueva celda de código y úsela para ejecutar el código siguiente, que define un esquema para los datos:

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

1. Observe que esta vez, el DataFrame incluye encabezados de columna. A continuación, agregue una nueva celda de código y úsela para ejecutar el código siguiente a fin de mostrar los detalles del esquema del DataFrame y comprobar que se han aplicado los tipos de datos correctos:

    ```python
   df.printSchema()
    ```

### Filtrado de un objeto DataFrame

1. Agregue una nueva celda de código y úsela para ejecutar el código siguiente para las tareas que se indican a continuación:
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
    - La sintaxis `dataframe['Field1', 'Field2', ...]` es una forma abreviada de definir un subconjunto de columna. También puede usar el método **select**, por lo que la primera línea del código anterior se podría escribir como `customers = df.select("CustomerName", "Email")`.

1. Ahora vamos a aplicar un filtro para incluir solo los clientes que han realizado un pedido para un producto específico ejecutando el código siguiente en una nueva celda de código:

    ```python
   customers = df.select("CustomerName", "Email").where(df['Item']=='Road-250 Red, 52')
   print(customers.count())
   print(customers.distinct().count())
   display(customers.distinct())
    ```

    Tenga en cuenta que puede "concatenar" varias funciones para que la salida de una función se convierta en la entrada de la siguiente; en este caso, el DataFrame creado por el método “select” es el DataFrame de origen para el método “where” que se usa para aplicar criterios de filtrado.

### Agregación y agrupación de datos en un objeto DataFrame

1. Ejecute el código siguiente en una nueva celda de código para agregar y agrupar los datos de pedidos:

    ```python
   productSales = df.select("Item", "Quantity").groupBy("Item").sum()
   display(productSales)
    ```

    Observe que los resultados muestran la suma de las cantidades de pedidos agrupadas por producto. El método **groupBy** agrupa las filas por *Item* y la función de agregado **sum** subsiguiente se aplica a todas las columnas numéricas restantes (en este caso, *Quantity*).

1. En una nueva celda de código, vamos a probar otra agregación:

    ```python
   yearlySales = df.select(year("OrderDate").alias("Year")).groupBy("Year").count().orderBy("Year")
   display(yearlySales)
    ```

    Esta vez, los resultados muestran el número de pedidos de ventas por año. Tenga en cuenta que el método “select” incluye una función **year** de SQL para extraer el componente del año del campo *OrderDate* y, a continuación, se usa un método **alias** para asignar un nombre de columna al valor del año extraído. A continuación, los datos se agrupan por la columna *Year* derivada, y el **recuento** de filas en cada grupo se calcula antes de que finalmente el método **orderBy** se usa para ordenar el DataFrame resultante.

> **Nota**: Para obtener más información sobre cómo trabajar con DataFrames en Azure Databricks, consulte [Introducción a DataFrames: Python](https://docs.microsoft.com/azure/databricks/spark/latest/dataframes-datasets/introduction-to-dataframes-python) en la documentación de Azure Databricks.

### Consulta de datos con Spark SQL

1. Agregue una nueva celda de código y úsela para ejecutar el código siguiente:

    ```python
   df.createOrReplaceTempView("salesorders")
   spark_df = spark.sql("SELECT * FROM salesorders")
   display(spark_df)
    ```

    Los métodos nativos del DataFrame que usó anteriormente le permiten consultar y analizar datos de forma bastante eficaz. Sin embargo, a muchos analistas de datos les gusta más trabajar con sintaxis SQL. Spark SQL es una API de lenguaje SQL en Spark que puedes usar para ejecutar instrucciones SQL o incluso conservar datos en tablas relacionales. El código que acaba de ejecutar crea una vista *relacional* de los datos de un DataFrame y, a continuación, usa la biblioteca **spark.sql** para insertar la sintaxis de Spark SQL en el código de Python, consultar la vista y devolver los resultados como un DataFrame.

### Ejecución de código SQL en una celda

1. Aunque resulta útil poder insertar instrucciones SQL en una celda que contenga código de PySpark, los analistas de datos suelen preferir trabajar directamente en SQL. Agregue una nueva celda de código y úsela para ejecutar el código siguiente.

    ```sql
   %sql
    
   SELECT YEAR(OrderDate) AS OrderYear,
          SUM((UnitPrice * Quantity) + Tax) AS GrossRevenue
   FROM salesorders
   GROUP BY YEAR(OrderDate)
   ORDER BY OrderYear;
    ```

    Observe lo siguiente:
    
    - La línea ``%sql` al principio de la celda (llamada un comando magic) indica que se debe usar el runtime del lenguaje Spark SQL para ejecutar el código en esta celda en lugar de PySpark.
    - El código SQL hace referencia a la vista de **salesorder** que creó anteriormente.
    - La salida de la consulta SQL se muestra automáticamente como resultado en la celda.
    
> **Nota**: Para más información sobre Spark SQL y los objetos DataFrame, consulte la [documentación de Spark SQL](https://spark.apache.org/docs/2.2.0/sql-programming-guide.html).

## Visualización de datos con Spark

Proverbialmente, una imagen vale más que mil palabras, y un gráfico suele ser mejor que mil filas de datos. Aunque los cuadernos de Azure Databricks admiten la visualización de datos desde un DataFrame o una consulta de Spark SQL, no están diseñados para generar gráficos completos. Sin embargo, puede usar bibliotecas de gráficos de Python como matplotlib y seaborn para crear gráficos a partir de datos de objetos DataFrame.

### Ver los resultados como una visualización

1. En una nueva celda de código, ejecute el código siguiente para consultar la tabla **salesorders**:

    ```sql
   %sql
    
   SELECT * FROM salesorders
    ```

1. Encima de la tabla de resultados, selecciona **+** y luego **Visualización** para ver el editor de visualización y luego aplica las siguientes opciones:
    - **Tipo de visualización**: barra
    - **Columna X**: Elemento
    - **Columna Y**: *Agregue una nueva columna y seleccione* **Cantidad**. *Aplique la agregación* **Suma****.
    
1. Guarde la visualización y vuelva a ejecutar la celda de código para ver el gráfico resultante en el cuaderno.

### Introducción a matplotlib

1. En una nueva celda de código, ejecute el código siguiente para recuperar algunos datos de pedidos de ventas en un DataFrame:

    ```python
   sqlQuery = "SELECT CAST(YEAR(OrderDate) AS CHAR(4)) AS OrderYear, \
                   SUM((UnitPrice * Quantity) + Tax) AS GrossRevenue \
            FROM salesorders \
            GROUP BY CAST(YEAR(OrderDate) AS CHAR(4)) \
            ORDER BY OrderYear"
   df_spark = spark.sql(sqlQuery)
   df_spark.show()
    ```

1. Agregue una nueva celda de código y úsela para ejecutar el código siguiente, que importa la biblioteca **matplotlb** y la usa para crear un gráfico:

    ```python
   from matplotlib import pyplot as plt
    
   # matplotlib requires a Pandas dataframe, not a Spark one
   df_sales = df_spark.toPandas()
   # Create a bar plot of revenue by year
   plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'])
   # Display the plot
   plt.show()
    ```

1. Revise los resultados, que incluyen un gráfico de columnas con los ingresos brutos totales de cada año. Observe las siguientes características del código usado para generar este gráfico:
    - La biblioteca **matplotlib** requiere un DataFrame de Pandas, por lo que debe convertir el DataFrame de Spark devuelto por la consulta de Spark SQL a este formato.
    - En el centro de la biblioteca **matplotlib** se encuentra el objeto **pyplot**. Esta es la base de la mayor parte de la funcionalidad de trazado.

1. La configuración predeterminada da como resultado un gráfico utilizable, pero hay un margen considerable para personalizarla. Agregue una nueva celda de código con el código siguiente y ejecútelo:

    ```python
   # Clear the plot area
   plt.clf()
   # Create a bar plot of revenue by year
   plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')
   # Customize the chart
   plt.title('Revenue by Year')
   plt.xlabel('Year')
   plt.ylabel('Revenue')
   plt.grid(color='#95a5a6', linestyle='--', linewidth=2, axis='y', alpha=0.7)
   plt.xticks(rotation=45)
   # Show the figure
   plt.show()
    ```

1. Un gráfico está técnicamente contenido con una **Figura**. En los ejemplos anteriores, la figura se creó implícitamente; pero puede crearla explícitamente. Intente ejecutar lo siguiente en una nueva celda:

    ```python
   # Clear the plot area
   plt.clf()
   # Create a Figure
   fig = plt.figure(figsize=(8,3))
   # Create a bar plot of revenue by year
   plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')
   # Customize the chart
   plt.title('Revenue by Year')
   plt.xlabel('Year')
   plt.ylabel('Revenue')
   plt.grid(color='#95a5a6', linestyle='--', linewidth=2, axis='y', alpha=0.7)
   plt.xticks(rotation=45)
   # Show the figure
   plt.show()
    ```

1. Una figura puede contener varios subtrazados, cada uno en su propio eje. Use este código para crear varios gráficos:

    ```python
   # Clear the plot area
   plt.clf()
   # Create a figure for 2 subplots (1 row, 2 columns)
   fig, ax = plt.subplots(1, 2, figsize = (10,4))
   # Create a bar plot of revenue by year on the first axis
   ax[0].bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')
   ax[0].set_title('Revenue by Year')
   # Create a pie chart of yearly order counts on the second axis
   yearly_counts = df_sales['OrderYear'].value_counts()
   ax[1].pie(yearly_counts)
   ax[1].set_title('Orders per Year')
   ax[1].legend(yearly_counts.keys().tolist())
   # Add a title to the Figure
   fig.suptitle('Sales Data')
   # Show the figure
   plt.show()
    ```

> **Nota**: Para más información sobre el trazado con matplotlib, consulte la [documentación de matplotlib](https://matplotlib.org/).

### Uso de la biblioteca seaborn

1. Agregue una nueva celda de código y úsela para ejecutar el código siguiente, que usa la biblioteca **seaborn** (que se basa en matplotlib y abstrae parte de su complejidad) para crear un gráfico:

    ```python
   import seaborn as sns
   
   # Clear the plot area
   plt.clf()
   # Create a bar chart
   ax = sns.barplot(x="OrderYear", y="GrossRevenue", data=df_sales)
   plt.show()
    ```

1. La biblioteca **seaborn** facilita la creación de trazados complejos de datos estadísticos y le permite controlar el tema visual para ver visualizaciones de datos coherentes. Ejecute el código siguiente en una nueva celda:

    ```python
   # Clear the plot area
   plt.clf()
   
   # Set the visual theme for seaborn
   sns.set_theme(style="whitegrid")
   
   # Create a bar chart
   ax = sns.barplot(x="OrderYear", y="GrossRevenue", data=df_sales)
   plt.show()
    ```

1. A igual que matplotlib, seaborn admite varios tipos de gráficos. Ejecute el código siguiente para crear un gráfico de líneas:

    ```python
   # Clear the plot area
   plt.clf()
   
   # Create a bar chart
   ax = sns.lineplot(x="OrderYear", y="GrossRevenue", data=df_sales)
   plt.show()
    ```

> **Nota**: Para más información sobre el trazado con seaborn, consulte la [documentación de seaborn](https://seaborn.pydata.org/index.html).

## Limpiar

En el portal de Azure Databricks, en la página **Proceso**, seleccione el clúster y **&#9632; Finalizar** para apagarlo.

Si ha terminado de explorar Azure Databricks, puede eliminar los recursos que ha creado para evitar costos innecesarios de Azure y liberar capacidad en su suscripción.