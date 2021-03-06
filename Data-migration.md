When a developer makes changes in the business application that affect the tables and columns in the database (renaming an entity, for example), there is often a need to migrate the data from the old data structure to the new one.

This article describes how to develop and deploy data migration scripts that are used to migrate the data when upgrading your Rhetos application.

Table of contents:

1. [Developing data-migration scripts](#developing-data-migration-scripts)
   1. [SQL scripts](#sql-scripts)
   2. [Formatting](#formatting)
   3. [Rules for writing data-migration scripts](#rules-for-writing-data-migration-scripts)
2. [Examples](#examples)
   1. [Moving a property from one entity to another](#moving-a-property-from-one-entity-to-another)
   2. [Initializing a new unique property value](#initializing-a-new-unique-property-value)
   3. [Changing a property's type](#changing-a-propertys-type)
3. [Advanced Topics](#advanced-topics)
   1. [Deploying migration scripts](#deploying-migration-scripts)
   2. [Automatic use of the migration tables when dropping and creating columns](#automatic-use-of-the-migration-tables-when-dropping-and-creating-columns)
   3. [Database structure independence](#database-structure-independence)
   4. [Cleanup](#cleanup)

## Developing data-migration scripts

### SQL scripts

Data-migration scripts are regular SQL scripts, with the code following specific rules as describes in this article.
The scripts are placed in a DataMigration subfolder in the Rhetos package.
When deploying the Rhetos package, the scripts will be executed in the alphabetical order (by folder name, than file name), skipping the scripts that were already executed in previous deployments.

The recommended naming conception:

* The data-migration scripts inside DataMigration folder should be grouped in subfolders by the package release version.
* Each script in the subfolder should have numbered prefix (starting with "1 "), to ensure the correct order when executing the scripts.

For example:

    DataMigration\1.0\1 - Insert master.sql
    DataMigration\1.0\2 - Insert detail.sql
    DataMigration\2.0\1 - Update detail codes.sql

### Formatting

A data-migration script must be written in a specific format, see the example below.
It is recommended to use the stored procedure `Rhetos.HelpDataMigration` to **generate the data-migration script**.

In the following example, the data-migration script modifies the data in the column `Name` of the table `Common.Principal`, removing the excess spaces in the principal's name:

```SQL
/*DATAMIGRATION 2B676531-76AB-43F3-AE07-868434DEAB7F*/ -- Change the script's code only if it needs to be executed again.

-- The following lines are generated by: EXEC Rhetos.HelpDataMigration 'Common', 'Principal';
EXEC Rhetos.DataMigrationUse 'Common', 'Principal', 'ID', 'uniqueidentifier';
EXEC Rhetos.DataMigrationUse 'Common', 'Principal', 'Name', 'nvarchar(256)';
GO

UPDATE
    _Common.Principal
SET
    Name = LTRIM(Name)
WHERE
    Name <> LTRIM(Name);

EXEC Rhetos.DataMigrationApplyMultiple 'Common', 'Principal', 'ID, Name';
```

### Rules for writing data-migration scripts

1. Each script must have a unique tag at the beginning (GUID is recommended).
   Based on that tag, Rhetos monitors which scripts are already executed.
   After the deployment, the script's that should not be edited, otherwise the script might be executed again on the same database.
2. The script may only read and write data from the migration tables; it should never directly access the original tables.
   The migration tables are created by `DataMigrationUse` procedure, executed at the beginning of the script.
   They have same name as the original tables, but are placed in the database schema with "_" prefix (underscore).
3. The `DataMigrationUse` procedure will:
    * Create a migration table with the selected columns.
    * Copy the data from the original table.
    * If the column already exists, but has a different data type, it will convert the migrated data type if the
      implicit conversion is supported by the SQL server (see ALTER TABLE ALTER COLUMN).
4. The `DataMigrationApplyMultiple` procedure will:
    * Copy the data from the migration table to the original table.
    * If there were inserted or deleted records in the migration table, the procedure will also insert/delete the records in the original table.
    * If the original table does not exist yet (will be created in the next version), the procedure will not
      try to copy the data. The data will be copied automatically by Rhetos when the original table is created.

These rules allow the script to be executed **independently of the current database structure**,
so that a script can be executed either before or after a specific version is deployed.
This allows Rhetos to execute the scripts in different situations:

1. when upgrading an existing application/database to a new version
2. when deploying the application for the first time to an empty database
3. after the new version if deployed to development environment (when developing the migration script)

Note that after generating the data-migration script by executing `Rhetos.HelpDataMigration`,
you might need to **manually adjust** the *DataMigrationUse* and *DataMigrationApplyMultiple* parts of this script:

* You might need to add new lines to the script with *DataMigrationUse*,
  if some columns do not currently exist in your database (not yet deployed),
  but you need to use them the script.
  Make sure to specify the correct [column type](Data-structure-properties).
* You may remove *DataMigrationUse* for columns that are not needed in this script. Keep the ID column.
* You may remove columns from *DataMigrationApplyMultiple* if not modifying the data in those columns. Keep the ID column.

## Examples

### Moving a property from one entity to another

Example task:

* There are entities `E1` and `E2` in the module `Demo`.
  `E2` extends `E1`, so they have the same ID values.
* In the old version of the application, the entity `E1` has the property `P`.
* We have developed a new version, where the property `P` is moved from the entity `E1` and to `E2`.
* We need to migrate the data from `E1` to `E2`.

Solution:

```SQL
/*DATAMIGRATION 3CCB28D7-F257-4EDE-9A8E-6B9A7991241B*/ -- Change the script's code only if it needs to be executed again.

-- The following lines are generated by: EXEC Rhetos.HelpDataMigration 'Demo', 'E1';
EXEC Rhetos.DataMigrationUse 'Demo', 'E1', 'ID', 'uniqueidentifier';
EXEC Rhetos.DataMigrationUse 'Demo', 'E1', 'P', 'nvarchar(256)';
-- The following lines were generated by: EXEC Rhetos.HelpDataMigration 'Demo', 'E2';
EXEC Rhetos.DataMigrationUse 'Demo', 'E2', 'ID', 'uniqueidentifier';
EXEC Rhetos.DataMigrationUse 'Demo', 'E2', 'P', 'nvarchar(256)';
GO

-- Don't forget to use the underscore '_' in the schema name.
UPDATE
    e2
SET
    P = e1.P
FROM
    _Demo.E2 e2
    INNER JOIN _Demo.E1 e1 ON e1.ID = e2.ID
WHERE
    e2.P IS NULL; -- Safeguard if script is executed multiple times.

EXEC Rhetos.DataMigrationApplyMultiple 'Demo', 'E2', 'ID, P';
```

How to write this script:

1. Usually you will first execute *HelpDataMigration* for both entities E1 and E2,
   to generate the overall script structure and copy *DataMigrationUse* property preparation lines.
2. You will need to manually write a new line with *DataMigrationUse* for
   column 'P' on entity 'E1' or 'E2', depending on whether you executed
   *HelpDataMigration* before or after deploying modifications to the database.
3. Keep only the *DataMigrationApplyMultiple* for E2, because the script does not modify the data in E1.

### Initializing a new unique property value

Data initialization is obligatory if the new property is unique,
to avoid the unique constraint database error for multiple null values
if any records might exist in the table.

The following script initializes the new column "Code" with the generated numeric codes.
This is an example from the [Bookstore](https://github.com/Rhetos/Bookstore) demo application.

```SQL
/*DATAMIGRATION DC31DB21-8E87-49F9-A334-E7EB246DBD53*/ -- Change the script's code only if it needs to be executed again.

-- The following lines are generated by: EXEC Rhetos.HelpDataMigration 'Bookstore', 'Topic';
EXEC Rhetos.DataMigrationUse 'Bookstore', 'Topic', 'ID', 'uniqueidentifier';
EXEC Rhetos.DataMigrationUse 'Bookstore', 'Topic', 'Code', 'nvarchar(256)';
EXEC Rhetos.DataMigrationUse 'Bookstore', 'Topic', 'Name', 'nvarchar(256)';
GO

SELECT
    t.ID,
    NewCode = CAST(ROW_NUMBER() OVER (ORDER BY t.Name, t.ID) AS NVARCHAR(10))
INTO
    #codes
FROM
    _Bookstore.Topic t
WHERE
    t.Code IS NULL; -- Make sure to ignore already initialized values, if this script is executed multiple times.

UPDATE
    t
SET
    Code = c.NewCode
FROM
    _Bookstore.Topic t
    INNER JOIN #codes c ON c.ID = t.ID

EXEC Rhetos.DataMigrationApplyMultiple 'Bookstore', 'Topic', 'ID, Code';
```

### Changing a property's type

If a property's type is changed, deployment of the new version will to the following

* First, the old column will be removed from the table, automatically keeping the backup in the migration table.
* New property will be created and the data will be restored from the migration table.
* When restoring the data, SQL Server will try to automatically convert the data type.
* If SQL Server does not support implicit conversion of the these type, the developer should create
  a data-migration script that modifies the data or the column type in the migration table.

## Advanced Topics

### Deploying migration scripts

The `DeployPackages.exe` utility will automatically execute the data-migration scripts.

* `DeployPackages.exe` executes the data-migration scripts before upgrading the database structure.
* If `DeployPackages.exe` fails while upgrading the database structure, it will execute those data-migration scripts
  again on next deployment. This means that the migration scripts should be written in such way that allows a script
  to be executed multiple times without negative consequences.
* If a Rhetos package depends on another package (dependencies are defined in .nuspec),
  migration scripts from the other package will be executed first.
* DateMigration scripts within a package are executed ordered by folder name, then by file name. [Natural sort](https://en.wikipedia.org/wiki/Natural_sort_order) is used to order the scripts.
* The previously executed scripts are logged in `Rhetos.DataMigrationScript` table.
* Rhetos will check the tag from the script's header to verify if the script is already executed.
  This allows developers to later reorganize and rename the old data-migration scripts without executing them again.

### Automatic use of the migration tables when dropping and creating columns

* If a property, an entity or a module is removed in a new version of the application being deployed,
  Rhetos will keep a backup of its data in the migration tables.
* If the property/entity/module is brought back later, Rhetos will automatically restore the data
  from the corresponding data migration table.
* During deployment, Rhetos keeps track of the data that is copied from the original tables to the migration tables.
  If multiple migration scripts execute the `Rhetos.DataMigrationUse` procedure for a same table, the data will
  be copied only on a fist call.
  This ensures that later script will not overwrite data modification of a previous script.

### Database structure independence

The data migration scripts are independent of the current database structure,
so the developers don't need to couple the database structure versioning with the data migration scripts.

In the following examples, the same data-migration script is executed in different database versions (before or after upgrading the database), resulting with the identical data at the end. The steps that follow the main data flow are marked bold.

The examples show the effect of a data-migration script that copies the data from entity A to B, when the entity in module "M" is renamed from "A" to "B".

Option A) Deployment before the data migration:

1. DeployPackages.exe - Update database
    1. **Drop M.A: backup to _M.A**
    2. Create M.B
2. Manual execution of the data-migration script
    1. DataMigrationUse M.A: M.A => _M.A (nothing to do, A does not exits, _A already contains old data)
    2. DataMigrationUse M.B: M.B => _M.B (B is empty)
    3. **SQL update query: _M.A => _M.B**
    4. **DataMigrationApply M.B: _M.B => M.B**

Option B) Data migration before the deployment:

1. DeployPackages.exe - Data migration
    1. **DataMigrationUse M.A: M.A => _M.A**
    2. DataMigrationUse M.B: M.B => _M.B (nothing to do, B does not exist)
    3. **Update: _M.A => _M.B**
    4. DataMigrationApply M.B: _M.B => M.B (nothing to do, B does not exist)
2. DeployPackages.exe - Update database
    1. Drop M.A
    2. **Create M.B: restore from _B**

### Cleanup

* After upgrading the database, `DeployPackages.exe` will delete the migration columns and tables
  that are copied to the original tables.
* The migration table and columns that don't exist in the new version of the application will be kept as a backup.
* You can optionally delete the backup data by running `CleanupOldData.exe`;
  it will delete everything in database schemas that start with an underscore ("_").
  This is not recommended if the last deployment failed, because it could result with a data loss.
