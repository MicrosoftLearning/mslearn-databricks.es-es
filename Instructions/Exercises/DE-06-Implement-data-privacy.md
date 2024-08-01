---
lab:
  title: "Implementación de la privacidad y la gobernanza de datos mediante Microsoft\_Purview y Unity\_Catalog con Azure\_Databricks"
---

# Implementación de la privacidad y la gobernanza de datos mediante Microsoft Purview y Unity Catalog con Azure Databricks

Microsoft Purview permite una gobernanza de datos completa en todo el estado de datos, la integración sin problemas con Azure Databricks para administrar los datos de Lakehouse y la incorporación de metadatos al mapa de datos. Unity Catalog mejora esto al proporcionar administración y gobernanza de datos centralizadas, lo que simplifica la seguridad y el cumplimiento en las áreas de trabajo de Databricks.

Este laboratorio se tarda aproximadamente **30** minutos en completarse.

## Aprovisiona un área de trabajo de Azure Databricks.

> **Sugerencia**: Si ya tiene un área de trabajo de Azure Databricks, puede omitir este procedimiento y usar el área de trabajo existente.

En este ejercicio, se incluye un script para aprovisionar una nueva área de trabajo de Azure Databricks. El script intenta crear un recurso de área de trabajo de Azure Databricks de nivel *Premium* en una región en la que la suscripción de Azure tiene cuota suficiente para los núcleos de proceso necesarios en este ejercicio, y da por hecho que la cuenta de usuario tiene permisos suficientes en la suscripción para crear un recurso de área de trabajo de Azure Databricks. Si se produjese un error en el script debido a cuota o permisos insuficientes, intente [crear un área de trabajo de Azure Databricks de forma interactiva en Azure Portal](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. En un explorador web, inicia sesión en [Azure Portal](https://portal.azure.com) en `https://portal.azure.com`.

2. Usa el botón **[\>_]** a la derecha de la barra de búsqueda en la parte superior de la página para crear un nuevo Cloud Shell en Azure Portal, selecciona un entorno de ***PowerShell*** y crea almacenamiento si se te solicita. Cloud Shell proporciona una interfaz de línea de comandos en un panel situado en la parte inferior de Azure Portal, como se muestra a continuación:

    ![Azure Portal con un panel de Cloud Shell](./images/cloud-shell.png)

    > **Nota**: Si creaste anteriormente un Cloud Shell que usa un entorno de *Bash*, usa el menú desplegable situado en la parte superior izquierda del panel de Cloud Shell para cambiarlo a ***PowerShell***.

3. Tenga en cuenta que puede cambiar el tamaño de Cloud Shell arrastrando la barra de separación en la parte superior del panel, o usando los iconos **&#8212;** , **&#9723;** y **X** en la parte superior derecha para minimizar, maximizar y cerrar el panel. Para obtener más información sobre el uso de Azure Cloud Shell, consulte la [documentación de Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

4. En el panel de PowerShell, introduce los siguientes comandos para clonar este repositorio:

     ```powershell
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
     ```

5. Una vez clonado el repositorio, escriba el siguiente comando para ejecutar el script **setup.ps1**, que aprovisiona un área de trabajo de Azure Databricks en una región disponible:

     ```powershell
    ./mslearn-databricks/setup.ps1
     ```

6. Si se solicita, elige la suscripción que quieres usar (esto solo ocurrirá si tienes acceso a varias suscripciones de Azure).

7. Espera a que se complete el script: normalmente puede tardar entre 5 y 10 minutos, pero en algunos casos puede tardar más. Mientras espera, revise el artículo [Introducción a Delta Lake](https://docs.microsoft.com/azure/databricks/delta/delta-intro) en la documentación de Azure Databricks.

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

## Configuración de Unity Catalog

Los metastores de Unity Catalog registran metadatos sobre objetos protegibles (como tablas, volúmenes, ubicaciones externas y recursos compartidos) y los permisos que rigen el acceso a ellos. En cada metastore, se expone un espacio de nombres de tres niveles (`catalog`.`schema`.`table`) que sirve de ayuda a la hora de organizar los datos. Debe tener un metastore para cada región en la que opera su organización. Para trabajar con Unity Catalog, los usuarios deben estar en un área de trabajo que esté asociada a un metastore de su región.

1. En la barra lateral, selecciona **Catálogo**.

2. En el Explorador de catálogos, debe estar presente un Unity Catalog predeterminado con el nombre del área de trabajo (**databricks-*xxxxxxx*** si usaste el script de instalación para crearlo). Selecciona el catálogo y, luego, en la parte superior del panel derecho, selecciona **Crear esquema**.

3. Asigna un nombre al nuevo **comercio electrónico** de esquema, elige la ubicación de almacenamiento creada con el área de trabajo y selecciona **Crear**.

4. Selecciona el catálogo y, en el panel derecho, elige la pestaña **Áreas de trabajo**. Comprueba que el área de trabajo tiene acceso `Read & Write` a él.

## Ingesta de datos de ejemplo en Azure Databricks

1. Descarga los archivos de datos de ejemplo:
   * [customers.csv](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/DE-05/customers.csv)
   * [products.csv](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/DE-05/products.csv)
   * [sales.csv](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/DE-05/sales.csv)

2. En el área de trabajo de Azure Databricks, en la parte superior del explorador de catálogos, selecciona **+** y, luego, **Agregar datos**.

3. En la nueva ventana, selecciona **Cargar archivos en el volumen**.

4. En la nueva ventana, ve al esquema `ecommerce`, expándelo y selecciona **Crear un volumen**.

5. Asigna un nombre a **sample_data** del nuevo volumen y selecciona **Crear**.

6. Selecciona el nuevo volumen y carga los archivos `customers.csv`, `products.csv`y `sales.csv`. Seleccione **Cargar**.

7. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**. En la lista desplegable **Conectar**, selecciona el clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

8. En la primera celda del cuaderno, escribe el siguiente código para crear tablas a partir de los archivos CSV:

     ```python
    # Load Customer Data
    customers_df = spark.read.format("csv").option("header", "true").load("/Volumes/databricksxxxxxxx/ecommerce/sample_data/customers.csv")
    customers_df.write.saveAsTable("ecommerce.customers")

    # Load Sales Data
    sales_df = spark.read.format("csv").option("header", "true").load("/Volumes/databricksxxxxxxx/ecommerce/sample_data/sales.csv")
    sales_df.write.saveAsTable("ecommerce.sales")

    # Load Product Data
    products_df = spark.read.format("csv").option("header", "true").load("/Volumes/databricksxxxxxxx/ecommerce/sample_data/products.csv")
    products_df.write.saveAsTable("ecommerce.products")
     ```

>**Nota:** En la ruta de acceso del archivo `.load`, reemplaza `databricksxxxxxxx` por el nombre del catálogo.

9. En el explorador de catálogos, ve al volumen `sample_data` y comprueba que contiene las nuevas tablas.
    
## Configuración de Microsoft Purview

Microsoft Purview es un servicio unificado de gobernanza de datos que ayuda a las organizaciones a administrar y proteger sus datos en varios entornos. Con características como la prevención de pérdida de datos, la protección de la información y la administración de cumplimiento, Microsoft Purview proporciona herramientas para comprender, administrar y proteger los datos a lo largo de su ciclo de vida.

1. Acceda a [Azure Portal](https://portal.azure.com/).

2. Selecciona **Crear un recurso** y busca **Microsoft Purview**.

3. Crea un recurso de **Microsoft Purview** con la siguiente configuración:
    - **Suscripción**: *Seleccione la suscripción de Azure*
    - **Grupo de recursos**: *elige el mismo grupo de recursos como área de trabajo de Azure Databricks*
    - **Nombre de cuenta de Microsoft Purview**: *nombre único que prefieras*
    - **Ubicación**: *selecciona la misma región que tu área de trabajo de Azure Databricks*

4. Seleccione **Revisar + crear**. Espera la validación y, después, selecciona **Crear**.

5. Espere a que la implementación finalice. Luego, ve al recurso de Microsoft Purview implementado en Azure Portal.

6. En el portal de gobernanza de Microsoft Purview, ve a la sección **Mapa de datos** de la barra lateral.

7. En el panel **Orígenes de datos**, selecciona **Registrar**.

8. En la ventana **Registrar origen de datos**, busca y selecciona **Azure Databricks**. Seleccione **Continuar**.

9. Asigna un nombre único al origen de datos y, luego, selecciona el área de trabajo de Azure Databricks. Seleccione **Registrar**.

## Implementación de directivas de gobernanza y privacidad de datos

1. En la sección **Mapa de datos** de la barra lateral, selecciona **Clasificaciones**.

2. En el panel **Clasificaciones**, selecciona **+ Nuevo** y crea una nueva clasificación denominada **DCP** (información de identificación personal). Seleccione **Aceptar**.

3. Selecciona **Data Catalog** en la barra lateral y ve a la tabla de **clientes**.

4. Aplica la clasificación de DCP a las columnas de correo electrónico y teléfono.

5. Ve a Azure Databricks y abre el cuaderno creado anteriormente.
 
6. En una nueva celda, ejecuta el código siguiente para crear una directiva de acceso a datos para restringir el acceso a los datos de DCP.

     ```sql
    CREATE OR REPLACE TABLE ecommerce.customers (
      customer_id STRING,
      name STRING,
      email STRING,
      phone STRING,
      address STRING,
      city STRING,
      state STRING,
      zip_code STRING,
      country STRING
    ) TBLPROPERTIES ('data_classification'='PII');

    GRANT SELECT ON TABLE ecommerce.customers TO ROLE data_scientist;
    REVOKE SELECT (email, phone) ON TABLE ecommerce.customers FROM ROLE data_scientist;
     ```

7. Intenta consultar la tabla de clientes como usuario con el rol de data_scientist. Comprueba que el acceso a las columnas DCP (correo electrónico y teléfono) está restringido.

## Limpiar

En el portal de Azure Databricks, en la página **Proceso**, seleccione el clúster y **&#9632; Finalizar** para apagarlo.

Si ha terminado de explorar Azure Databricks, puede eliminar los recursos que ha creado para evitar costos innecesarios de Azure y liberar capacidad en su suscripción.
