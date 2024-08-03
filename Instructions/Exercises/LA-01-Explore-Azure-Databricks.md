---
lab:
  title: Explorar Azure Databricks
---

# Explorar Azure Databricks

Azure Databricks es una versión basada en Microsoft Azure de la conocida plataforma de código abierto Databricks.

De forma similar a Azure Synapse Analytics, un *área de trabajo* de Azure Databricks proporciona un punto central para administrar clústeres, datos y recursos de Databricks en Azure.

Este ejercicio debería tardar aproximadamente **20** minutos en completarse.

## Aprovisiona un área de trabajo de Azure Databricks.

> **Sugerencia**: si ya tienes un área de trabajo de Azure Databricks, puedes omitir este procedimiento y usar el área de trabajo existente.

En este ejercicio, se incluye un script para aprovisionar una nueva área de trabajo de Azure Databricks. El script intenta crear un recurso de área de trabajo de Azure Databricks de nivel *Premium* en una región en la que la suscripción de Azure tiene cuota suficiente para los núcleos de proceso necesarios en este ejercicio, y da por hecho que la cuenta de usuario tiene permisos suficientes en la suscripción para crear un recurso de área de trabajo de Azure Databricks. Si se produjese un error en el script debido a cuota o permisos insuficientes, intente [crear un área de trabajo de Azure Databricks de forma interactiva en Azure Portal](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. En un explorador web, inicia sesión en [Azure Portal](https://portal.azure.com) en `https://portal.azure.com`.
2. Usa el botón **[\>_]** a la derecha de la barra de búsqueda en la parte superior de la página para crear un nuevo Cloud Shell en Azure Portal, selecciona un entorno de ***PowerShell*** y crea almacenamiento si se te solicita. Cloud Shell proporciona una interfaz de línea de comandos en un panel situado en la parte inferior de Azure Portal, como se muestra a continuación:

    ![Azure Portal con un panel de Cloud Shell](./images/cloud-shell.png)

    > **Nota**: Si creaste anteriormente un Cloud Shell que usa un entorno de *Bash*, usa el menú desplegable situado en la parte superior izquierda del panel de Cloud Shell para cambiarlo a ***PowerShell***.

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
7. Espera a que se complete el script: normalmente tarda unos 5 minutos, pero en algunos casos puede tardar más. Mientras esperas, consulta el artículo [Qué es Azure Databricks](https://learn.microsoft.com/azure/databricks/introduction/) en la documentación de Azure Databricks.

## Crear un clúster

Azure Databricks es una plataforma de procesamiento distribuido que usa clústeres* de Apache Spark *para procesar datos en paralelo en varios nodos. Cada clúster consta de un nodo de controlador para coordinar el trabajo y nodos de trabajo para hacer tareas de procesamiento. En este ejercicio, crearás un clúster de *nodo único* para minimizar los recursos de proceso usados en el entorno de laboratorio (en los que se pueden restringir los recursos). En un entorno de producción, normalmente crearías un clúster con varios nodos de trabajo.

> **Sugerencia**: si ya dispones de un clúster con una versión de runtime 13.3 LTS o superior en tu área de trabajo de Azure Databricks, puedes utilizarlo para completar este ejercicio y omitir este procedimiento.

1. En Azure Portal, ve al grupo de recursos **msl-*xxxxxxx*** que se creó con el script (o al grupo de recursos que contiene el área de trabajo de Azure Databricks existente)
1. Selecciona el recurso Azure Databricks Service (llamado **databricks-*xxxxxxx*** si usaste el script de instalación para crearlo).
1. En la página **Información general** del área de trabajo, usa el botón **Inicio del área de trabajo** para abrir el área de trabajo de Azure Databricks en una nueva pestaña del explorador; inicia sesión si se solicita.

    > **Sugerencia**: al usar el portal del área de trabajo de Databricks, se pueden mostrar varias sugerencias y notificaciones. Descártalas y sigue las instrucciones proporcionadas para completar las tareas de este ejercicio.

1. En la barra lateral de la izquierda, selecciona la tarea **(+) Nuevo** y luego selecciona **Clúster**.
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

## Uso de Spark para analizar datos

Como en muchos entornos de Spark, Databricks admite el uso de cuadernos para combinar notas y celdas de código interactivas que puedes usar para explorar datos.

1. Descargue el archivo [**products.csv**](https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv) de `https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv` en el equipo local y guárdelo como **products.csv**.
1. En la barra lateral, en el menú del vínculo **(+) Nuevo**, seleccione **Carga de archivos**.
1. Cargue el archivo **products.csv** que descargó en el equipo.
1. En la página **Crear o modificar tabla a partir de la carga de archivos**, asegúrese de que el clúster esté seleccionado en la parte superior derecha de la página. A continuación, elija el catálogo de **hive_metastore** y su esquema predeterminado para crear una nueva tabla denominada **productos**.
1. En la página **Explorador de catálogos**, cuando se haya creado la página **Productos**, en el menú del botón **Crear**, seleccione **Notebook** para crear un cuaderno.
1. En el cuaderno, asegúrese de que el cuaderno esté conectado al clúster y, a continuación, revise el código que se agregó automáticamente a la primera celda, el cual debería tener un aspecto similar al siguiente:

    ```python
    %sql
    SELECT * FROM `hive_metastore`.`default`.`products`;
    ```

1. Use la opción de menú **&#9656; Ejecutar celda** situada a la izquierda de la celda para ejecutarla, iniciando y adjuntando el clúster si se le solicitase.
1. Espera a que el código ejecute el trabajo de Spark. El código recupera datos de la tabla que se creó en función del archivo que cargó.
1. Encima de la tabla de resultados, selecciona **+** y luego **Visualización** para ver el editor de visualización y luego aplica las siguientes opciones:
    - **Tipo de visualización**: barra
    - **Columna X**: categoría
    - **Columna Y**: *agrega una nueva columna y selecciona ***ProductID**. *Aplica la **agregación* de la **cuenta**.

    Guarda la visualización y observa que aparece en el cuaderno, del siguiente modo:

    ![Un gráfico de barras con los recuentos de productos por categoría](./images/databricks-chart.png)

## Análisis de datos con un dataframe

Aunque la mayoría de los análisis de datos son correctos con el uso de código SQL, tal y como se usa en el ejemplo anterior, algunos analistas de datos y científicos de datos usan objetos nativos de Spark, como un *dataframe* en lenguajes de programación como *PySpark* (una versión optimizada para Spark de Python) para trabajar de forma eficaz con datos.

1. En el cuaderno, en la salida del gráfico de la celda de código ejecutada anteriormente, use el icono de **+** para agregar una nueva celda.
1. Escribe y ejecuta el siguiente código en la nueva celda:

    ```python
    df = spark.sql("SELECT * FROM products")
    df = df.filter("Category == 'Road Bikes'")
    display(df)
    ```

1. Ejecute la nueva celda, que devolverá los productos de la categoría *Bicicletas de carretera*.

## Limpiar

En el portal de Azure Databricks, en la página **Proceso**, seleccione el clúster y **&#9632; Finalizar** para apagarlo.

Si has terminado de explorar Azure Databricks, puedes eliminar los recursos que has creado para evitar costes innecesarios de Azure y liberar capacidad en tu suscripción.
