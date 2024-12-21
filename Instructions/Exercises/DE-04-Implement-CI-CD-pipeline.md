---
lab:
  title: Implementación de flujos de trabajo de CI/CD con Azure Databricks
---

# Implementación de flujos de trabajo de CI/CD con Azure Databricks

La implementación de flujos de trabajo de CI/CD con Acciones de GitHub y Azure Databricks puede simplificar el proceso de desarrollo y mejorar la automatización. Las Acciones de GitHub proporcionan una plataforma eficaz para automatizar flujos de trabajo de software, incluida la integración continua (CI) y la entrega continua (CD). Cuando se integra con Azure Databricks, estos flujos de trabajo pueden ejecutar tareas de datos complejas, como ejecutar cuadernos o implementar actualizaciones en entornos de Databricks. Por ejemplo, puedes usar Acciones de GitHub para automatizar la implementación de los cuadernos de Databricks, administrar cargas del sistema de archivos de Databricks y configurar la CLI de Databricks dentro de los flujos de trabajo. Esta integración facilita un ciclo de desarrollo más eficaz y resistente a errores, especialmente para aplicaciones controladas por datos.

Se tardan aproximadamente **40** minutos en completar este laboratorio.

> **Nota**: la interfaz de usuario de Azure Databricks está sujeta a una mejora continua. Es posible que la interfaz de usuario haya cambiado desde que se escribieron las instrucciones de este ejercicio.

>**Nota:** necesitas una cuenta de Github para realizar este ejercicio.

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

## Creación de un cuaderno e ingesta de datos

1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**. En la lista desplegable **Conectar**, selecciona el clúster si aún no está seleccionado. Si el clúster no se está ejecutando, puede tardar un minuto en iniciarse.

2. En la primera celda del cuaderno, escribe el siguiente código, que utiliza comandos del *shell* para descargar los archivos de datos de GitHub en el sistema de archivos utilizado por el clúster.

     ```python
    %sh
    rm -r /dbfs/FileStore
    mkdir /dbfs/FileStore
    wget -O /dbfs/FileStore/sample_sales.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales.csv
     ```

3. Usa la opción del menú **&#9656; Ejecutar celda** situado a la izquierda de la celda para ejecutarla. A continuación, espera a que se complete el trabajo de Spark ejecutado por el código.
   
## Configuración de un repositorio de GitHub

Una vez que conectes un repositorio de GitHub a un área de trabajo de Azure Databricks, puedes configurar canalizaciones de CI/CD en Acciones de GitHub que se desencadenen con los cambios realizados en el repositorio.

1. Ve a la [cuenta de GitHub](https://github.com/) y crea un nuevo repositorio para tu proyecto.

2. Clona el repositorio en la máquina local mediante `git clone`.

3. Descarga los archivos necesarios para este ejercicio en el repositorio local:
   - [Archivo CSV](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales.csv)
   - [Notebook de Databricks](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales_notebook.dbc)
   - [Archivo de configuración del trabajo](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/job-config.json):

   Confirma e inserta los cambios.

## Configuración de secretos de repositorio

Los secretos son variables que se crean en una organización, repositorio o entorno de repositorio. Los secretos que crees están disponibles para usarlos en flujos de trabajo de Acciones de GitHub. Acciones de GitHub solo puede leer un secreto si incluyes explícitamente el secreto en un flujo de trabajo.

A medida que los flujos de trabajo de Acciones de GitHub necesitan acceder a los recursos de Azure Databricks, las credenciales de autenticación se almacenarán como variables cifradas que se usarán con las canalizaciones de CI/CD.

Antes de crear secretos de repositorio, debes generar un token de acceso personal en Azure Databricks:

1. En el área de trabajo de Azure Databricks, selecciona el nombre de usuario de Azure Databricks en la barra superior y después selecciona **Configuración** en la lista desplegable.

2. Selecciona **Desarrollador**.

3. Junto a **Tokens de acceso**, selecciona **Administrar**.

4. Selecciona **Generar nuevo token** y, después, selecciona **Generar**.

5. Copia el token mostrado en una ubicación segura y después selecciona **Listo**.

6. En la página de tu repositorio, selecciona la pestaña **Configuración**.

   ![La pestaña Configuración GitHub](./images/github-settings.png)

7. En la barra lateral izquierda, selecciona **Secretos y variables** y después selecciona **Acciones**.

8. Selecciona **Nuevo secreto de repositorio** y agrega cada una de estas variables:
   - **Nombre:** DATABRICKS_HOST **Secreto:** agrega la dirección URL del área de trabajo de Databricks.
   - **Nombre:** DATABRICKS_TOKEN **Secreto:** agrega el token de acceso generado anteriormente.

## Configuración de canalizaciones de integración y entrega continuas

Ahora que has almacenado las variables necesarias para acceder al área de trabajo de Azure Databricks desde GitHub, crearás flujos de trabajo para automatizar la ingesta y el procesamiento de datos, lo que se desencadenará cada vez que se actualice el repositorio.

1. Selecciona la pestaña **Acciones** de la página de tu repositorio.

    ![Acerca de las Acciones de GitHub](./images/github-actions.png)

2. Selecciona **configurar un flujo de trabajo tú mismo** y escribe el código siguiente:

     ```yaml
    name: CI Pipeline for Azure Databricks

    on:
      push:
        branches:
          - main
      pull_request:
        branches:
          - main

    jobs:
      deploy:
        runs-on: ubuntu-latest

        steps:
        - name: Checkout code
          uses: actions/checkout@v3

        - name: Set up Python
          uses: actions/setup-python@v4
          with:
            python-version: '3.x'

        - name: Install Databricks CLI
          run: |
            pip install databricks-cli

        - name: Configure Databricks CLI
          run: |
            databricks configure --token <<EOF
            ${{ secrets.DATABRICKS_HOST }}
            ${{ secrets.DATABRICKS_TOKEN }}
            EOF

        - name: Download Sample Data from DBFS
          run: databricks fs cp dbfs:/FileStore/sample_sales.csv . --overwrite
     ```

Este código instalará y configurará la CLI de Databricks y descargará los datos de ejemplo en el repositorio cada vez que se inserte una confirmación o se combine una solicitud de incorporación de cambios.

3. Asigna un nombre a la **canalización de CI** de flujo de trabajo y selecciona **Confirmar cambios**. La canalización se ejecutará automáticamente y podrás comprobar su estado en la pestaña **Acciones**.

Una vez completado el flujo de trabajo, es el momento de configurar las configuraciones de la canalización de CD.

4. Ve a la página del área de trabajo, selecciona **Proceso** y después selecciona el clúster.

5. En la página del clúster, selecciona **Más...** y después selecciona **Ver JSON**. Copia el id. del clúster.

6. Abre `job-config.json` en el repositorio y reemplaza `your_cluster_id` por el id. del clúster que acabas de copiar. Reemplaza también `/Workspace/Users/your_username/your_notebook` por la ruta de acceso del área de trabajo donde deseas almacenar el cuaderno usado en la canalización. Confirma los cambios.

> **Nota:** si vas a la pestaña **Acciones**, verás que la canalización de CI comenzó a ejecutarse de nuevo. Puesto que se supone que se desencadena cada vez que se inserta una confirmación, el cambio `job-config.json` implementará la canalización según lo previsto.

7. En la pestaña **Acciones**, crea un flujo de trabajo denominado **canalización de CD** y escribe el código siguiente:

     ```yaml
    name: CD Pipeline for Azure Databricks

    on:
      push:
        branches:
          - main

    jobs:
      deploy:
        runs-on: ubuntu-latest

        steps:
        - name: Checkout code
          uses: actions/checkout@v3

        - name: Set up Python
          uses: actions/setup-python@v4
          with:
            python-version: '3.x'

        - name: Install Databricks CLI
          run: pip install databricks-cli

        - name: Configure Databricks CLI
          run: |
            databricks configure --token <<EOF
            ${{ secrets.DATABRICKS_HOST }}
            ${{ secrets.DATABRICKS_TOKEN }}
            EOF
        - name: Upload Notebook to DBFS
          run: databricks fs cp /path/to/your/notebook /Workspace/Users/your_username/your_notebook --overwrite
          env:
            DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}

        - name: Run Databricks Job
          run: |
            databricks jobs create --json-file job-config.json
            databricks jobs run-now --job-id $(databricks jobs list | grep 'CD pipeline' | awk '{print $1}')
          env:
            DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
     ```

Antes confirmar los cambios, reemplaza por `/path/to/your/notebook` la ruta de acceso al directorio donde tienes el cuaderno en el repositorio y `/Workspace/Users/your_username/your_notebook` por la ruta de acceso del archivo donde deseas que se importe el cuaderno en el área de trabajo de Azure Databricks.

Este código volverá a instalar y configurar la CLI de Databricks, importará el cuaderno al sistema de archivos de Databricks y creará y ejecutará un trabajo que puedes supervisar en la página **Flujos de trabajo** del área de trabajo. Comprueba la salida y comprueba que se ha modificado el ejemplo de datos.

## Limpieza

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si has terminado de explorar Azure Databricks, puedes eliminar los recursos que has creado para evitar costes innecesarios de Azure y liberar capacidad en tu suscripción.
