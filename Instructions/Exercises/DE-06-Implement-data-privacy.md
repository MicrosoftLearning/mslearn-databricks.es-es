---
lab:
  title: "Implementación de la privacidad y la gobernanza de datos mediante Unity\_Catalog con Azure\_Databricks"
---

# Implementación de la privacidad y la gobernanza de datos mediante Unity Catalog con Azure Databricks

Unity Catalog ofrece una solución de gobernanza centralizada para datos e IA, simplificando la seguridad al proporcionar un lugar central para administrar y auditar el acceso a los datos. Admite listas de control de acceso (ACL) específicas y enmascaramiento de datos dinámicos, que son esenciales para proteger la información confidencial. 

Este laboratorio se tarda aproximadamente **30** minutos en completarse.

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

     ```powershell
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
     ```

5. Una vez clonado el repositorio, escribe el siguiente comando para ejecutar el script **setup.ps1**, que aprovisiona un área de trabajo de Azure Databricks en una región disponible:

     ```powershell
    ./mslearn-databricks/setup.ps1
     ```

6. Si se solicita, elige la suscripción que quieres usar (esto solo ocurrirá si tienes acceso a varias suscripciones de Azure).

7. Espera a que se complete el script: normalmente puede tardar entre 5 y 10 minutos, pero en algunos casos puede tardar más. Mientras esperas, revisa el artículo [Introducción a Delta Lake](https://docs.microsoft.com/azure/databricks/delta/delta-intro) en la documentación de Azure Databricks.

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
    - **Tipo de nodo**: Standard_D4ds_v5
    - **Finaliza después de** *20* **minutos de inactividad**

1. Espera a que se cree el clúster. Esto puede tardar un par de minutos.

    > **Nota**: si el clúster no se inicia, es posible que la suscripción no tenga cuota suficiente en la región donde se aprovisiona el área de trabajo de Azure Databricks. Para obtener más información, consulta [El límite de núcleos de la CPU impide la creación de clústeres](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si esto sucede, puedes intentar eliminar el área de trabajo y crear una nueva en otra región. Puedes especificar una región como parámetro para el script de configuración de la siguiente manera: `./mslearn-databricks/setup.ps1 eastus`

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

6. Selecciona el nuevo volumen y carga los archivos `customers.csv`, `products.csv`y `sales.csv`. Selecciona **Cargar**.

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

9. En el explorador de catálogos, ve al esquema `ecommerce` y comprueba que contiene las nuevas tablas.
    
## Configuración de ACL y enmascaramiento de datos dinámicos

Las listas de control de acceso (ACL) son un aspecto fundamental de la seguridad de los datos en Azure Databricks, lo que te permite configurar permisos para varios objetos de área de trabajo. Con Unity Catalog, puedes centralizar la gobernanza y la auditoría del acceso a datos, lo que proporciona un modelo de seguridad específico que es esencial para administrar los recursos de datos e IA. 

1. En una nueva celda, ejecuta el código siguiente para crear una vista segura de la tabla `customers` para restringir el acceso a los datos de DCP (información de identificación personal).

     ```sql
    CREATE VIEW ecommerce.customers_secure_view AS
    SELECT 
        customer_id, 
        name, 
        address,
        city,
        state,
        zip_code,
        country, 
        CASE 
            WHEN current_user() = 'admin_user@example.com' THEN email
            ELSE NULL 
        END AS email, 
        CASE 
            WHEN current_user() = 'admin_user@example.com' THEN phone 
            ELSE NULL 
        END AS phone
    FROM ecommerce.customers;
     ```

2. Consulta la vista segura:

     ```sql
    SELECT * FROM ecommerce.customers_secure_view
     ```

Comprueba que el acceso a las columnas DCP (correo electrónico y teléfono) está restringido, ya que no tienes acceso a los datos como `admin_user@example.com`.

## Limpieza

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si has terminado de explorar Azure Databricks, puedes eliminar los recursos que has creado para evitar costes innecesarios de Azure y liberar capacidad en tu suscripción.
