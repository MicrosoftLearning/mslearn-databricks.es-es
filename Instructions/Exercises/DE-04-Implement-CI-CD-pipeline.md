---
lab:
  title: "Implementación de canalizaciones de CI/CD con Azure\_Databricks y Azure\_DevOps o Azure\_Databricks y GitHub"
---

# Implementación de canalizaciones de CI/CD con Azure Databricks y Azure DevOps o Azure Databricks y GitHub

La implementación de canalizaciones de integración continua (CI) e implementación continua (CD) con Azure Databricks y Azure DevOps o Azure Databricks y GitHub implica la configuración de una serie de pasos automatizados para asegurarse de que los cambios de código se integran, prueban e implementan de forma eficaz. El proceso normalmente incluye la conexión a un repositorio de Git, la ejecución de trabajos mediante Azure Pipelines para compilar y llevar a cabo la prueba de código unitaria, e implementar los artefactos de compilación para su uso en cuadernos de Databricks. Este flujo de trabajo posibilita un ciclo de desarrollo sólido, lo que permite la integración y entrega continuas en línea con las prácticas modernas de DevOps.

Se tardan aproximadamente **40** minutos en completar este laboratorio.

>**Nota:** Necesitas una cuenta de Github y acceso a Azure DevOps para realizar este ejercicio.

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

## Creación de un cuaderno e ingesta de datos

1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**. En la lista desplegable **Conectar**, selecciona el clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

2. En la primera celda del cuaderno, escriba el siguiente código, que utiliza comandos del *shell* para descargar los archivos de datos de GitHub en el sistema de archivos utilizado por el clúster.

     ```python
    %sh
    rm -r /dbfs/FileStore
    mkdir /dbfs/FileStore
    wget -O /dbfs/FileStore/sample_sales.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales.csv
     ```

3. Use la opción del menú **&#9656; Ejecutar celda** situado a la izquierda de la celda para ejecutarla. A continuación, espere a que se complete el trabajo de Spark ejecutado por el código.
   
## Configuración de un repositorio de GitHub y un proyecto de Azure DevOps

Una vez que conectes un repositorio de GitHub a un proyecto de Azure DevOps, puedes configurar canalizaciones de CI que se desencadenen con los cambios realizados en el repositorio.

1. Ve a la [cuenta de GitHub](https://github.com/) y crea un nuevo repositorio para tu proyecto.

2. Clona el repositorio en la máquina local mediante `git clone`.

3. Descarga el [archivo CSV](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales.csv) en el repositorio local y confirma los cambios.

4. Descarga el [cuaderno de Databricks](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales_notebook.dbc) que se usará para leer el archivo CSV y realizar la transformación de datos. Confirme los cambios.

5. Ve al [portal de Azure DevOps](https://azure.microsoft.com/en-us/products/devops/) y crea un nuevo proyecto.

6. En el proyecto de Azure DevOps, ve a la sección **Repos** y selecciona **Importar** para conectarlo al repositorio de GitHub.

7. En la barra de la izquierda, ve a **Configuración del proyecto > Conexiones de servicio**.

8. Selecciona **Crear conexión de servicio** y, luego, **Azure Resource Manager**.

9. En el panel **Método de autenticación**, selecciona **Federación de identidades de carga de trabajo (automática).** Seleccione **Siguiente**.

10. En **Nivel de ámbito**, selecciona **Suscripción**. Selecciona la suscripción y el grupo de recursos donde hayas creado el área de trabajo de Databricks.

11. Asigna un nombre a la conexión de servicio y marca la opción **Conceder permiso de acceso a todas las canalizaciones**. Seleccione **Guardar**.

Ahora el proyecto de DevOps tiene acceso al área de trabajo de Databricks y puedes conectarlo a las canalizaciones.

## Configuración de la canalización de CI

1. En la barra izquierda, ve a **Canalizaciones** y selecciona **Crear canalización**.

2. Selecciona **GitHub** como origen y selecciona el repositorio.

3. En el panel **Configura la canalización**, selecciona **Canalización de inicio** y usa la siguiente configuración de YAML para la canalización de CI:

```yaml
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: UsePythonVersion@0
  inputs:
    versionSpec: '3.x'
    addToPath: true

- script: |
    pip install databricks-cli
  displayName: 'Install Databricks CLI'

- script: |
    databricks fs cp dbfs:/FileStore/sample_sales.csv .
  displayName: 'Download Sample Data from DBFS'

- script: |
    python -m unittest discover -s tests
  displayName: 'Run Unit Tests'
```

4. Seleccione **Guardar y ejecutar**.

Este archivo YAML configurará una canalización de CI que se desencadena mediante cambios en la rama `main` del repositorio. La canalización configura un entorno de Python, instala CLI de Databricks, descarga los datos de ejemplo del área de trabajo de Databricks y ejecuta pruebas unitarias de Python. Se trata de una configuración común para los flujos de trabajo de CI.

## Configuración de la canalización de CD

1. En la barra izquierda, ve a **Canalizaciones > Versiones** y selecciona **Crear versión**.

2. Selecciona tu canalización de compilación como origen del artefacto.

3. Agrega una fase y configura las tareas que se van a implementar en Azure Databricks:

```yaml
stages:
- stage: Deploy
  jobs:
  - job: DeployToDatabricks
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: UsePythonVersion@0
      inputs:
        versionSpec: '3.x'
        addToPath: true

    - script: |
        pip install databricks-cli
      displayName: 'Install Databricks CLI'

    - script: |
        databricks workspace import_dir /path/to/notebooks /Workspace/Notebooks
      displayName: 'Deploy Notebooks to Databricks'
```

Antes de ejecutar esta canalización, reemplaza por `/path/to/notebooks` la ruta de acceso al directorio donde tienes el cuaderno en el repositorio y `/Workspace/Notebooks` por la ruta de acceso del archivo donde deseas que el cuaderno se guarde en el área de trabajo de Databricks.

4. Seleccione **Guardar y ejecutar**.

## Ejecución de las canalizaciones

1. En tu repositorio local, agrega la siguiente línea al final del archivo `sample_sales.csv`:

     ```sql
    2024-01-01,ProductG,1,500
     ```

2. Confirma e inserta los cambios en el repositorio de GitHub.

3. Los cambios en el repositorio desencadenarán la canalización de CI. Comprueba que la canalización de CI se completa correctamente.

4. Crea una nueva versión en la canalización de versión e implementa los cuadernos en Databricks. Comprueba que los cuadernos se implementan y se ejecutan correctamente en el área de trabajo de Databricks.

## Limpiar

En el portal de Azure Databricks, en la página **Proceso**, seleccione el clúster y **&#9632; Finalizar** para apagarlo.

Si ha terminado de explorar Azure Databricks, puede eliminar los recursos que ha creado para evitar costos innecesarios de Azure y liberar capacidad en su suscripción.







