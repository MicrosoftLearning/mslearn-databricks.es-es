---
lab:
  title: Automatizar un cuaderno de Azure Databricks con Azure Data Factory
---

# Automatizar un cuaderno de Azure Databricks con Azure Data Factory

Puedes usar cuadernos en Azure Databricks para realizar tareas de ingeniería de datos, como procesar archivos de datos y cargar datos en tablas. Cuando necesites organizar estas tareas como parte de una canalización de ingeniería de datos, puedes usar Azure Data Factory.

Este ejercicio debería tardar en completarse **40** minutos aproximadamente.

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

    ```
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

5. Una vez clonado el repositorio, escribe el siguiente comando para ejecutar el script **setup.ps1**, que aprovisiona un área de trabajo de Azure Databricks en una región disponible:

    ```
    ./mslearn-databricks/setup.ps1
    ```

6. Si se solicita, elige la suscripción que quieres usar (esto solo ocurrirá si tienes acceso a varias suscripciones de Azure).
7. Espera a que se complete el script: normalmente tarda unos 5 minutos, pero en algunos casos puede tardar más. Mientras esperas, consulta [Qué es Azure Data Factory](https://docs.microsoft.com/azure/data-factory/introduction).

## Creación de un recurso de Azure Data Factory

Además del área de trabajo de Azure Databricks, deberá aprovisionar un recurso de Azure Data Factory en la suscripción.

1. En Azure Portal, cierre el panel de Cloud Shell y vaya al grupo de recursos ***msl-*xxxxxxx*** creado por el script de instalación (o el grupo de recursos que contiene el área de trabajo de Azure Databricks existente).
1. En la barra de herramientas, seleccione **+Crear** y busque `Data Factory`. A continuación, cree un nuevo recurso de **Data Factory** con la siguiente configuración:
    - **Suscripción**: *Su suscripción*
    - **Grupo de recursos**: msl-*xxxxxxx* (o el grupo de recursos que contiene el área de trabajo de Azure Databricks existente)
    - **Nombre**: *Un nombre único, por ejemplo, **adf-xxxxxxx***
    - **Región**: *La misma región que el área de trabajo de Azure Databricks (o cualquier otra región disponible si esta no aparece)*
    - **Versión**: V2
1. Cuando se cree el nuevo recurso, compruebe que el grupo de recursos contiene tanto el área de trabajo de Azure Databricks como los recursos de Azure Data Factory.

## Crear un cuaderno

Puedes crear cuadernos en tu área de trabajo de Azure Databricks para ejecutar código escrito en diversos lenguajes de programación. En este ejercicio, creará un cuaderno sencillo que ingiere datos de un archivo y los guarda en una carpeta del sistema de archivos de Databricks (DBFS).

1. En Azure Portal, vaya al grupo de recursos **msl-*xxxxxxx*** que se creó con el script (o al grupo de recursos que contiene el área de trabajo de Azure Databricks existente).
1. Selecciona el recurso Azure Databricks Service (llamado **databricks-*xxxxxxx*** si usaste el script de instalación para crearlo).
1. En la página **Información general** del área de trabajo, usa el botón **Inicio del área de trabajo** para abrir el área de trabajo de Azure Databricks en una nueva pestaña del explorador; inicia sesión si se solicita.

    > **Sugerencia**: al usar el portal del área de trabajo de Databricks, se pueden mostrar varias sugerencias y notificaciones. Descarta estos elementos y sigue las instrucciones proporcionadas para completar las tareas de este ejercicio.

1. Visualiza el portal del área de trabajo de Azure Databricks y observa que la barra lateral del lado izquierdo contiene iconos para las distintas tareas que puedes realizar.
1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**.
1. Cambia el nombre predeterminado del cuaderno (**Cuaderno sin título *[fecha]***) por `Process Data`
1. En la primera celda del cuaderno, escriba (pero no ejecute) el código siguiente para configurar una variable para la carpeta donde este cuaderno guardará los datos.

    ```python
   # Use dbutils.widget define a "folder" variable with a default value
   dbutils.widgets.text("folder", "data")
   
   # Now get the parameter value (if no value was passed, the default set above will be used)
   folder = dbutils.widgets.get("folder")
    ```

1. Debajo de la celda de código existente, usa el icono **+** para agregar una nueva celda de código. A continuación, en la nueva celda, escriba (pero no ejecute) el código siguiente para descargar los datos y guardarlos en la carpeta:

    ```python
   import urllib3
   
   # Download product data from GitHub
   response = urllib3.PoolManager().request('GET', 'https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv')
   data = response.data.decode("utf-8")
   
   # Save the product data to the specified folder
   path = "dbfs:/{0}/products.csv".format(folder)
   dbutils.fs.put(path, data, True)
    ```

1. En la barra lateral de la izquierda, seleccione **Área de trabajo** y asegúrese de que se muestran los cuadernos de **Procesamiento de datos**. Usará Azure Data Factory para ejecutar el cuaderno como parte de una canalización.

    > **Nota**: El cuaderno podría contener prácticamente cualquier lógica de procesamiento de datos que necesite. Este sencillo ejemplo está diseñado para mostrar los principios clave.

## Habilitar la integración de Azure Databricks con Azure Data Factory

Para usar Azure Databricks desde una canalización de Azure Data Factory, necesitas crear un servicio vinculado en Azure Data Factory que permita el acceso a tu área de trabajo de Azure Databricks.

### Generar token de acceso

1. En el portal de Azure Databricks, en la barra de menú superior derecha, selecciona el nombre de usuario y después **Configuración de usuario** en el menú desplegable.
1. En la página **Configuración de usuario**, selecciona **Desarrollador**. Después, junto a **Tokens de acceso**, selecciona **Administrar**.
1. Selecciona **Generar nuevo token** y genera un nuevo token con el comentario *Data Factory* y una duración en blanco (para que el token no expire). Ten cuidado para **copiar el token cuando aparezca <u>antes</u> de seleccionar *Listo***.
1. Pega el token copiado en un archivo de texto para tenerlo a mano para más adelante en este ejercicio.

## Usar una canalización para ejecutar el cuaderno de Azure Databricks

Ahora que creaste un servicio vinculado, puedes usarlo en una canalización para ejecutar el cuaderno que visualizaste anteriormente.

### Crear una canalización

1. En Azure Data Factory Studio, en el panel de navegación, selecciona **Autor**.
2. En la página **Autor**, en el panel **Recursos Factory**, usa el icono **+** para agregar una **canalización**.
3. En el panel **Propiedades** de la nueva canalización, cambia su nombre a `Process Data with Databricks`. Después, usa el botón **Propiedades** (que tiene un aspecto similar a **<sub>*</sub>**) situado en el extremo derecho de la barra de herramientas para ocultar el panel **Propiedades**.
4. En el panel **Actividades**, expande **Databricks** y arrastra una actividad de **Cuaderno** a la superficie del diseñador de canalizaciones.
5. Con la nueva actividad **Notebook1** seleccionada, establece las siguientes propiedades en el panel inferior:
    - **General:**
        - **Nombre**: `Process Data`
    - **Azure Databricks**:
        - **Servicio vinculado de Databricks**: *selecciona el servicio **AzureDatabricks** vinculado que creaste anteriormente*
    - **Configuración**:
        - **Ruta de acceso del cuaderno**: *Vaya a la carpeta **Usuarios/su_nombre_de_usuario** y seleccione el *cuaderno **Procesamiento de datos**.
        - **Parámetros base**: *Agregue un nuevo parámetro llamado `folder` con el valor `product_data`*.
6. Usa el botón **Validar** encima de la superficie del diseñador de canalizaciones para validar la canalización. Después, usa el botón **Publicar todo** para publicarlo (guardarlo).

### Crear un servicio vinculado en Azure Data Factory

1. Vuelva a Azure Portal y, en el grupo de recursos **msl-*xxxxxxx***, seleccione el recurso **adf*xxxxxxx*** de Azure Data Factory.
2. En la página **Información general**, selecciona **Iniciar Studio** para abrir Azure Data Factory Studio. Inicie sesión si se le solicita hacerlo.
3. En Azure Data Factory Studio, usa el icono **>>** para expandir el panel de navegación de la izquierda. Después, selecciona la página **Administrar**.
4. En la página **Administrar**, en la pestaña **Servicios vinculados**, selecciona **+ Nuevo** para agregar un nuevo servicio vinculado.
5. En el panel **Nuevo servicio vinculado**, selecciona la pestaña **Procesar** en la parte superior. Después, selecciona **Azure Databricks**.
6. Continúa y crea el servicio vinculado con la siguiente configuración:
    - **Nombre**: `AzureDatabricks`
    - **Descripción**: `Azure Databricks workspace`
    - **Conectar mediante Integration Runtime**: AutoResolveIntegrationRuntime
    - **Método de selección de cuenta**: desde la suscripción de Azure
    - **Suscripción de Azure**: *selecciona tu suscripción*
    - **Área de trabajo de Databricks**: *selecciona tu área de trabajo **databricksxxxxxxx***
    - **Seleccionar clúster**: nuevo clúster de trabajo
    - **Dirección URL del área de trabajo de Databricks**: *configurado automáticamente a la dirección URL de tu área de trabajo de Databricks*
    - **Tipo de autenticación**: token de acceso
    - **** Token de acceso: *pega el token de acceso.*
    - **Versión del clúster**: 13.3 LTS (Spark 3.4.1, Scala 2.12)
    - **Tipo de nodo de clúster**: Standard_D4ds_v5
    - **Versión de Python**: 3
    - **Opciones de trabajador**: fijas
    - **Trabajadores**: 1

### Ejecución de la canalización

1. Encima de la superficie del diseñador de canalizaciones, selecciona **Agregar desencadenador** y después **Desencadenar ahora**.
2. En el panel **Ejecución de la canalización**, selecciona **Aceptar** para ejecutar la canalización.
3. En el panel de navegación de la izquierda, selecciona **Supervisar** y observa la canalización **Procesar datos con Databricks** en la pestaña **Ejecuciones de canalizaciones**. Puede tardar un poco en ejecutarse, ya que crea de forma dinámica un clúster de Spark y ejecuta el cuaderno. Puedes usar el botón **↻ Actualizar** de la página **Ejecuciones de canalizaciones** para actualizar el estado.

    > **Nota**: Si se produce un error en tu canalización, es posible que tu suscripción tenga una cuota insuficiente en la región en la que se aprovisiona tu área de trabajo de Azure Databricks para crear un clúster de trabajos. Para más información consulta [El límite de núcleos de la CPU impide la creación de clústeres](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si esto sucede, puedes intentar eliminar el área de trabajo y crear una nueva en otra región. Puedes especificar una región como parámetro para el script de configuración de la siguiente manera: `./setup.ps1 eastus`

4. Cuando se haya ejecutado correctamente, selecciona su nombre para ver los detalles de la ejecución. Después, en la página **Procesar datos con Databricks**, en la sección **Ejecuciones de actividades**, selecciona la actividad **Procesar datos** y usa su icono ***salida*** para ver el JSON de salida de la actividad, que debería tener el siguiente aspecto:

    ```json
    {
        "runPageUrl": "https://adb-..../run/...",
        "effectiveIntegrationRuntime": "AutoResolveIntegrationRuntime (East US)",
        "executionDuration": 61,
        "durationInQueue": {
            "integrationRuntimeQueue": 0
        },
        "billingReference": {
            "activityType": "ExternalActivity",
            "billableDuration": [
                {
                    "meterType": "AzureIR",
                    "duration": 0.03333333333333333,
                    "unit": "Hours"
                }
            ]
        }
    }
    ```

## Limpieza

Si ha terminado de explorar Azure Databricks, puede eliminar los recursos que ha creado para evitar costos innecesarios de Azure y liberar capacidad en la suscripción.
