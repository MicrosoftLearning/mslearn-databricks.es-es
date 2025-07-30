---
lab:
  title: Entrenamiento de un modelo con AutoML
---

# Entrenamiento de un modelo con AutoML

AutoML es una característica de Azure Databricks que prueba varios algoritmos y parámetros con sus datos para entrenar un modelo de Machine Learning óptimo.

Este ejercicio debería tardar en completarse **45** minutos aproximadamente.

> **Nota**: la interfaz de usuario de Azure Databricks está sujeta a una mejora continua. Es posible que la interfaz de usuario haya cambiado desde que se escribieron las instrucciones de este ejercicio.

## Antes de empezar

Necesitará una [suscripción de Azure](https://azure.microsoft.com/free) en la que tenga acceso de nivel administrativo.

## Aprovisiona un área de trabajo de Azure Databricks.

> **Nota**: Para este ejercicio, necesita un área de trabajo de Azure Databricks **Premium** en una región en la que se admita el *servicio de modelo*. Consulte las [Regiones de Azure Databricks](https://learn.microsoft.com/azure/databricks/resources/supported-regions) para obtener más información sobre sus funcionalidades regionales. Si ya tiene un área de trabajo de Azure Databricks *Premium* o *de prueba* en una región adecuada, puede omitir este procedimiento y usar el área de trabajo existente.

En este ejercicio se incluye un script para aprovisionar una nueva área de trabajo de Azure Databricks. El script intenta crear un recurso de área de trabajo de Azure Databricks de nivel *Premium* en una región en la que la suscripción de Azure tiene cuota suficiente para los núcleos de proceso necesarios en este ejercicio, y da por hecho que la cuenta de usuario tiene permisos suficientes en la suscripción para crear un recurso de área de trabajo de Azure Databricks. Si se produjese un error en el script debido a cuota o permisos insuficientes, intenta [crear un área de trabajo de Azure Databricks de forma interactiva en Azure Portal](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

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
7. Espera a que se complete el script: normalmente puede tardar entre 5 y 10 minutos, pero en algunos casos puede tardar más. Mientras espera, revise el artículo [¿Qué es el AutoML?](https://learn.microsoft.com/azure/databricks/machine-learning/automl/) en la documentación de Azure Databricks.

## Crear un clúster

Azure Databricks es una plataforma de procesamiento distribuido que usa clústeres* de Apache Spark *para procesar datos en paralelo en varios nodos. Cada clúster consta de un nodo de controlador para coordinar el trabajo y nodos de trabajo para hacer tareas de procesamiento. En este ejercicio, crearás un clúster de *nodo único* para minimizar los recursos de proceso usados en el entorno de laboratorio (en los que se pueden restringir los recursos). En un entorno de producción, normalmente crearías un clúster con varios nodos de trabajo.

> **Sugerencia**: Si ya dispone de un clúster con una versión de runtime 13.3 LTS **<u>ML</u>** o superior en su área de trabajo de Azure Databricks, puede utilizarlo para completar este ejercicio y omitir este procedimiento.

1. En Azure Portal, vaya al grupo de recursos **msl-*xxxxxxx*** que se creó con el script (o al grupo de recursos que contiene el área de trabajo de Azure Databricks existente)
1. Selecciona el recurso Azure Databricks Service (llamado **databricks-*xxxxxxx*** si usaste el script de instalación para crearlo).
1. En la página **Información general** del área de trabajo, usa el botón **Inicio del área de trabajo** para abrir el área de trabajo de Azure Databricks en una nueva pestaña del explorador; inicia sesión si se solicita.

    > **Sugerencia**: al usar el portal del área de trabajo de Databricks, se pueden mostrar varias sugerencias y notificaciones. Descártalas y sigue las instrucciones proporcionadas para completar las tareas de este ejercicio.

1. En la barra lateral de la izquierda, selecciona la tarea **(+) Nuevo** y luego selecciona **Clúster**.
1. En la página **Nuevo clúster**, crea un clúster con la siguiente configuración:
    - **Nombre del clúster**: clúster del *Nombre de usuario*  (el nombre del clúster predeterminado)
    - **Directiva**: Unrestricted (Sin restricciones)
    - **Modo de clúster** de un solo nodo
    - **Modo de acceso**: usuario único (*con la cuenta de usuario seleccionada*)
    - **Versión de runtime de Databricks**: *Seleccione la edición de **<u>ML</u>** de la última versión no beta más reciente del runtime (**No** una versión de runtime estándar) que:*
        - ***No** usa una GPU*
        - *Incluye Scala > **2.11***
        - *Incluye Spark > **3.4***
    - **Utilizar la Aceleración de fotones**: <u>No</u> seleccionada
    - **Tipo de nodo**: Standard_D4ds_v5
    - **Finaliza después de** *20* **minutos de inactividad**

1. Espera a que se cree el clúster. Esto puede tardar un par de minutos.

> **Nota**: si el clúster no se inicia, es posible que la suscripción no tenga cuota suficiente en la región donde se aprovisiona el área de trabajo de Azure Databricks. Para obtener más información, consulta [El límite de núcleos de la CPU impide la creación de clústeres](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si esto sucede, puedes intentar eliminar el área de trabajo y crear una nueva en otra región. Puedes especificar una región como parámetro para el script de configuración de la siguiente manera: `./mslearn-databricks/setup.ps1 eastus`

## Carga de datos de entrenamiento en SQL Warehouse

Para entrenar un modelo de Machine Learning mediante AutoML, debe cargar los datos de entrenamiento. En este ejercicio, entrenará un modelo para clasificar un pingüino como una de tres especies en función de observaciones, incluidas sus medidas corporales y su ubicación. Cargará los datos de entrenamiento que incluyen la etiqueta de especie en una tabla de un almacenamiento de datos de Azure Databricks.

1. En el portal de Azure Databricks de su área de trabajo, en la barra lateral, en **SQL**, seleccione**SQL Warehouses**.
1. Observa que el área de trabajo ya incluye una instancia de Almacén de SQL denominada **Almacén de inicio sin servidor**.
1. En el menú **Acciones** (**⁝**) del Almacén de SQL, selecciona **Editar**. Después, establece la propiedad **Tamaño del clúster** en **2X-Small** y guarda los cambios.
1. Usa el botón **Iniciar** para iniciar el Almacén de SQL (que puede tardar un par de minutos).

    > **Nota**: si el Almacén de SQL no se inicia, es posible que la suscripción no tenga cuota suficiente en la región donde se aprovisiona el área de trabajo de Azure Databricks. Para más información, consulta [Cuota necesaria de vCPU de Azure](https://docs.microsoft.com/azure/databricks/sql/admin/sql-endpoints#required-azure-vcpu-quota). Si esto sucede, puedes intentar pedir un aumento de cuota, tal como se detalla en el mensaje de error cuando el almacenamiento no se inicia. Como alternativa, puedes intentar eliminar el área de trabajo y crear una nueva en otra región. Puedes especificar una región como parámetro para el script de configuración de la siguiente manera: `./mslearn-databricks/setup.ps1 eastus`

1. Descargue el archivo [**penguins.csv**](https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv) de `https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv` en el equipo local y guárdelo como **penguins.csv**.
1. En el portal del área de trabajo de Azure Databricks, en la barra lateral, selecciona **Nuevo (+)** y después selecciona **Agregar o cargar datos**. En la página **Agregar datos**, selecciona **Crear o modificar tabla** y carga el archivo **penguins.csv** que descargaste en el equipo.
1. En la página **Crear o modificar una tabla desde la página de carga de archivos**, selecciona el esquema **predeterminado** y establece el nombre de tabla como **penguins**. Después selecciona **Crear tabla**.
1. Cuando la tabla esté creada, revisa sus detalles.

## Crear un experimento de AutoML

Ahora que tiene algunos datos, puede usarlos con AutoML para entrenar un modelo.

1. En la barra lateral de la izquierda, seleccione **Experimentos**.
1. En la página **Experimentos**, busca el icono **Clasificación** y selecciona **Iniciar entrenamiento**.
1. Configura el experimento de AutoML con la siguiente configuración:
    - **Clúster**: *Seleccione el clúster*
    - **Conjunto de datos de entrenamiento de entrada**: *Examinar la base de datos **predeterminada** y seleccionar la tabla **pingüinos***
    - **Destino de predicción**: Especie
    - **Nombre del experimento**: Clasificación de pingüinos
    - **Configuración avanzada**:
        - **Métrica de evaluación**: Precisión
        - **Marcos de entrenamiento**: lightgbm, sklearn, xgboost
        - **Timeout**: 5
        - **Columna de tiempo para división de entrenamiento, validación y pruebas**: *Déjelo en blanco*
        - **Etiqueta positiva**: *Déjelo en blanco*
        - **Ubicación del almacenamiento de datos intermedia**: Artefacto de MLflow
1. Use el botón **Iniciar AutoML** para iniciar el experimento. Cierre los cuadros de diálogo de información que se muestran.
1. Espere a que se complete el experimento. Puedes ver los detalles de las ejecuciones que se generan en la pestaña **Ejecuciones**.
1. Después de cinco minutos, el experimento finalizará. La actualización de las ejecuciones mostrará la ejecución que dio lugar al modelo de mejor rendimiento (en función de la métrica de *precisión* seleccionada) en la parte superior de la lista.

## Implementar el modelo de mejor rendimiento

Después de ejecutar un experimento de AutoML, puede explorar el modelo de mejor rendimiento que se generó.

1. En la página del experimento **Clasificación de pingüinos**, selecciona **Ver cuaderno para el mejor modelo** para abrir el cuaderno usado para entrenar al modelo en una nueva pestaña del explorador.
1. Desplácese por las celdas del cuaderno y observe el código que se usó para entrenar al modelo.
1. Cierre la pestaña del explorador que contiene el cuaderno para volver a la página del experimento de **Clasificación de pingüinos**.
1. En la lista de ejecuciones, seleccione el nombre de la primera ejecución (la que produjo el mejor modelo) para abrirla.
1. En la sección **Artefactos**, observe que el modelo se ha guardado como un artefacto de MLflow. A continuación, use el botón **Registrar modelo** para registrarlo como un nuevo modelo denominado **Clasificador de pingüinos**.
1. En la barra lateral de la izquierda, cambie a la página **Modelos**. A continuación, seleccione el modelo **Clasificador de pingüinos** que acaba de registrar.
1. En la página **Clasificador de pingüinos**, use el botón **Usar modelo para la inferencia** para crear un nuevo punto de conexión en tiempo real con los siguientes valores:
    - **Modelo**: Clasificador de pingüinos
    - **Versión del modelo**: 1
    - **Punto de conexión**: clasificar pingüino
    - **Tamaño de proceso**: Pequeña

    El punto de conexión de servicio se hospeda en un nuevo clúster, que puede tardar varios minutos en crearse.
  
1. Cuando se haya creado el punto de conexión, use el botón **Consultar punto de conexión** situado en la parte superior derecha para abrir una interfaz desde la que puede probarlo. A continuación, en la interfaz de prueba, en la pestaña **Explorador**, escriba la siguiente solicitud JSON y use el botón **Enviar solicitud** para llamar al punto de conexión y generar una predicción.

    ```json
    {
      "dataframe_records": [
      {
         "Island": "Biscoe",
         "CulmenLength": 48.7,
         "CulmenDepth": 14.1,
         "FlipperLength": 210,
         "BodyMass": 4450
      }
      ]
    }
    ```

1. Experimente con algunos valores diferentes para las características de los pingüinos y observe los resultados que se devuelven. A continuación, cierre la interfaz de prueba.

## Eliminación del punto de conexión

Cuando el punto de conexión ya no sea necesario, debe eliminarlo para evitar costos innecesarios.

En la página de punto de conexión **clasificar pingüino**, en el menú **&#8285;**, seleccione **Eliminar**.

## Limpiar

En el portal de Azure Databricks, en la página **Proceso**, selecciona el clúster y **&#9632; Finalizar** para apagarlo.

Si ha terminado de explorar Azure Databricks, puede eliminar los recursos que ha creado para evitar costos innecesarios de Azure y liberar capacidad en la suscripción.

> **Más información**: Para obtener más información, consulte [Funcionamiento de AutoML de Databricks](https://learn.microsoft.com/en-us/azure/databricks/machine-learning/automl/how-automl-works) en la documentación de Azure Databricks.
