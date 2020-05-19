---
description: A Description of how this thing will change your life.
---

# What is SnowConvert?

SnowConvert is a software that understands [Teradata SQL](https://www.teradata.com/) and converts it to:

* Teradata SQL to [Snowflake SQL](https://www.snowflake.com/)
* Teradata Stored Procedures to JavaScript
* Teradata BTEQ to Python

#### Teradata SQL to Snowflake SQL

SnowConvert understands the source code and converts Teradata's Data Definition Language \(DDL\), Data Manipulation Language \(DML\) and functions to Snowflakes corresponding SQL, except the Procedures and BTEQ.

**Example of Teradata SQL to Snowflake SQL**:

Here's an example of the conversion of a simple CREATE TABLE statement 

_The source code \(a simple CREATE TABLE\):_

```sql
-- CREATE TABLE DDL
CREATE SET TABLE TABLE1
    NO BEFORE JOURNAL,
    NO AFTER JOURNAL,
    CHECKSUM = DEFAULT,
    DEFAULT MERGEBLOCKRATIO
(
    COL1 VARCHAR(15) CHARACTER SET LATIN NOT CASESPECIFIC,
    Col2 BYTEINT CHECK ( CurrentFlag  IN (0 ,1 ) ) NOT NULL,
    COL3 DATE FORMAT 'yyyy-mm-dd',
    COL4 BLOB(2097088000),
    COL5 BYTEINT,
    COL7 INTEGER NOT NULL COMPRESS (1 ,2 ,3 ,4),
    COL8 INTERVAL HOUR(2) TO MINUTE
);

-- REPLACE VIEW DDL
REPLACE VIEW VIEW1 AS
SELECT * FROM TABLE1
UNION ALL
SELECT MAX(COL1) FROM TABLE1;
```

_The Converted Snowflake SQL Code_:

```sql
-- Snowflake converted SQL
CREATE TABLE PUBLIC. TABLE1
(
    COL1 VARCHAR(15) COLLATE 'en-ci',
    Col2 BYTEINT NOT NULL,
    COL3 DATE,
    COL4 BINARY /**** WARNING: Column converted from BLOB data type ****/,
    COL5 BYTEINT,
    COL7 INTEGER NOT NULL,
    COL8 VARCHAR (20) COMMENT 'INTERVAL HOUR(2) TO MINUTE' /**** WARNING: INTERVAL DATA TYPE "INTERVAL HOUR(2) TO MINUTE" CONVERTED TO VARCHAR ****/
) ;

CREATE OR REPLACE VIEW PUBLIC. VIEW1
AS
    SELECT * FROM PUBLIC. TABLE1
    UNION ALL
    SELECT MAX(COL1) FROM PUBLIC. TABLE1 ;
```

In this converted sql you will notice that we are converting many things such as:

* Adding `PUBLIC` Schema by default for all the Table and view names if the user doesn't specify one.
* `CREATE SET TABLE` to `CREATE TABLE`
* `REPLACE VIEW` to `CREATE OR REPLACE VIEW`
* Data Types: `BLOB` to `BINARY` and `INTERVAL` to `VARCHAR`
* Data Type Attributes: `NOT CASESPECIFIC` to `COLLATE`
* Removing unnecessary parts like `NO BEFORE JOURNAL`, `NO AFTER JOURNAL`, `CHECKSUM`, `COMPRESS`, `CHECK`

#### Teradata Stored Procedures to JavaScript

SnowConverts undertands the source code and converts the Teradata's _CREATE PROCEDURE_ and _REPLACE PROCEDURE_ Data Definition Language to Snowflake's _CREATE OR REPLACE PROCEDURE_ Definition SQL and its inner statements to JavaScript.

**Example of a Procedure**:

_The source code:_

```sql
-- Source Code
REPLACE PROCEDURE Procedure1()               
    DYNAMIC RESULT SETS 2
    BEGIN    
         DECLARE SQL_CMD,SQL_CMD_1  VARCHAR(20000) DEFAULT ' '; 
         DECLARE RESULTSET CURSOR WITH RETURN ONLY FOR FIRSTSTATEMENT;
         SET SQL_CMD='SEL * FROM SNOWCONVERT.EMPLOYEE';
        PREPARE FIRSTSTATEMENT FROM SQL_CMD; 
        OPEN RESULTSET; 
    END;
```

_Converted Snowflake SQL and Javascript Code_:

```javascript
// Converted code
CREATE OR REPLACE PROCEDURE PUBLIC. Procedure1 ()
   RETURNS STRING
   LANGUAGE JAVASCRIPT
   EXECUTE AS CALLER
   AS
   $$
     var resultSetCounter = 0;
    var tablelist = new Array;
    var SQL_CMD = ` `;
    var SQL_CMD_1 = ` `;
    var procname = `.PUBLIC.Procedure1_`;
    var sql_command = `select current_session() || '_' || to_varchar(current_timestamp, 'yyyymmddhh24missss')`;
    var stmt = snowflake.createStatement({
       sqlText : sql_command
    });
    var res = stmt.execute();
    res.next();
    var sessionid = procname + res.getColumnValue(1);
    SQL_CMD = `SELECT * FROM SNOWCONVERT.PUBLIC.EMPLOYEE`;
    var setname = SQL_CMD;
    var tablename = sessionid + `_` + resultSetCounter++;
    var sql_stmt = `CREATE TEMPORARY TABLE ${tablename} AS ${setname}`;
    tablelist.push(tablename);
    var RESULTSET = snowflake.createStatement({
       sqlText : sql_stmt
    }).execute();
    return tablelist;
$$;
```

In this converted sql you will notice that we are converting many things such as:

* `REPLACE PROCEDURE` to `CREATE OR REPLACE PROCEDURE`
* Local Declaration statement like `DECLARE SQL_CMD` to `var SQL_CMD = '  '`
* Assignment statement like `SET SQL_CMD = ''`  to `SQL_CMD = ''`
* The combination of `DECLARE CURSOR WITH RETURN` with `PREPARE` will be converted to:

  ```text
  var setname = SQL_CMD;
    var tablename = sessionid + `_` + resultSetCounter++;
    var sql_stmt = `CREATE TEMPORARY TABLE ${tablename} AS ${setname}`;
    tablelist.push(tablename);
  ```

* Open statement like `OPEN RESULTSET` to:

  ```text
  var RESULTSET = snowflake.createStatement({
       sqlText : sql_stmt
    }).execute();
  ```

#### BTEQ scripts converted to Python

All _BTEQ_ script files will be converted to Python script. And a helper is copied to the output folder.

_The source code:_

```sql
.LABEL CREATE_TMP_KS
CREATE SET TABLE TABLE1, NO FALLBACK , DEFAULT MERGEBLOCKRATIO
(
    COL1 TIMESTAMP(0)
);
.IF ERRORCODE <> 0 THEN .QUIT ERRORCODE;
```

_Converted Python Code_:

```python
import sys
import snowconvert_helpers
con = None
try:
   snowconvert_helpers.configure_log()
   con = snowconvert_helpers.log_on(sys.argv[1], sys.argv[2], sys.argv[3])
   snowconvert_helpers.execute_sql_statement("""CREATE TABLE PUBLIC. TABLE1
(
    COL1 TIMESTAMP
)""", con)

   if snowconvert_helpers.error_code != 0:
      snowconvert_helpers.quit_application(snowconvert_helpers.error_code)

except Exception as e:
   print(e)

finally:
   if con is not None:
      con.close()
   snowconvert_helpers.quit_application()
```

In this converted sql you will notice that we are converting many things such as:

* A new python code with the following template:

  \`\`\`python import sys import snowconvert\_helpers con = None try: snowconvert\_helpers.configure\_log\(\) con = snowconvert\_helpers.log\_on\(sys.argv\[1\], sys.argv\[2\], sys.argv\[3\]\)

  **Any BTEQ CODE will be located here**

except Exception as e: print\(e\)

finally: if con is not None: con.close\(\) snowconvert\_helpers.quit\_application\(\)

\`\`\`

* Removed `.Label` statement
* `CREATE TABLE` to a function call `snowconvert_helpers.execute_sql_statement('CREATE TABLE...')`
* `.IF` statement to `if` in python

### System requirements

* Windows 10
* .NET Runtime 4.6.2 or greater \(available by default starting from Windows 10 - 1607\)
* 4GB of RAM \(recommended\)

### SnowConvert terminology

* SQL: is a standard language for storing, manipulating and retrieving data in databases.
* BTEQ: Batch Teradata Query. BTEQ was the first utility and query tool for Teradata.
* _SnowConvert_: a software that converts securely and automatically your Teradata files to the Snowflake cloud data warehouse. 
* _Conversion rule or transformation rule_: statu that allow the SnowConvert to convert from a portion of source code to the expected code.  
* _Parse_: parse or parsing is an initial process of SnowConvert where it tries to understand the source code, builds up an internal data structure to be able to process the conversion rules. 

