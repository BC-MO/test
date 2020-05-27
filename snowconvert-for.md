# What is SnowConvert?

SnowConvert is a software that understands [Teradata SQL](https://www.teradata.com/) and performs the following conversions start writing things:

* Teradata SQL to [Snowflake SQL](https://www.snowflake.com/)
* Teradata Stored Procedures to JavaScript
* Teradata BTEQ to Python

Before we get lost in the magic of these code conversions, here a few terms/definitions so you know what we mean when we start dropping them all over the documentation:

* _SQL \(Structured Query Language\):_ the standard language for storing, manipulating, and retrieving data in most modern database architectures.
* _BTEQ \(Batch Teradata Query\):_ BTEQ was the first utility and query tool for Teradata.
* _SnowConvert_: the software that converts securely and automatically your Teradata files to the Snowflake cloud data platform. 
* _Conversion rule or transformation rule:_ rules that allow SnowConvert to convert from a portion of source code to the expected target code.  
* _Parse:_ parse or parsing is an initial process done by SnowConvert to understand the source code, and build up an internal data structure to process the conversion rules. 

Let's dive in to the code conversions that Mobilize.Net SnowConvert can perform. 

## Code Conversions

### Teradata SQL to Snowflake SQL

SnowConvert understands the Teradata source code and converts the Data Definition Language \(DDL\), Data Manipulation Language \(DML\) and functions in the source code to the corresponding SQL in the target: Snowflake. \([Procedures](https://bcarver.gitbook.io/snowconvert-documentation/snowconvert-for/~/settings/advanced#teradata-stored-procedures-to-javascript) and [BTEQ](https://bcarver.gitbook.io/snowconvert-documentation/snowconvert-for/~/settings/advanced#bteq-scripts-converted-to-python) are handled differently.\)

#### **Example of Teradata SQL to Snowflake SQL**

Here's an example of the conversion of a simple CREATE TABLE statement. 

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

In this converted SQL you will notice that we are converting many things such as:

* Adding `PUBLIC` Schema by default for all the Table and view names if the user doesn't specify one \(see how to specify a Schema\).
* `CREATE SET TABLE` to `CREATE TABLE`
* `REPLACE VIEW` to `CREATE OR REPLACE VIEW`
* Data Types: `BLOB` to `BINARY` and `INTERVAL` to `VARCHAR`
* Data Type Attributes: `NOT CASESPECIFIC` to `COLLATE`
* Removing pieces of the Teradata SQL that are not necessary in Snowflake due to Snowflake's architecture such as `NO BEFORE JOURNAL`, `NO AFTER JOURNAL`, `CHECKSUM`, `COMPRESS`, and `DEFAULT MERGEBLOCKRATIO`.

### Teradata Stored Procedures to JavaScript

SnowConvert takes Teradata stored procedures \(usually written in SQL\) and converts them to JavaScript embedded into Snowflake SQL. Teradata's CREATE PROCEDURE and REPLACE PROCEDURE language is replaced by Snowflake's CREATE OR REPLACE PROCEDURE language. JavaScript is called as a scripting language, and all of the inner statements are converted to JavaScript.

#### **Example of a Stored Procedure Conversion**

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

In this converted SQL, there are several conversions that take place:

* `REPLACE PROCEDURE` to `CREATE OR REPLACE PROCEDURE`
* Local declaration statement `DECLARE SQL_CMD` to `var SQL_CMD = '  '`
* Assignment statement `SET SQL_CMD = ''`  to `SQL_CMD = ''`
* The combination of `DECLARE CURSOR WITH RETURN` with `PREPARE` are converted to:

  ```text
  var setname = SQL_CMD;
    var tablename = sessionid + `_` + resultSetCounter++;
    var sql_stmt = `CREATE TEMPORARY TABLE ${tablename} AS ${setname}`;
    tablelist.push(tablename);
  ```

* The `OPEN` statement `OPEN RESULTSET` is converted to:

  ```text
  var RESULTSET = snowflake.createStatement({
       sqlText : sql_stmt
    }).execute();
  ```

### BTEQ scripts converted to Python

Basic Teradata Query \(BTEQ\) is Teradata's proprietary scripting language. All BTEQ script files will be converted to Python scripts. A helper class will be created by SnowConvert and copied to the output folder to create the functional equivalence between the source and the target. BTEQ can be batch run from outside the Snowflake environment. Learn more about how [you can connect Python scripts directly to Snowflake](https://docs.snowflake.com/en/user-guide/python-connector.html). 

BTEQ files are also the foundation for multiple other proprietary data types that Teradata has created:

* Fastload
* Multiload
* TPUMP

Each of these filetypes are extensions of BTEQ. SnowConvert converts each of these filetypes to Python. Here's an example of a BTEQ to Python conversion:

#### Example of a BTEQ conversion

_The BTEQ source code:_

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

In the converted output code in Python, note that the SQL statement and error handler are converted, but there are also setup and helper functions created to facilitate the code running in a Python connection to Snowflake. A few of the conversions are: 

* Removed `.Label` statement
* `CREATE TABLE` to a function call `snowconvert_helpers.execute_sql_statement('CREATE TABLE...')`
* `.IF` statement to `if` in python

The procedural conversions include: 

`REPLACE PROCEDURE` to `CREATE OR REPLACE PROCEDURE`

* importing helper classes such as `sys` and `snowconvert_helpers`, which is created by SnowConvert 
* Creating a connector variable `con = None`, and populating it in the `try` statement: `con = snowconvert_helpers.log_on(sys.argv[1], sys.argv[2], sys.argv[3])`.
* Setting up a log: `snowconvert_helpers.configure_log()`.
* All of the SnowConvert created Python files will end with the following `finally` syntax: 

  ```python
  finally:
     if con is not None:
        con.close()
     snowconvert_helpers.quit_application()
  ```

* Local declaration statement `DECLARE SQL_CMD` to `var SQL_CMD = '  '`

And that's it! Mobilize.Net SnowConvert takes the pain and frustration out of changing data platforms. Learn how to download and get access to SnowConvert on the next page.

