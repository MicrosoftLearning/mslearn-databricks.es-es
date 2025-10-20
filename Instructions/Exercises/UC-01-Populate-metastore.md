---
lab:
  title: Rellenado y navegación por el metastore de Unity Catalog
---

# Rellenado y navegación por el metastore de Unity Catalog

Unity Catalog proporciona una solución de gobernanza centralizada para todos los recursos de datos de Azure Databricks. El metastore actúa como contenedor de nivel superior en Unity Catalog, y organiza objetos de datos mediante un espacio de nombres de tres niveles que proporciona límites para el aislamiento de datos y el control de acceso.

En este ejercicio, creará catálogos mediante el Explorador de catálogos, rellenará esquemas y tablas mediante comandos SQL y, después, navegará por la jerarquía mediante consultas SQL y la interfaz del Explorador de catálogos.

Este ejercicio debería tardar en completarse **30** minutos aproximadamente.

> **Nota**: la interfaz de usuario de Azure Databricks está sujeta a una mejora continua. Es posible que la interfaz de usuario haya cambiado desde que se escribieron las instrucciones de este ejercicio.

## Aprovisiona un área de trabajo de Azure Databricks.

> **Sugerencia**: si ya tienes un área de trabajo de Azure Databricks, puedes omitir este procedimiento y usar el área de trabajo existente.

En este ejercicio, se incluye un script para aprovisionar una nueva área de trabajo de Azure Databricks. El script intenta crear un recurso de área de trabajo de Azure Databricks de nivel *Premium* en una región en la que la suscripción de Azure tiene cuota suficiente para los núcleos de proceso necesarios en este ejercicio, y da por hecho que la cuenta de usuario tiene permisos suficientes en la suscripción para crear un recurso de área de trabajo de Azure Databricks. 

Si se produjese un error en el script debido a cuota o permisos insuficientes, intenta [crear un área de trabajo de Azure Databricks de forma interactiva en Azure Portal](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

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
7. Espera a que se complete el script: normalmente tarda unos 5 minutos, pero en algunos casos puede tardar más. Mientras espera, revise el artículo [¿Qué es Unity Catalog?](https://learn.microsoft.com/azure/databricks/data-governance/unity-catalog/) en la documentación de Azure Databricks.

## Abra el área de trabajo de Azure Databricks

1. En Azure Portal, vaya al grupo de recursos **msl-*xxxxxxx*** que se ha creado con el script (o al grupo de recursos que contiene el área de trabajo de Azure Databricks existente).

2. Selecciona el recurso Azure Databricks Service (llamado **databricks-*xxxxxxx*** si usaste el script de instalación para crearlo).

3. En la página **Información general** del área de trabajo, usa el botón **Inicio del área de trabajo** para abrir el área de trabajo de Azure Databricks en una nueva pestaña del explorador; inicia sesión si se solicita.

## Exploración del catálogo predeterminado

1. En el menú de la barra lateral izquierda, haga clic en **Catálogo** para abrir el Explorador de catálogos.

2. Observe que algunos catálogos ya se han creado automáticamente:
   - Un catálogo del **sistema** que contiene metadatos integrados
   - Un catálogo predeterminado con el mismo nombre que el área de trabajo (por ejemplo, **databricks-* xxxxxxx***)

3. Haga clic en el catálogo predeterminado (el que tiene el mismo nombre que el área de trabajo) para explorar su estructura.

4. Observe que ya se han creado dos esquemas en el catálogo predeterminado:
   - **default**: un esquema predeterminado para organizar objetos de datos
   - **information_schema**: un esquema del sistema que contiene metadatos

5. Haga clic en el botón **Detalles** del panel de detalles del catálogo y observe los campos **Ubicación de almacenamiento** y **Tipo**. El tipo **MANAGED_CATALOG** indica que Databricks administra automáticamente el almacenamiento y el ciclo de vida de los recursos de datos dentro de este catálogo.

## Creación de un catálogo

Ahora que ha explorado el catálogo predeterminado, creará un catálogo propio para organizar los datos.

1. Seleccione **Catálogo** en el menú de la izquierda.

2. Seleccione **Catálogos** debajo de **Acceso rápido**.
3. Seleccione **Crear catálogo**.
4. En el cuadro de diálogo **Crear un catálogo nuevo**:
   - Escriba un **nombre de catálogo** (por ejemplo, `my_catalog`)
   - Seleccione **Estándar** como **Tipo** del catálogo
   - En **Ubicación de almacenamiento**, seleccione el nombre del catálogo predeterminado (por ejemplo, **databricks-* xxxxxxx***) para utilizar la misma ubicación de almacenamiento que usa.
   - Haga clic en **Crear**

3. En la ventana **Catálogo creado** que aparece, haga clic en **Ver catálogo**.

4. Haga clic en el catálogo recién creado para ver su estructura. Observe que contiene los esquemas **default** e **information_schema**, similares al catálogo de áreas de trabajo que ha explorado antes.

## Crear un cuaderno

Usará un cuaderno para ejecutar comandos SQL que crean y exploran objetos de Unity Catalog en el nuevo catálogo.

1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**.
   
2. Cambie el nombre predeterminado del cuaderno (**Cuaderno sin título *[fecha]***) a `Populate and navigate the metastore` y, en la lista desplegable **Conectar**, seleccione el **clúster sin servidor** si aún no está seleccionado. Observe que **Sin servidor** está habilitado de manera predeterminada.

3. En la primera celda del cuaderno, escriba y ejecute el código siguiente para establecer el nuevo catálogo como el predeterminado y compruebe lo siguiente:

    ```
    %sql
    USE CATALOG <your_catalog_name>;
    SELECT current_catalog();
    ```

## Creación y administración de esquemas

1. Agregue una nueva celda y ejecute el código siguiente para crear un esquema denominado **sales** y establecerlo como predeterminado:

    ```
    %sql
    CREATE SCHEMA IF NOT EXISTS sales
    COMMENT 'Schema for sales data';

    USE SCHEMA sales;
    SELECT current_schema();
    ```

2. En el **Explorador de catálogos**, vaya al catálogo y expándalo para ver el esquema **sales** que acaba de crear junto con los esquemas **default** e **information_schema**.

3. Para volver al cuaderno, seleccione **Área de trabajo** en el menú de la izquierda y, después, vaya al cuaderno.

## Creación y administración de tablas

1. Agregue una nueva celda y ejecute el código siguiente a fin de crear una tabla administrada para los datos del cliente:

    ```
    %sql
    CREATE OR REPLACE TABLE customers (
      customer_id INT,
      customer_name STRING,
      email STRING,
      city STRING,
      country STRING
    )
    COMMENT 'Customer information table';
    ```

2. Agregue una nueva celda y ejecute el código siguiente para insertar datos de ejemplo y comprobar que se han insertado:

    ```
    %sql
    INSERT INTO customers VALUES
      (1, 'Aaron Gonzales', 'aaron@contoso.com', 'Seattle', 'USA'),
      (2, 'Anne Patel', 'anne@contoso.com', 'London', 'UK'),
      (3, 'Casey Jensen', 'casey@contoso.com', 'Toronto', 'Canada'),
      (4, 'Elizabeth Moore', 'elizabeth@contoso.com', 'Sydney', 'Australia'),
      (5, 'Liam Davis', 'liam@contoso.com', 'Berlin', 'Germany');

    SELECT * FROM customers;
    ```
3. Cambie al **Explorador de catálogos** y vaya a la tabla catálogo > **sales** schema > **customers**. Haga clic en la tabla para explorar:
   - Pestaña **Esquema**: vea las definiciones de columna y los tipos de datos
   - Pestaña **Datos de ejemplo**: consulte una vista previa de los datos que ha insertado
   - Pestaña **Detalles**: vea metadatos de tabla, incluida la ubicación de almacenamiento y el tipo de tabla (ADMINISTRADA)

4. Para volver al cuaderno, seleccione **Área de trabajo** en el menú de la izquierda y, después, vaya al cuaderno.

## Creación y administración de vistas

1. Agregue una nueva celda y ejecute el código siguiente para crear una vista que filtre a los clientes:

    ```
    %sql
    CREATE OR REPLACE VIEW usa_customers AS
    SELECT customer_id, customer_name, email, city
    FROM customers
    WHERE country = 'USA';

    SELECT * FROM usa_customers;
    ```

2. Cambie al **Explorador de catálogos** y vaya al esquema catálogo > **sales**. Observe que ahora se muestran la tabla **customers** y la vista **usa_customers**.

3. Haga clic en la vista **usa_customers**, seleccione la pestaña **Linaje** y luego **Ver gráfico de linaje**. Observe cómo en el gráfico de linaje se muestra la relación de dependencia entre la vista y su tabla de origen. Unity Catalog realiza el seguimiento de estas dependencias de objetos, lo que le ayuda a comprender el flujo de datos y evaluar el impacto de los cambios en las tablas subyacentes.

## Exploración del metastore con comandos SQL

Ahora que ha creado objetos mediante SQL y los ha comprobado en el Explorador de catálogos, se usarán comandos SQL para inspeccionar la estructura del metastore mediante programación.

1. Para volver al cuaderno, seleccione **Área de trabajo** en el menú de la izquierda y, después, vaya al cuaderno.

2. Agregue una nueva celda y ejecute el código siguiente para enumerar todos los catálogos a los que tiene acceso:

    ```
    %sql
    SHOW CATALOGS;
    ```

   Aquí se enumeran todos los catálogos a los que tiene acceso, incluido el catálogo **system**, el catálogo del área de trabajo y el catálogo personalizado.

3. Agregue una nueva celda y ejecute el código siguiente para enumerar todos los esquemas del catálogo actual:

    ```
    %sql
    SHOW SCHEMAS;
    ```

4. Agregue una nueva celda y ejecute el código siguiente para enumerar todas las tablas y vistas del catálogo actual:

    ```
    %sql
    SHOW TABLES;
    ```

5. Agregue una nueva celda y ejecute el código siguiente para usar DESCRIBE a fin de obtener metadatos de tabla detallados:

    ```
    %sql
    DESCRIBE EXTENDED customers;
    ```

   El comando DESCRIBE EXTENDED proporciona información completa sobre la tabla, incluidas las definiciones de columna, las propiedades de la tabla, la ubicación de almacenamiento, etc.

6. Vuelva al **Explorador de catálogos** por última vez. Compare lo que ve en la interfaz visual con la salida SQL. Observe cómo ambos enfoques, los comandos SQL y el Explorador de catálogos, ofrecen vistas diferentes de los mismos metadatos, lo que le proporciona flexibilidad para navegar y administrar el metastore de Unity Catalog.

## Limpieza

Cuando termine de explorar Unity Catalog, puede eliminar los recursos que ha creado para evitar costes innecesarios de Azure.

1. Cierra la pestaña del explorador y vuelve a la pestaña de Azure Portal.
2. En Azure Portal, en la página **Inicio**, seleccione **Grupos de recursos**.
3. Selecciona el grupo de recursos que contiene el área de trabajo de Azure Databricks (no el grupo de recursos administrado).
4. En la parte superior de la página **Información general** del grupo de recursos, seleccione **Eliminar grupo de recursos**. 
5. Escriba el nombre del grupo de recursos para confirmar que quiere eliminarlo y seleccione **Eliminar**.

    Después de unos minutos, tu grupo de recursos y el grupo de recursos del área de trabajo administrada asociada a él se eliminarán.
