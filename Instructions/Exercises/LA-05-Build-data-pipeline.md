---
lab:
  title: Creación de una canalización de datos con Delta Live Tables
---

# Creación de una canalización de datos con Delta Live Tables

Delta Live Tables es una plataforma para crear canalizaciones de procesamiento de datos confiables, fáciles de mantener y que se pueden probar. Una canalización es la unidad principal utilizada para configurar y ejecutar flujos de trabajo de procesamiento de datos con Delta Live Tables. Vincula los orígenes de datos a los conjuntos de datos de destino a través de un grafo Acíclico dirigido (DAG) declarado en Python o SQL.

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

7. Espera a que se complete el script: normalmente tarda unos 5 minutos, pero en algunos casos puede tardar más. Mientras esperas, revisa el artículo [Qué es Delta Live Tables](https://learn.microsoft.com/azure/databricks/delta-live-tables/) en la documentación de Azure Databricks.

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

2. Cambia el nombre por defecto del cuaderno (**Cuaderno sin título *[fecha]***) por `Create a pipeline with Delta Live tables` y en la lista desplegable **Conectar**, selecciona tu clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

3. En la primera celda del cuaderno, escribe el siguiente código, que utiliza comandos del *shell* para descargar los archivos de datos de GitHub en el sistema de archivos utilizado por el clúster.

     ```python
    %sh
    rm -r /dbfs/delta_lab
    mkdir /dbfs/delta_lab
    wget -O /dbfs/delta_lab/covid_data.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/covid_data.csv
     ```

4. Usa la opción del menú **&#9656; Ejecutar celda** situado a la izquierda de la celda para ejecutarla. A continuación, espera a que se complete el trabajo de Spark ejecutado por el código.

## Creación de una canalización de Delta Live Tables mediante SQL

1. Crea un nuevo cuaderno y cámbiale el nombre por `Pipeline Notebook`.

1. Junto al nombre del cuaderno, selecciona **Python** y cambia el lenguaje predeterminado a **SQL**.

1. Escribe el código siguiente en la primera celda sin ejecutarlo. Todas las celdas se ejecutarán después de crear la canalización. Este código define una tabla Delta Live Table que se rellenará con los datos sin procesar descargados anteriormente:

     ```sql
    CREATE OR REFRESH LIVE TABLE raw_covid_data
    COMMENT "COVID sample dataset. This data was ingested from the COVID-19 Data Repository by the Center for Systems Science and Engineering (CSSE) at Johns Hopkins University."
    AS
    SELECT
      Last_Update,
      Country_Region,
      Confirmed,
      Deaths,
      Recovered
    FROM read_files('dbfs:/delta_lab/covid_data.csv', format => 'csv', header => true)
     ```

1. En la primera celda, utiliza el icono **+ Código** para agregar una nueva celda y especifica el código siguiente para consultar, filtrar y dar formato a los datos de la tabla anterior antes del análisis.

     ```sql
    CREATE OR REFRESH LIVE TABLE processed_covid_data(
      CONSTRAINT valid_country_region EXPECT (Country_Region IS NOT NULL) ON VIOLATION FAIL UPDATE
    )
    COMMENT "Formatted and filtered data for analysis."
    AS
    SELECT
        TO_DATE(Last_Update, 'MM/dd/yyyy') as Report_Date,
        Country_Region,
        Confirmed,
        Deaths,
        Recovered
    FROM live.raw_covid_data;
     ```

1. En una tercera nueva celda de código, especifica el código siguiente que creará una vista de datos enriquecida para su posterior análisis una vez que la canalización se ejecute correctamente.

     ```sql
    CREATE OR REFRESH LIVE TABLE aggregated_covid_data
    COMMENT "Aggregated daily data for the US with total counts."
    AS
    SELECT
        Report_Date,
        sum(Confirmed) as Total_Confirmed,
        sum(Deaths) as Total_Deaths,
        sum(Recovered) as Total_Recovered
    FROM live.processed_covid_data
    GROUP BY Report_Date;
     ```
     
1. Selecciona **Delta Live Tables** en la barra lateral izquierda y, luego, selecciona **Crear canalización**.

1. En la página **Crear canalización**, crea una canalización con la siguiente configuración:
    - **Nombre de la canalización**: `Covid Pipeline`
    - **Edición del producto**: avanzado
    - **Modo de canalización**: desencadenado
    - **Código fuente**: *navega a tu cuaderno* Cuaderno de canalización *en la carpeta *Users/user@name**.
    - **Opciones de almacenamiento**: metastore de Hive
    - **Ubicación de almacenamiento**: `dbfs:/pipelines/delta_lab`
    - **Esquema de destino**: *especifica*`default`

1. Selecciona **Crear** y después **Guardar**. A continuación, espera a que se ejecute la canalización (que puede tardar algún tiempo).
 
1. Una vez que la canalización se haya ejecutado correctamente, vuelve al cuaderno *Crear una canalización con tablas Delta Live* reciente que creaste primero y ejecuta el código siguiente en una nueva celda para comprobar que los archivos de las 3 tablas nuevas se han creado en la ubicación de almacenamiento especificada:

     ```python
    display(dbutils.fs.ls("dbfs:/pipelines/delta_lab/tables"))
     ```

1. Agrega otra celda de código y ejecuta el código siguiente para comprobar que las tablas se han creado en la base de datos **predeterminada**:

     ```sql
    %sql

    SHOW TABLES
     ```

## Ver los resultados como una visualización

Después de crear las tablas, es posible cargarlas en dataframes y visualizar los datos.

1. En el cuaderno *Crear una canalización con tablas Delta Live*, agrega una nueva celda de código y ejecuta el código siguiente para cargar `aggregated_covid_data` en un dataframe:

    ```sql
    %sql
    
    SELECT * FROM aggregated_covid_data
    ```

1. Encima de la tabla de resultados, selecciona **+** y luego **Visualización** para ver el editor de visualización y luego aplica las siguientes opciones:
    - **Tipo de visualización**: líneas
    - **Columna X**: Report_Date
    - **** Columna Y: *agrega una nueva columna y selecciona***Total_Confirmed**. *Aplica la agregación***Suma****.

1. Guarda la visualización y vuelve a ejecutar la celda de código para ver el gráfico resultante en el cuaderno.

## Limpieza

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si has terminado de explorar Azure Databricks, puedes eliminar los recursos que has creado para evitar costes innecesarios de Azure y liberar capacidad en tu suscripción.
