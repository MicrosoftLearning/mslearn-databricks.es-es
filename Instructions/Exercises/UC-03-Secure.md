---
lab:
  title: Protección de datos en Unity Catalog
---

# Protección de datos en Unity Catalog

La seguridad de los datos es una preocupación fundamental para las organizaciones que administran información confidencial en su almacén de lago de datos. A medida que los equipos de datos colaboran entre distintos roles y departamentos, garantizar que las personas adecuadas tengan acceso a los datos correctos —mientras se protege la información sensible del acceso no autorizado— se vuelve cada vez más complejo.

En este laboratorio práctico se mostrarán dos eficaces características de seguridad de Unity Catalog que van más allá del control de acceso básico:

1. **Filtrado de filas y enmascaramiento de columnas**: descubra cómo proteger los datos confidenciales a nivel de tabla ocultando filas específicas o enmascarando valores de columna en función de los permisos de usuario.
2. **Vistas dinámicas**: cree vistas inteligentes que ajusten automáticamente qué datos pueden ver los usuarios en función de sus pertenencias a grupos y niveles de acceso

Este laboratorio tardará aproximadamente **45** minutos en completarse.

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
7. Espera a que se complete el script: normalmente tarda unos 5 minutos, pero en algunos casos puede tardar más. Mientras espera, revise el módulo [Implementación de la seguridad y el control de acceso en Unity Catalog](https://learn.microsoft.com/training/modules/implement-security-unity-catalog) en Microsoft Learn.

## Creación de un catálogo

1. Inicie sesión en un área de trabajo vinculada al metastore.
2. Seleccione **Catálogo** en el menú de la izquierda.
3. Seleccione **Catálogos** debajo de **Acceso rápido**.
4. Seleccione **Crear catálogo**.
5. En el cuadro de diálogo **Crear un nuevo catálogo**, escriba un **nombre de catálogo** y seleccione el **tipo** de catálogo que desea crear: **Catálogo** estándar.
6. Especifique una **ubicación de almacenamiento** administrada.

## Crear un cuaderno

1. En la barra lateral, usa el vínculo **(+) Nuevo** para crear un **cuaderno**.
   
1. Cambie el nombre predeterminado del cuaderno (**Cuaderno sin título *[fecha]***) a `Secure data in Unity Catalog` y, en la lista desplegable **Conectar**, seleccione el **clúster sin servidor** si aún no está seleccionado.  Tenga en cuenta que **Sin servidor** está habilitado de forma predeterminada.

1. Copie y ejecute el código siguiente en una nueva celda del cuaderno para configurar el entorno de trabajo para este curso. También establecerá el catálogo predeterminado en su catálogo específico y el esquema en el nombre de esquema que se muestra a continuación mediante las instrucciones `USE`.

```SQL
USE CATALOG `<your catalog>`;
USE SCHEMA `default`;
```

1. Ejecute el código siguiente y confirme que el catálogo actual está establecido en el nombre de catálogo único y que el esquema actual es **predeterminado**.

```sql
SELECT current_catalog(), current_schema()
```

## Protección de columnas y filas con enmascaramiento de columnas y filtrado de filas

### Creación de la tabla Customers

1. Ejecute el código siguiente para crear la tabla **customers** en el esquema **predeterminado**.

```sql
CREATE OR REPLACE TABLE customers AS
SELECT *
FROM samples.tpch.customer;
```

2. Ejecute una consulta para ver *10* filas de la tabla **customers** en el esquema **predeterminado**. Observe que la tabla contiene información como **c_name**, **c_phone** y **c_mktsegment**.
   
```sql
SELECT *
FROM customers  
LIMIT 10;
```

### Creación de una función para realizar el enmascaramiento de columnas

Consulte la documentación [Filtrado de datos confidenciales de la tabla mediante filtros de fila y máscaras de columna](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/filters-and-masks/) para obtener ayuda adicional.

1. Cree una función denominada **phone_mask** que tache la columna **c_phone** de la tabla **customers** si el usuario no es miembro del grupo ("admin") mediante la función `is_account_group_member`. La función **phone_mask** debe devolver la cadena *REDACTED PHONE NUMBER* si el usuario no es miembro.

```sql
CREATE OR REPLACE FUNCTION phone_mask(c_phone STRING)
  RETURN CASE WHEN is_account_group_member('metastore_admins') 
    THEN c_phone 
    ELSE 'REDACTED PHONE NUMBER' 
  END;
```

2. Aplique la función de enmascaramiento de columnas **phone_mask** a la tabla **customers** mediante la instrucción `ALTER TABLE`.

```sql
ALTER TABLE customers 
  ALTER COLUMN c_phone 
  SET MASK phone_mask;
```

3. Ejecute la consulta siguiente para ver la tabla **customers** con la máscara de columna aplicada. Confirme que la columna **c_phone** muestra el valor *REDACTED PHONE NUMBER*.

```sql
SELECT *
FROM customers
LIMIT 10;
```

### Creación de una función para realizar el filtrado de filas

Consulte la documentación [Filtrado de datos confidenciales de la tabla mediante filtros de fila y máscaras de columna](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/filters-and-masks/) para obtener ayuda adicional.

1. Ejecute el código siguiente para contar el número total de filas de la tabla **customers**. Confirme que la tabla contiene 750 000 filas de datos.

```sql
SELECT count(*) AS TotalRows
FROM customers;
```

2. Cree una función denominada **nation_filter** que filtre por **c_nationkey** en la tabla **customers** si el usuario no es miembro del grupo ("admin") mediante la función `is_account_group_member`. La función solo debe devolver filas en las que **c_nationkey** sea igual a *21*.

    Vea la documentación de la [función if](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/functions/if) para obtener ayuda adicional.

```sql
CREATE OR REPLACE FUNCTION nation_filter(c_nationkey INT)
  RETURN IF(is_account_group_member('admin'), true, c_nationkey = 21);
```

3. Aplique la función de filtrado de filas `nation_filter` a la tabla **customers** mediante la instrucción `ALTER TABLE`.

```sql
ALTER TABLE customers 
SET ROW FILTER nation_filter ON (c_nationkey);
```

4. Ejecute la consulta siguiente para contar el número de filas de la tabla **customers**, ya que ha filtrado las filas para los usuarios que no son administradores. Confirme que solo puede ver *29 859* filas (*donde c_nationkey = 21*). 

```sql
SELECT count(*) AS TotalRows
FROM customers;
```

5. Ejecute la consulta siguiente para ver la tabla **customers**. 

    Confirmar la tabla final:
    - tacha la columna **c_phone** y
    - filtra las filas en función de la columna **c_nationkey** para los usuarios que no son *administradores*.

```sql
SELECT *
FROM customers;
```

## Protección de columnas y filas con vistas dinámicas

### Creación de la tabla Customers_new

1. Ejecute el código siguiente para crear la tabla **customers_new** en el esquema **predeterminado**.

```sql
CREATE OR REPLACE TABLE customers_new AS
SELECT *
FROM samples.tpch.customer;
```

2. Ejecute una consulta para ver *10* filas de la tabla **customers_new** en el esquema **predeterminado**. Observe que la tabla contiene información como **c_name**, **c_phone** y **c_mktsegment**.

```sql
SELECT *
FROM customers_new
LIMIT 10;
```

### Creación de la vista dinámica

Vamos a crear una vista denominada **vw_customers** que presenta una vista procesada de los datos de la tabla de **customers_new** con las transformaciones siguientes:

- Seleccione todas las columnas de la tabla **customers_new**.

- Tache todos los valores de la columna **c_phone** con *REDACTED PHONE NUMBER* a menos que esté en `is_account_group_member('admins')`.
    - SUGERENCIA: Use una instrucción `CASE WHEN` en la cláusula `SELECT`.

- Restrinja las filas en las que **c_nationkey** sea igual a *21* a menos que esté en `is_account_group_member('admins')`.
    - SUGERENCIA: Use una instrucción `CASE WHEN` en la cláusula `WHERE`.

```sql
-- Create a movies_gold view by redacting the "votes" column and restricting the movies with a rating below 6.0

CREATE OR REPLACE VIEW vw_customers AS
SELECT 
  c_custkey, 
  c_name, 
  c_address, 
  c_nationkey,
  CASE 
    WHEN is_account_group_member('admins') THEN c_phone
    ELSE 'REDACTED PHONE NUMBER'
  END as c_phone,
  c_acctbal, 
  c_mktsegment, 
  c_comment
FROM customers_new
WHERE
  CASE WHEN
    is_account_group_member('admins') THEN TRUE
    ELSE c_nationkey = 21
  END;
```

3. Muestre los datos en la vista **vw_customers**. Confirme que la columna **c_phone** está tachada. Confirme que **c_nationkey** es igual a *21* a menos que sea el administrador.

```sql
SELECT * 
FROM vw_customers;
```

6. Cuente el número de filas de la vista **vw_customers**. Confirme que la vista contiene *29 859* filas.

```sql
SELECT count(*)
FROM vw_customers;
```

### Emisión de la vista de concesión de acceso

1. Vamos a emitir una concesión para "usuarios de cuenta" para que puedan ver la vista **vw_customers**.

**NOTA:** También deberá proporcionar a los usuarios acceso al catálogo y al esquema. En este entorno de entrenamiento compartido, no puede conceder acceso al catálogo a otros usuarios.

```sql
GRANT SELECT ON VIEW vw_customers TO `account users`
```

2. Use la instrucción `SHOW` para mostrar todos los privilegios (heredados, denegados y concedidos) que afectan a la vista **vw_customers**. Confirme que la columna **Principal** contiene *usuarios de cuenta*.

    Consulte la documentación de [SHOW GRANTS](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/security-show-grant) para obtener ayuda.

```sql
SHOW GRANTS ON vw_customers
```