---
description: >-
  Consider this page a walkthrough that will guide you through some highs and
  lows of performing a migration from Teradata to Snowflake. Enjoy.
---

# Migrating from Teradata to Snowflake

This document describes the main Teradata SQL statement transformations, and gives an overview of transformations in Teradata Scripting languages such as BTEQ, FastLoad, and MultiLoad. Use this as a guide to how the transformed code might look when migrating from [Teradata](https://docs.teradata.com/) to [Snowflake](https://docs.snowflake.net/manuals/index.html). SQL has a similar syntax between dialects, but each dialect can extend or add new functionalities. For this reason, when running SQL in one environment \(such as Teradata\) vs. another \(such as Snowflake\), there are many statements that require transformation or even removal. These transformations are done by Mobilize.net using the SnowConvert tool.

## 1. SQL transformations

### 1.1 Transforming SQL DDL statements

This section illustrates Data Definition Language \(DDL\) syntax transformations from Teradata to Snowflake.

#### 1.1.1 Create table transformation

> See [Create table](https://docs.teradata.com/reader/scPHvjfglIlB8F70YliLAw/t9ZHBbmVpK7GrnocHmVG1Q)

**Teradata**

```sql
Create table table1, no fallback,
no before journal,
no after journal (
    c1 integer not null,
    f1 integer not null,
    p1 integer not null,
   DATE,
   TIME,
    foreign key (f1) references with check option table2 (d1)
)
unique primary index(c1)
partition by column (p1);
```

**Snowflake**

```sql
create table my_dw.public.table1 (
c1 integer not null,
f1 integer not null,
p1 integer not null,
CURRENT_DATE,
CURRENT_TIME,
unique (c1)
);

alter table my_dw.public.table1 add constraint f1
foreign key (f1) references my_dw.public.table2 (d1) ;
/**** warning: performance review - cluster by ****/
/*cluster by(p1)*/;
```

> * _Notes:_
>   * _As shown in the example above, Snowflake does not support Teradata create table options. They are removed._
>   * _The `partition by` statement is transformed to a `cluster by`, but it is commented out due to performance considerations._
>   * _In Teradata, the primary index constraint is declared outside of the `create table` statement, but in Snowflake it is required to be inside._
>   * _In Teradata, the foreign key constraint can be declared inside the `create table` statement, but in Snowflake it is required to be declared_ 
>
>     _as an `alter table` statement. Also notice that the `with check option` statement is removed._
>
>   * _Functions such as `DATE` and `TIME` are mapped to their respective Snowflake equivalent._

* [Create table](https://docs.teradata.com/reader/scPHvjfglIlB8F70YliLAw/t9ZHBbmVpK7GrnocHmVG1Q) with data option

**Teradata**

```sql
Create table table1 as table2 with data
```

**Snowflake**

```sql
Create table table1 (select * from table2)
```

* [Create table](https://docs.teradata.com/reader/scPHvjfglIlB8F70YliLAw/t9ZHBbmVpK7GrnocHmVG1Q) column constraints

| Teradata | Snowflake |
| :--- | :--- |
| Default time | Default current\_time\(\) |
| Default date '9999-12-31' | Default '9999-12-31'::date |
| Default user | Default current\_user\(\) |
| Default current\_timestamp\(6\) | Default current\_date\(\) |

* Removed statements

| Teradata | Snowflake |
| :--- | :--- |
| Normalize | Unsupported |
| Table Options | Unsupported |
| Block compression | Unsupported |
| On commit | Unsupported |
| COMPRESS \(value1. , value2. , value3.\) | Unsupported |

#### 1.1.2 Create or Replace view transformation

> See [Create view](https://docs.teradata.com/reader/scPHvjfglIlB8F70YliLAw/EXhAa7frdTDJwg2OZukLgQ)

**Teradata**

```sql
Replace view view1 (someTable.col1, someTable.col2) as locking row for access
    select 
    my_table.col1, my_table.col2
    from table1 as my_table
    where my_table.col1 = 'Mobilize'
    union all
    select other_table.col2
    from table2 as other_table
    where my_table.col2 = other_table.col2)
```

**Snowflake**

```sql
Create or replace view view1 (col1, col2)
as
   select
   my_table.col1, my_table.col2
   from table1 as my_table
   where my_table.col1 = 'Mobilize'
   union all
   select other_table.col2
   from table2 as other_table
   where my_table.col2 = other_table.col2
```

* Summary of notable changes

| Fragment | Action | Snowflake | Notes |
| :--- | :--- | :--- | :--- |
| Replace view | Transform to | Create or replace view | Snowflake supports CREATE OR REPLACE |
| \( someTable.col1, someTable.col2 \) | Transform to | \(col1, col2\) | Reference is not required |
| Locking row for access | Remove |  | Not supported |

#### 1.1.3 Stored procedure transformation

> See [Stored procedure](https://docs.teradata.com/reader/scPHvjfglIlB8F70YliLAw/MrlbjzckdTNAiFTirFtXoQ)

**Teradata**

```sql
replace procedure my_procedure (in param1 varchar(10), out param2 blob) dynamic result sets 9
select * from table1;
```

**Snowflake**

```sql
create or replace procedure my_procedure ( param1 string, param2 binary )
   returns string
   language javascript
   execute as caller
   as
```

```javascript
   $$
     var resultsetcounter = 0;
    var tablelist = new array;
    var sql_stmt = `select * from table1`;
    snowflake.createstatement({
       sqltext : sql_stmt
    }).execute();
    return tablelist;
$$
```

_Note: Stored procedure's body in Snowflake are executed as javascript functions._

**1.1.3.1 If statement**

The transformation for the [IF statement](https://docs.teradata.com/reader/I5Vi6UNnylkj3PsoHlLHVQ/GOzyPogDqU7DWoFvg6YMCw) is:

**Teradata**

```sql
IF value = 2 THEN
```

**Snowflake**

```javascript
if(value == 2){
}
```

**1.1.3.2 Case statement**

The transformation for the [Case statement](https://docs.teradata.com/reader/I5Vi6UNnylkj3PsoHlLHVQ/nuR4riyH6QmcdmMu01TQEw) is:

**Teradata**

```sql
case value
when 0 then
  select * from table1
else
  update table1 set name = "Mobilize" where id = value;
end case
```

**Snowflake**

```javascript
switch(value) {
       case 0:var sql_stmt = ` SELECT * from table1 `;
       var stmt = snowflake.createStatement({
          sqlText : sql_stmt
       });
       var res = stmt.execute();
       res.next();
       var pid = res.getColumnValue(1);
       break;
       default:var sql_stmt = ` UPDATE table1 SET name=Mobilize WHERE  id = value `;
       snowflake.createStatement({
          sqlText : sql_stmt
       }).execute();
       break;
}
```

**1.1.3.3 Cursor statement**

The transformation for [cursor statements](https://docs.teradata.com/reader/I5Vi6UNnylkj3PsoHlLHVQ/vLlfGRxfadgP4k0a~0jpkA) is:

**Teradata**

```sql
Replace procedure procedure1()               
dynamic result sets 2
begin

    -------- Local variables --------
    declare sql_cmd varchar(20000) default ' '; 
    declare num_cols integer;

    ------- Declare cursor with return only-------
    declare resultset cursor with return only for firststatement;

    ------- Declare cursor -------
    declare cur2 cursor for select count(columnname) from table1;

    -------- Set --------
    set sql_cmd='sel * from table1';

    -------- Prepare cursor --------
    prepare firststatement from sql_cmd; 

    -------- Open cursors --------
    open resultset;        
    open cur1;

    -------- Fetch -------------
    fetch cur1 into val1, val2;

    -------- Close cursor --------
    close cur1;
end;
```

**Snowflake**

```sql
create or replace procedure procedure1 ()
returns string 
language javascript
execute as caller
as
```

```javascript
$$
     var resultSetCounter = 0;
    var tablelist = new Array;
    var sql_cmd = ` `;
    var num_cols;

    //----- Declare cursor with return only-------
    var procname = `procedure1`;
    var sql_command = `select current_session() || '_' || to_varchar(current_timestamp, 'yyyymmddhh24missss')`;
    var stmt = snowflake.createStatement({
       sqlText : sql_command
    });
    var res = stmt.execute();
    res.next();
    var sessionid = procname + res.getColumnValue(1);

    //----- Declare cursor-------
    var sql_stmt = `select count(columnname) from table1;`;
    var cur2 = snowflake.createStatement({
       sqlText : sql_stmt
    }).execute();

    //------ Set --------
    sql_cmd = `select * from table1`;

    //------ Prepare cursor --------
    var setname = sql_cmd;
    var tablename = sessionid + `_` + resultSetCounter++;
    var sql_stmt = `CREATE TEMPORARY TABLE ${tablename} AS ${setname}`;
    tablelist.push(tablename);

    //------ Open cursors --------
    var resultset = snowflake.createStatement({
       sqlText : sql_stmt
    }).execute();
    var cur1 = snowflake.createStatement({
       sqlText : sql_stmt
    }).execute();
    cur1.next();

    //------ Fetch cursor -------
    val1 = cur1.getColumnValue(1);
    val2 = cur1.getColumnValue(2);

    //------ Close cursor --------
    ;
    return tablelist;
$$;
```

_Note: The close cursor statement is transformed to an empty statement since it is not used in Snowflake. For every value that is being fetched_ _from the cursor, one `getColumnValue` is added._

**1.1.3.4 While statement**

The transformation for [while statement](https://docs.teradata.com/reader/I5Vi6UNnylkj3PsoHlLHVQ/fTCuW3l9hT6vtPc7V3QpcA) is:

**Teradata**

```sql
while (counter < 10) do
    set counter = counter + 1;
```

**Snowflake**

```javascript
while ( counter < 10) {
    counter = counter + 1;
}
```

**1.1.3.5 Security statements**

The transformation for [security statements](https://docs.teradata.com/reader/zzfV8dn~lAaKSORpulwFMg/knEJa8MckUZrAYquDblFAA) is:

| Teradata | Snowflake |
| :--- | :--- |
| SQL SECURITY CREATOR | EXECUTE AS OWNER |
| SQL SECURITY INVOKER | EXECUTE AS CALLER |
| SQL SECURITY DEFINER | EXECUTE AS OWNER |

**1.1.3.6 Create macro**

The transformation for [create macro](https://docs.teradata.com/reader/scPHvjfglIlB8F70YliLAw/ro_n1pKi4lxSEISZ_IP9lA) is:

**Teradata**

```sql
Create macro new_table (col1   INTEGER, 
col2     VARCHAR(12)) as
(insert into table1 (col1, col2)
values (:col1, :col2);
select * from table1 
where col1 = :col1; )
```

**Snowflake**

```sql
Create or replace procedure new_table (col1   integer, col2 varchar(12))
  returns string
  language javascript
  execute as caller
  as
```

```javascript
  $$
     var sql_stmt = `insert into table1 (col1, col2)
    values (:col1, :col2)`;
    snowflake.createstatement({
       sqltext : sql_stmt
    }).execute();
    var sql_stmt = `select *
    from table1
    where col1 = :col1`;
    snowflake.createstatement({
       sqltext : sql_stmt
    }).execute();
$$
```

_Note: The Teradata Macro is transformed to a stored procedure since Snowflake does not support Macros._

**1.1.3.7 FOR-CURSOR-FOR loop**

The transformation for [FOR-CURSOR-FOR loop](https://docs.teradata.com/reader/scPHvjfglIlB8F70YliLAw/YyY70D3vVqnHSAIE30t78g) is:

**Teradata**

```sql
REPLACE PROCEDURE Database1.Proc1()
BEGIN
    DECLARE lNumber INTEGER DEFAULT 1;
    FOR class1 AS class2 CURSOR FOR 
      SELECT COL0,
      TRIM(COL1) AS COL1ALIAS,
      TRIM(COL2),
      COL3
      FROM someDb.prefixCol
    DO
      INSERT INTO TempDB.Table1 (:lgNumber, :lNumber, (',' || :class1.ClassCD || '_Ind CHAR(1) NOT NULL'));
      SET lNumber = lNumber + 1;
    END FOR;
END;
```

**Snowflake**

```javascript
CREATE OR REPLACE PROCEDURE Database1.PUBLIC.Proc1 ()
  RETURNS STRING
  LANGUAGE JAVASCRIPT
  EXECUTE AS CALLER
  AS
  $$
     var lNumber = `1`;
    var sql_stmt = `              SELECT COL0,
                  TRIM(COL1) AS COL1ALIAS,
                  TRIM(COL2),
                  COL3
                  FROM someDb.PUBLIC.prefixCol`;

    var res = snowflake.createStatement({
       sqlText : sql_stmt
    }).execute();
    while ( res.next() ) {
       var class1 = {
          COL0 : res.getColumnValue(1),
          COL1ALIAS : res.getColumnValue(2),
          COL3 : res.getColumnValue(4)
       };
       var sql_stmt = `              INSERT INTO TempDB.PUBLIC.Table1 (:lgNumber, :lNumber, (',' || :class1.ClassCD || '_Ind CHAR(1) NOT NULL'))`;
       snowflake.createStatement({
          sqlText : sql_stmt
       }).execute();
       test = class1.COL1ALIAS + 1;
    }

$$;
```

_Note: The FOR loop present in the Teradata procedure is transformed to a WHILE block in javascript that emulates its functionality._

**1.1.3.8 Procedure parameters and variables referenced inside statements**

The transformation for the procedure parameters and variables that are referenced inside the statements of the procedure is:

**Teradata**

```sql
REPLACE PROCEDURE PROC1 (param1 INTEGER, param2 VARCHAR(30))

BEGIN
    DECLARE var1            VARCHAR(1024); 
    DECLARE var2                  SMALLINT;
    DECLARE weekstart date;                                 
    set weekstart= '2019-03-03';
    set var1 = 'something';
    set var2 = 123;

    SELECT * FROM TABLE1 WHERE SOMETHING = :param1;
    SELECT * FROM TABLE1 WHERE var1 = var1 AND date1 = weekstart AND param2 = :param2;
    INSERT INTO TABLE2 (col1, col2, col3, col4, col5) VALUES (:param1, :param2, var1, var2, weekstart);
END;
```

**Snowflake**

```javascript
CREATE OR REPLACE PROCEDURE sfSchema.PROC1 (param1 FLOAT, param2 STRING)
   RETURNS STRING
   LANGUAGE JAVASCRIPT
   EXECUTE AS CALLER
   AS
   $$
    var var1;
    var var2;
    var weekstart;
    weekstart = `2019-03-03`;
    var1 = `something`;
    var2 = 123;

    var sql_stmt = `    SELECT * FROM sfSchema.TABLE1 WHERE SOMETHING = ` + PARAM1;
    snowflake.createStatement({
       sqlText : sql_stmt
    }).execute();
    var sql_stmt = `    SELECT * FROM sfSchema.TABLE1 WHERE var1 =  ` + var1 + ` AND date1 =`+ weekstart +` AND param2 = ` + PARAM2;
    snowflake.createStatement({
       sqlText : sql_stmt
    }).execute();
    var sql_stmt = `    INSERT INTO sfSchema.TABLE2 (col1, col2, col3, col4, col5) VALUES (` + PARAM1 + (`, ` + PARAM2) + `, `+ var1 +`,`+ var2+`,`+ weekstart+`)`;
    snowflake.createStatement({
       sqlText : sql_stmt
    }).execute();

$$;
```

_Note: Whenever a procedure parameter or a variable declared inside the procedure is referenced inside a Teradata statement that has to be converted,_ _this reference is escaped from the resulting text to preserve the original reference's functionality._

**1.1.3.9  Leave statement**

In Javascript, it's possible to use `break` with an additional parameter, thus emulating the behavior of a Teradata `LEAVE` jump.

Labels can also be emulated by using Javascript Labeled Statements.

The transformation for [LEAVE statement](https://docs.teradata.com/reader/I5Vi6UNnylkj3PsoHlLHVQ/60WkuZd8ir9NgHlhlJcxIA) is:

**Teradata**

```sql
REPLACE PROCEDURE  PROC1 ()
BEGIN
  DECLARE v_propval            VARCHAR(1024);

 DECLARE Cur1 cursor for 
   Select 
      propID
   from viewName.viewCol
   where propval is not null;

LABEL_WHILE:
  WHILE (SQLCODE = 0)
  DO
      IF (SQLSTATE = '02000' ) 
       THEN LEAVE LABEL_WHILE;
      END IF;
      LABEL_INNER_WHILE:
      WHILE (SQLCODE = 0)
      DO
        IF (SQLSTATE = '02000' ) 
          THEN LEAVE LABEL_INNER_WHILE;
        END IF;
      END WHILE LABEL_INNER_WHILE;
      SELECT * FROM TABLE1;
  END WHILE L1;
END;
```

**Snowflake**

```javascript
CREATE OR REPLACE PROCEDURE PUBLIC.PROC1 ()
  RETURNS STRING
  LANGUAGE JAVASCRIPT
  EXECUTE AS CALLER
  AS
  $$
     var v_propval;
    var sql_stmt = `   Select
          propID
       from viewName.PUBLIC.viewCol
       where propval is not null;`;
    var Cur1 = snowflake.createStatement({
       sqlText : sql_stmt
    }).execute();
    LABEL_WHILE: {
       while ( SQLCODE == 0 ) {
          if (SQLSTATE == `02000`) {
             break LABEL_WHILE;
          }
          LABEL_INNER_WHILE: {
             while ( SQLCODE == 0 ) {
                if (SQLSTATE == `02000`) {
                   break LABEL_INNER_WHILE;
                }
             }
          }
          var sql_stmt = `      SELECT * FROM PUBLIC.TABLE1`;
          snowflake.createStatement({
             sqlText : sql_stmt
          }).execute();
       }
    }
```

### 1.2 Transforming SQL DML statements

This section illustrates Data Manipulation Language \(DML\) syntax transformations from Teradata to Snowflake.

#### 1.2.1 Supported statements

| Teradata | Snowflake |
| :--- | :--- |
| [Set operators](https://docs.teradata.com/reader/b8dd8xEYJnxfsq4uFRrHQQ/Q8qU3AO1RXLNFCPOGTX73g) | Supported |
| [Join expressions](https://docs.teradata.com/reader/b8dd8xEYJnxfsq4uFRrHQQ/BykP75yB91XVKqdlAwK46g) | Supported except for the `self join` |
| [Merge](https://docs.teradata.com/reader/b8dd8xEYJnxfsq4uFRrHQQ/4OgUH6g~Qfr0TwCyhBrg9g) | Supported except for the `logging errors` statement |
| [Insert statement](https://docs.teradata.com/reader/b8dd8xEYJnxfsq4uFRrHQQ/JCM9lCdfy7vLks19B2f1nA) | Supported except for the `INS` abbreviation which is transformed to `INSERT INTO` |
| [Delete statement](https://docs.teradata.com/reader/b8dd8xEYJnxfsq4uFRrHQQ/q7f5d0VHHUBMakwj_dYF~A) | Supported except for the `with isolated loading` statement. DEL is transformed to DELETE. |

#### 1.2.2 Select statement transformation

> See [Select statement](https://docs.teradata.com/reader/b8dd8xEYJnxfsq4uFRrHQQ/kH97CTRIXdd~i1yLemdvKw)

**Teradata**

```sql
sel distinct col1, col2 from table1
```

**Snowflake**

```sql
select distinc col1, col2 from table1
```

_Note: Snowflake supports Teradata's select syntax with a few exceptions. Primarily, it does not support the `sel` abbreviation._

* [Select](https://docs.teradata.com/reader/b8dd8xEYJnxfsq4uFRrHQQ/kH97CTRIXdd~i1yLemdvKw) with alias reference

**Teradata**

```sql
select
my_val, sum(col1),
col2 as my_val from table1
```

**Snowflake**

```sql
select
col2, sum(col1),
col2 as my_val from MY_DW.table1
```

_Note: Teradata supports referencing an alias before it is declared, but Snowflake does not. The transformation for this scenario is to_ _take the referenced column and change the alias for the column name it references._

* Removed statements

| Teradata | Snowflake |
| :--- | :--- |
| Expand on | Unsupported |
| Normilize | Unsupported |
| With check option \(Query\) | Unsupported |

#### 1.2.3 Alternate insert structure transformation

There's an alternate INSERT structure that assigns the value for each table column inline. This alternate structure requires a special transformation to be supported in Snowflake.

**Teradata**

```sql
INSERT INTO appDB.logTable (
    process_name = 'S2F_BOOKS_LOAD_NEW'
    , session_id = 105678989 
    , message_txt = '' 
    , message_ts = '2019-07-23 00:00:00'
    , Insert_dt = CAST((CURRENT_TIMESTAMP(0)) AS DATE FORMAT 'YYYY-MM-DD'));
```

**Snowflake**

```sql
INSERT INTO appDB.sfSchema.logTable (
    process_name
    , session_id
    , message_txt
    , message_ts
    , Insert_dt)
VALUES (
    'S2F_BOOKS_LOAD_NEW'
    , 105678989
    , ''
    ,'2019-07-23 00:00:00'
    , CAST((CURRENT_TIMESTAMP(0)) AS VARCHAR));
```

_Note: The inline assignment of the values is separated and placed inside the VALUES\(...\) part of the Snowflake INSERT INTO statement._

### 1.3 Transforming functions and data types

This section illustrates functions and data type transformations from Teradata to Snowflake.

#### 1.3.1 Aggregate functions

| Teradata | Snowflake |
| :--- | :--- |
| AVG | AVG |
| CORR | CORR |
| COUNT | COUNT |
| COVAR\_POP | COVAR\_POP |
| COVAR\_SAMP | COVAR\_SAMP |
| MAX | MAX |
| MIN | MIN |
| STDDEV\_POP | STDDEV\_POP |
| STDDEV\_SAMP | STDDEV\_SAMP |
| SUM | SUM |
| VAR\_POP | VAR\_POP |
| VAR\_SAMP | VAR\_SAMP |
| REGR\_AVGX | REGR\_AVGX |
| REGR\_AVGY | REGR\_AVGY |
| REGR\_COUNT | REGR\_COUNT |
| REGR\_INTERCEPT | REGR\_INTERCEPT |
| REGR\_R2 | REGR\_R2 |
| REGR\_SLOPE | REGR\_SLOPE |
| REGR\_SXX | REGR\_SXX |
| REGR\_SXY | REGR\_SXY |
| REGR\_SYY | REGR\_SYY |
| KURTOSIS | KURTOSIS |
| SKEW | SKEW |
| GROUPING | GROUPING |

> See [Aggregate functions](https://docs.teradata.com/reader/756LNiPSFdY~4JcCCcR5Cw/PAbAQ9g1f_Usv2nQi1XoIA)

#### 1.3.2 Arithmetic functions

| Teradata | Snowflake |
| :--- | :--- |
| ABS | ABS |
| CEILING | CEIL |
| EXP | EXP |
| FLOOR | FLOOR |
| MOD | MOD |
| ROUND | ROUND |
| SIGN | SIGN |
| SQRT | SQRT |
| LOG | LOG |
| LN | LN |
| POWER | POWER |
| TRUNC | TRUNC/TRUNCATE |
| DEGREES | DEGREES |
| RADIANS | RADIANS |

> See [Arithmetic functions](https://docs.teradata.com/reader/756LNiPSFdY~4JcCCcR5Cw/B4qUFD7C8tpKiZVFsYV4fw)

#### 1.3.3 Attribute functions

| Teradata | Snowflake |
| :--- | :--- |
| BIT\_LENGTH | BIT\_LENGTH |
| OCTECT\_LENGTH | OCTECT\_LENGTH |

* See [Attribute functions](https://docs.teradata.com/reader/756LNiPSFdY~4JcCCcR5Cw/5MJZqzn6GV66MZYubeTbTg)

#### 1.3.4 String operators and functions

| Teradata | Snowflake |
| :--- | :--- |
| ASCII | ASCII |
| CHR | CHR/CHAR |
| CONCAT | CONCAT |
| LENGTH | LENGTH |
| LPAD | LPAD |
| RPAD | RPAD |
| LTRIM | LTRIM |
| RTRIM | RTRIM |
| TRIM\(LEADING '0' FROM aTABLE\) | LTRIM |
| TRIM\(TRAILING '0' FROM aTABLE\) | RTRIM |
| REVERSE | REVERSE |
| SOUNDEX | SOUNDEX |
| TRANSALTE | TRANSALTE |
| OREPLACE | REPLACE |
| INSTR | POSITION |

> See [String operators and functions](https://docs.teradata.com/reader/756LNiPSFdY~4JcCCcR5Cw/5nyfztBE7gDQVCVU2MFTnA)

#### 1.3.5 Data types

| Teradata | Snowflake |
| :--- | :--- |
| INTEGER | INTEGER |
| SMALLINT | SMALLINT |
| BIGINT | BIGINT |
| DECIMAL | DECIMAL |
| FLOAT | FLOAT |
| NUMERIC | NUMERIC |
| NUMBER | NUMBER |
| CHAR | CHAR/CHARACTER |
| VARCHAR | VARCHAR |
| BLOB | BINARY |
| BYTE | BINARY |
| CLOB | VARCHAR |
| JSON | VARIANT |
| VARBYTE | BINARY |
| CHAR | CHAR/CHARACTER |
| VARCHAR | STRING |
| XML | VARIANT |
| ARRAY | ARRAY |
| DATETIME | DATETIME |
| DATE | DATE |
| TIME | TIME |
| TIME WITH TIMEZONE | TIME |
| TIMESTAMP | TIMESTAMP |
| TIMESTAMP WITH TIMEZONE | TIMESTAMP\_TZ |
| INTERVAL YEAR | VARCHAR |
| INTERVAL MONTH | VARCHAR |
| INTERVAL DAY | VARCHAR |
| INTERVAL HOUR | VARCHAR |
| INTERVAL MINUTE | VARCHAR |
| INTERVAL SECOND | VARCHAR |
| PERIOD\(DATE\) | VARCHAR |
| PERIOD\(TIME\) | VARCHAR |
| PERIOD\(TIMESTAMP\) | VARCHAR |
| PERIOD\(TIMESTAMP WITH TIMEZONE\) | VARCHAR |

> See [Data types](https://docs.teradata.com/reader/~_sY_PYVxZzTnqKq45UXkQ/I_xWuywcishQ9U3Xal6zjA)

#### 1.3.6 Data type conversions

| Teradata | Snowflake |
| :--- | :--- |
| TRYCAST | TRY\_CAST |

> See [Data Type Conversions](https://docs.teradata.com/reader/~_sY_PYVxZzTnqKq45UXkQ/iZ57TG_CtznEu1JdSbFNsQ)

#### 1.3.7 BuiltIn functions conversion

| Teradata | Snowflake |
| :--- | :--- |
| DATE | CURRENT\_DATE |
| TIME | CURRENT\_TIME |

> See [Built-In Functions](https://docs.teradata.com/reader/756LNiPSFdY~4JcCCcR5Cw/Zi07FNNZGhn7X2i3dWewuQ)

#### 1.3.8 Functions mapping

There are some special and slightly more complex transformations for certain patterns of Teradata functions:

| Teradata | Snowflake |
| :--- | :--- |
| BEGIN\(period\) | PERIOD\_BEGIN\_UDF\(period\) |
| CAST\(my\_date as INT\) | DATE\_TO\_INT\_UDF\(my\_date\) |
| CAST \('12 : 34 : 56.78' AS HOUR\(2\) TO SECOND\(2\)\) | INTERVAL '12 hour, 34 min, 56.78 sec' |
| MONTHS\_BETWEEN\(value1, value2\) | MONTHS\_BETWEEN\_UDF\(value1, value2\) |
| NULLIFZERO\(value\) | CASE WHEN value = 0 THEN null ELSE value END |
| ROUND\(DATE, 'RM'\) RND\_DATE | ROUND\_DATE\_UDF\(DATEFIRSTPURCHASE, 'RM'\) RND\_DATE |
| COLUMN1 OVERLAPS PERIOD\(DATE '2009-01-01', DATE '2010-09-24'\) | PERIOD\_OVERLAPS\_UDF\( COLUMN1, '2009-01-01 \* 2010-09-24'\) |

## 2. BTEQ transformations

This section illustrates [BTEQ](https://docs.teradata.com/reader/bwmTub8T37PmQEaOVoPd1A/D7oJz6odzabgPfRQA8ZaKQ) transformations from Teradata to Snowflake.

### 2.1 Snowconvert\_helpers

In order to simulate the BTEQ functionality for Teradata in Snowflake, BTEQ files and commands are transformed to Python code. This code uses the Mobilize.net Python project called Snowconvert\_helpers that contains the required functions for simulating BTEQ statements in Snowflake.

### 2.2 BTEQ statements transformation

| Teradata | Snowflake |
| :--- | :--- |
| ERRORCODE != 0 | Snowconvert\_helpers.error\_code != 0 |
| .QUIT ERRORCODE | Snowconvert\_helpers.quit\_application\(snowconvert\_helpers.error\_code\) |
| QUIT | Snowconvert\_helpers.quit\_application\(\) |
| .IF ERRORCODE != 0 THEN .QUIT ERRORCODE | If snowconvert\_helpers.error\_code != 0: snowconvert\_helpers.quit\_application \(snowconvert\_helpers.error\_code\) |
| .SET ERRORLEVEL 3807 SEVERITY 0 | snowconvert\_helpers.set\_error\_level\(3807, 0\) |
| SQL statements | Snowconvert\_helpers.execute\_sql\_statement\(statement\) |
| LOGOFF | Ignored since it is not required |
| .OS /fs/fs01/bin/filename.sh 'load' | snowconvert\_helpers.os\(""/fs/fs01/bin/filename.sh 'load' ""\) |
| .Remark ""Hello world!""" | snowconvert\_helpers.remark\(r""""""Hello world!""""""\) |
| = \(Repeat previous command\) | snowconvert\_helpers.repeat\_previous\_sql\_statement\(con\) |

#### 2.2.1 .GOTO transformation

Since we are converting BTEQ scripts to Python, certain structures that are valid in BTEQ are not inherently supported in Python. This is the case for the `.GOTO` command using the `.Label` commands.

For this reason, an alternative has been developed so that the functionality of these commands can be emulated, turning the `.Label` commands into functions with subsequent call statements.

Check the following code:

```sql
.LABEL FIRSTLABEL
SELECT * FROM MyTable1;
.LABEL SECONDLABEL
SELECT * FROM MyTable2;
SELECT * FROM MyTable3;
```

In the example above, there were 5 commands. Two of them were `.Label` commands. The command `FIRSTLABEL` was turned into  
a function where the statement\(s\) that follow that function below to that function until the code runs into another `.LABEL` command. When another label is called \(in this case, `SECONDLABEL`\), that call ends the first function and starts a new one.

If we were to migrate the previous example, the result would be:

```python
import sys

import snowconvert_helpers


def SECONDLABEL():
   snowconvert_helpers.execute_sql_statement("""SELECT * FROM PUBLIC.MyTable2""", con)
   snowconvert_helpers.execute_sql_statement("""SELECT * FROM PUBLIC.MyTable3""", con)

def FIRSTLABEL():
   snowconvert_helpers.execute_sql_statement("""SELECT * FROM PUBLIC.MyTable1""", con)
   SECONDLABEL()
con = None

try:
   snowconvert_helpers.configure_log()
   con = snowconvert_helpers.log_on(sys.argv[1], sys.argv[2], sys.argv[3])
   FIRSTLABEL()
except Exception as e:
   print(e)
finally:

   if con is not None:
      con.close()


   snowconvert_helpers.quit_application()
```

_Note that in the `try` statements, there is a call to the function FIRSTLABEL. This function has only one statement, which would be the only non label command that follows FIRSTLABEL in the original code. Before the FIRSTLABEL function ends, it calls SECONDLABEL, with the statements that it requires._

Let's see an example with one `.Label`:

**Teradata**

```sql
...
...

.IF ERRORLEVEL > 0 THEN .GOTO LABELEND;

...
...

.LABEL LABELEND
.QUIT ERRORCODE
```

**Snowflake**

```python
import sys

import snowconvert_helpers


def LABELEND():
   snowconvert_helpers.quit_application(snowconvert_helpers.error_code)
con = None

try:
   snowconvert_helpers.configure_log()
   con = snowconvert_helpers.log_on(sys.argv[1], sys.argv[2], sys.argv[3])
   ...
   ...

   if snowconvert_helpers.error_level > 0:
      LABELEND()
   LABELEND()
except Exception as e:
   print(e)
finally:

   if con is not None:
      con.close()


   snowconvert_helpers.quit_application()
```

Now, let's see another example with multiple `.Label` commands.

Consider the following code:

**Teradata**

```sql
.LABEL SKIPDROP

.IF ACTIVITYCOUNT <> 0 THEN .GOTO SKIPCREATETABLE;

CREATE SET TABLE MyTable ,NO FALLBACK ,
    NO BEFORE JOURNAL,
    NO AFTER JOURNAL,
    CHECKSUM = DEFAULT
    (
        col1 DATE FORMAT 'YYYYMMDD' NOT NULL,
        col2 DATE FORMAT 'YYYYMMDD' NOT NULL
    )
    UNIQUE PRIMARY INDEX UPI_dat_relevant (col1);

.LABEL SKIPCREATETABLE
.IF ERRORCODE <> 0 THEN .GOTO ERRORLABEL;

DELETE FROM MyTable All;
INSERT INTO MyTable VALUES(1,2);

.IF ERRORCODE <> 0 THEN .GOTO SKIPSELECT;
SELECT SUM(COL1) FROM MyTable;
SELECT SUM(COL2) FROM MyTable;
SELECT * FROM MyTable;

.LABEL SKIPSELECT

.LABEL GOODEND
.QUIT

.LABEL ERRORLABEL
.QUIT ERRORCODE
```

The Teradata code above has multiple labels. If this is transformed to Python, there is an order of execution to keep the functionality the same to the one in the BTEQ script. If the code is migrated to Snowflake, the result is given below.

**Snowflake**

```python
import sys

import snowconvert_helpers


def ERRORLABEL():
   snowconvert_helpers.quit_application(snowconvert_helpers.error_code)

def GOODEND():
   snowconvert_helpers.quit_application()
   ERRORLABEL()

def SKIPSELECT():
   GOODEND()

def SKIPCREATETABLE():

   if snowconvert_helpers.error_code != 0:
      ERRORLABEL()
   snowconvert_helpers.execute_sql_statement("""DELETE FROM PUBLIC.MyTable""", con)
   snowconvert_helpers.execute_sql_statement("""INSERT INTO PUBLIC.MyTable VALUES (1,2)""", con)

   if snowconvert_helpers.error_code != 0:
      SKIPSELECT()
   snowconvert_helpers.execute_sql_statement("""SELECT SUM(COL1) FROM PUBLIC.MyTable""", con)
   snowconvert_helpers.execute_sql_statement("""SELECT SUM(COL2) FROM PUBLIC.MyTable""", con)
   snowconvert_helpers.execute_sql_statement("""SELECT * FROM PUBLIC.MyTable""", con)
   SKIPSELECT()

def SKIPDROP():

   if snowconvert_helpers.activity_count != 0:
      SKIPCREATETABLE()
   snowconvert_helpers.execute_sql_statement("""/**** WARNING: SET TABLE FUNCTIONALITY NOT SUPPORTED ****/
CREATE TABLE PUBLIC.MyTable (
col1 DATE NOT NULL /**** WARNING: FORMAT 'YYYYMMDD' NOT SUPPORTED ****/,
col2 DATE NOT NULL /**** WARNING: FORMAT 'YYYYMMDD' NOT SUPPORTED ****/,
UNIQUE (col1)
)""", con)
   SKIPCREATETABLE()
con = None

try:
   snowconvert_helpers.configure_log()
   con = snowconvert_helpers.log_on(sys.argv[1], sys.argv[2], sys.argv[3])
   SKIPDROP()
except Exception as e:
   print(e)
finally:

   if con is not None:
      con.close()


   snowconvert_helpers.quit_application()
```

Initially, it may look like the order of the functions are inverted. However, if you follow the flow of execution, you will notice that both code segments are functionally the same.

> * _Notes:_
>   * _The code shows a case when the label's original purpose is to exit the application._
>   * _The `.IF` command is mapped to a normal Python IF statement._
>   * _The `.GOTO LABELEND` is mapped to a Python call to `LABEL_LABELEND()`._

## 3. FastLoad transformations

This section illustrates [FastLoad](https://docs.teradata.com/reader/zZBXk3M0Kz2mYRdYk2Dynw/2T77lYOWeDrkMzd5sA5tnA) command transformations from Teradata to Snowflake.

FastLoad scripts are transformed to Python in a similar manner to the transformations performed for BTEQ scripts.

### 3.1 FastLoad commands transformation

Most of the [FastLoad commands](https://docs.teradata.com/reader/vIWhrlrRPxEfMbR9H0qaTQ/M~KKy88THLFCPhbu3knTfg) are considered "not relevant" in Snowflake. Here is the list of the commands for FastLoad along with its transformation status.

| Teradata FastLoad Command | Transformation Status | Note |
| :--- | :--- | :--- |
| AXSMOD | Commented |  |
| BEGIN LOADING | Transformed |  |
| CLEAR | Commented |  |
| DATEFORM | Commented |  |
| DEFINE | Commented |  |
| END LOADING | Commented |  |
| ERRLIMIT | Commented |  |
| HELP | Commented |  |
| HELP TABLE | Commented |  |
| INSERT | Transformed | This is taken as a Teradata Statement, so it doesn't appear in this chapter. |
| LOGDATA | Commented |  |
| LOGMECH | Commented |  |
| LOGOFF | Commented |  |
| LOGON | Commented |  |
| NOTIFY | Commented |  |
| OS | Commented |  |
| QUIT | Commented |  |
| RECORD | Commented |  |
| RUN | Commented |  |
| SESSIONS | Commented |  |
| SET RECORD | Commented |  |
| SET SESSION CHARSET | Commented |  |
| SHOW | Commented |  |
| SHOW VERSIONS | Commented |  |
| SLEEP | Commented |  |
| TENACITY | Commented |  |

The default behavior of the ConversionTool for these statements is to comment them out. For example:

**Input: Teradata \(FastLoad\)**

```sql
SESSIONS 4;
ERRLIMIT 25;
```

**Output: Snowflake \(Python\)**

```python
  # 
  # Conversion Note - Removed next statement, not applicable in Snowflake.
  # SESSIONS 4

  # 
  # Conversion Note - Removed next statement, not applicable in Snowflake.
  # ERRLIMIT 25
```

Nonetheless, there are some exceptions that must be converted to specific Python statements in order to work as intended in Snowflake.

#### 3.1.1 BEGIN LOADING transformation

Given a `BEGIN LOADING` command:

```sql
BEGIN LOADING FastTable ERRORFILES Error1,Error2
   CHECKPOINT 10000;
```

The following Python code will be created:

```python
    snowconvert_helpers.execute_sql_statement
        (
            """COPY INTO FastTable FROM {} ON_ERROR = CONTINUE""".format(inputDataPlaceholder)
        )
    sql = """CREATE TABLE CTE_FastTable  AS
               SELECT DISTINCT * FROM FastTable"""
    snowconvert_helpers.execute_sql_statement(sql, con)

    sql = """DROP TABLE FastTable"""
    snowconvert_helpers.execute_sql_statement(sql, con)

    sql = """ALTER TABLE CTE_FastTable RENAME TO FastTable"""
    snowconvert_helpers.execute_sql_statement(sql, con)
```

In the example above, `FastTable` is the name of the table associated to the `BEGIN LOADING` command.

#### 3.1.2 Embedded SQL transformations

FastLoad scripts support Teradata statements inside the same file. The majority of these statements are converted just as if they were inside a BTEQ file, with some exceptions.

* Dropping an error table is commented out if inside a FastLoad file.

**FastLoad**

```sql
DROP TABLE Error1;
DROP TABLE Error2;
```

**Snowflake**

```python
  # 
  # Conversion Note - Commented out code related with dropping error table.
  # DROP TABLE Error1

  # 
  # Conversion Note - Commented out code related with dropping error table.
  # DROP TABLE Error2
```

#### 3.1.3 Example

Taking into consideration the transformation rules for FastLoad, let's see an example and it's transformation result.

```sql
   SESSIONS 4;
   ERRLIMIT 25;
   DROP TABLE FastTable;
   DROP TABLE Error1;
   DROP TABLE Error2;
   CREATE TABLE FastTable, NO FALLBACK
      ( ID INTEGER, UFACTOR INTEGER, MISC CHAR(42))
      PRIMARY INDEX(ID);
   DEFINE ID (INTEGER), UFACTOR (INTEGER), MISC (CHAR(42))
      FILE=FileName;
   SHOW;
   BEGIN LOADING FastTable ERRORFILES Error1,Error2
      CHECKPOINT 10000;
   INSERT INTO FastTable (ID, UFACTOR, MISC) VALUES
      (:ID, :MISC);
   END LOADING;
```

The transformation for the last example:

```python
import sys

import snowconvert_helpers

con = None

try:
   snowconvert_helpers.configure_log()
   con = snowconvert_helpers.log_on(sys.argv[1], sys.argv[2], sys.argv[3])
   # 
   # Conversion Note - Removed next statement, not applicable in Snowflake.
   # SESSIONS 4

   # 
   # Conversion Note - Removed next statement, not applicable in Snowflake.
   # ERRLIMIT 25

   snowconvert_helpers.execute_sql_statement("""DROP TABLE PUBLIC.FastTable""", con)
   # 
   # Conversion Note - Commented out code related with dropping error table.
   # DROP TABLE Error1

   # 
   # Conversion Note - Commented out code related with dropping error table.
   # DROP TABLE Error2

   snowconvert_helpers.execute_sql_statement("""CREATE TABLE PUBLIC.FastTable
(
ID INTEGER,
UFACTOR INTEGER,
MISC CHAR(42))""", con)
   # 
   # Conversion Note - Removed next statement, not applicable in Snowflake.
   # DEFINE ID (INTEGER), UFACTOR (INTEGER), MISC (CHAR(42))
   #    FILE=FileName

   # 
   # Conversion Note - Removed next statement, not applicable in Snowflake.
   # SHOW

   snowconvert_helpers.execute_sql_statement("""COPY INTO FastTable FROM {} ON_ERROR = CONTINUE""".format(inputDataPlaceholder))
   sql = """CREATE TABLE CTE_FastTable  AS SELECT DISTINCT * FROM FastTable"""
   snowconvert_helpers.execute_sql_statement(sql, con)
   sql = """DROP TABLE FastTable"""
   snowconvert_helpers.execute_sql_statement(sql, con)
   sql = """ ALTER TABLE CTE_FastTable RENAME TO FastTable"""
   snowconvert_helpers.execute_sql_statement(sql, con)
   # 
   # Conversion Note - Removed next statement, not applicable in Snowflake.
   # END LOADING

except Exception as e:
   print(e)
finally:

   if con is not None:
      con.close()


   snowconvert_helpers.quit_application()
```

## 4. MultiLoad transformations

This section illustrates [MultiLoad](https://docs.teradata.com/reader/u5g65Je3hpMChJXfDyt1hg/TwU3KZ98YBpEbST_i7txrg) command transformations from Teradata to Snowflake.

Multiload scripts are transformed to Python, similar to the transformations performed for BTEQ and FastLoad scripts.

### 4.1 MultiLoad commands transformation

Like Fastload, the majority of [MultiLoad commands](https://docs.teradata.com/reader/u5g65Je3hpMChJXfDyt1hg/KRpA6tp8QD64m48~ng0PFw) are considered "not relevant" in Snowflake. These commands that are commented out.

Below is a list of MultiLoad commands and their transformation status into Snowflake:

| Teradata MultiLoad Command | Transformation Status | Note |
| :--- | :--- | :--- |
| ACCEPT | Commented |  |
| BEGIN MLOAD | Commented |  |
| BEGIN DELETE MLOAD | Commented |  |
| DATEFORM | Commented |  |
| DELETE | Commented |  |
| DISPLAY | Commented |  |
| DML LABEL | Transformed |  |
| END MLOAD | Commented |  |
| EOC | Commented |  |
| FIELD | Removed |  |
| FILLER | Transformed | This command needs to be with a FIELD and LAYOUT command to be converted. |
| IF, .ELSE, and .ENDIF | Commented |  |
| IMPORT | Transformed |  |
| INSERT | Transformed | This is taken as a Teradata Statement, so it doesn't appear in this chapter. |
| LAYOUT | Transformed | This command needs to be with a FIELD and FILLER command to be converted. |
| LOGDATA | Commented |  |
| LOGMECH | Commented |  |
| LOGOFF | Commented |  |
| LOGON | Commented |  |
| LOGTABLE | Commented |  |
| PAUSE ACQUISITION | Commented |  |
| RELEASE MLOAD | Commented |  |
| ROUTE MESSAGES | Commented |  |
| RUN FILE | Commented |  |
| SET | Commented |  |
| SYSTEM | Commented |  |
| TABLE | Commented |  |
| UPDATE | Transformed | This is taken as a Teradata Statement, so it doesn't appear in this chapter. |
| VERSION | Commented |  |

#### 4.1.1 .LAYOUT, .FIELD and .FILLER

If the commands `.LAYOUT`, `.FIELD`, and `.FILLER` are present in the following pattern:

```sql
    .LAYOUT INFILE_LAYOUT;        
    .FIELD TABLE_ID        * INTEGER;                      
    .FIELD TABLE_DESCR     * CHAR(8);                      
    .FIELD TABLE_NBR       * SMALLINT;
```

It will be transformed to:

```python
   INFILE_LAYOUT = "INFILE_LAYOUT_TEMP_TABLE"
   sql = """CREATE TRANSIENT TABLE {}
    (
      TABLE_ID INTEGER,
      TABLE_DESCR CHAR(8),
      TABLE_NBR SMALLINT,
      )""".format(INFILE_LAYOUT)
   snowconvert_helpers.execute_sql_statement(sql, con)
   snowconvert_helpers.import_data_to_temptable(INFILE_LAYOUT, "DATA PLACEHOLDER", con)
   ...
   ...
   ...
   snowconvert_helpers.drop_transient_table(INFILE_LAYOUT, con)
```

Note that the command `.LAYOUT` is transformed to a SQL statement creating a temporal table. After the statement, there are two calls of functions.

The import\_data\_to\_temptable call takes the data in the place holder and copies it to the new table. Why? Since a temporal table is created, the data will be needed for the following commands `insert`, `update`, and/or `delete`.

Finally, drop\_transient\_table is done to delete the temporal table.

#### 4.1.2 .DML LABEL

The transformation For this command, the transformation will create a function, but the content of the function will be determinated based on the next command followed by the `.DML LABEL` command. Note that after this command, an `Insert`, `Update`, or `Delete` can be present.

Here's an example of `.DML LABEL` with `Insert`:

```sql
    .DML LABEL INSERT_TABLE;                                                                                    
    INSERT INTO mydb.mytable( TABLE_ID,TABLE_DESCR,TABLE_NBR ) VALUES( :TABLE_ID,:TABLE_DESCR,:TABLE_NBR );
```

Based on the previous example, the transformation generates the following:

```python
   def INSERT_TABLE(tempTableName):
      sql = """INSERT INTO mydb.mytable (TABLE_ID, TABLE_DESCR, TABLE_NBR) SELECT * FROM {}""".format(tempTableName)
      snowconvert_helpers.execute_sql_statement(sql, con)
```

Now, an example of `.DML LABEL` with `Delete`.

```sql
    .DML Label Deletes;              
    Delete from Employee where EmpNo  = :EmpNo;
```

The result would be:

```python
   def Deletes(tempTableName):
      sql = """DELETE from PUBLIC.Employee where EmpNo  = :EmpNo"""
      snowconvert_helpers.execute_sql_statement(sql, con)
```

Finally, a `.DML LABEL` with an `update`, followed by an `insert`.

```sql
    .DML LABEL UPSERT1 DO INSERT FOR MISSING UPDATE ROWS;
    UPDATE   mydb.mytable SET TABLE_ID = :TABLE_ID WHERE TABLE_DESCR = :somedescription
    INSERT INTO mydb.mytable(TABLE_ID, TABLE_DESCR, TABLE_NBR) VALUES(:TABLE_ID, :TABLE_DESCR, :TABLE_NBR );
```

The result is shown here:

```python
   def UPSERT1(tempTableName):
      sql = """MERGE INTO mydb.mytable USING {} WHEN MATCHED TABLE_DESCR = :somedescription THEN TABLE_ID = :TABLE_ID WHEN NOT MATCHED INSERT VALUES(:TABLE_ID, :TABLE_DESCR, :TABLE_NBR)""".format(tempTableName)
      snowconvert_helpers.execute_sql_statement(sql, con)
      sql = """DROP TABLE {}""".format(tempTableName)
      snowconvert_helpers.execute_sql_statement(sql, con)
```

#### 4.1.3 .IMPORT

Here's an example to illustrate the `.IMPORT` transformation.

```sql
    .IMPORT
        INFILE INFILE_FILENAME
        LAYOUT INFILE_LAYOUT
        APPLY INSERT_TABLE
        APPLY UPSERT1
        Apply Deletes;
```

The transformation would be:

```python
   INSERT_TABLE(INFILE_LAYOUT)
   UPSERT1(INFILE_LAYOUT)
   Deletes(INFILE_LAYOUT)
```

#### 4.1.4 A Larger Example

Given the transformations shown above for a variety of commands, consider the following example:

```sql
.LOGON tpid/TDUSER,PASSWORD;
.LOGTABLE mydb.mytable;

DROP TABLE mydb.mytable_WORKTABLE;              
DROP TABLE mydb.mytable_ERRORTABLE1;                                                            
DROP TABLE mydb.mytable_ERRORTABLE2;                                                            
.BEGIN MLOAD                                                                                          
       TABLES mydb.mytable                                                             
       WORKTABLES  mydb.mytable_WORKTABLE
       ERRORTABLES mydb.mytable_ERRORTABLE1
                   mydb.mytable_ERRORTABLE2
       CHECKPOINT 10;

.LAYOUT INFILE_LAYOUT;        
.FIELD TABLE_ID        * INTEGER;                      
.FIELD TABLE_DESCR     * CHAR(8);                      
.FIELD TABLE_NBR       * SMALLINT;
.FILLER TABLE_SOMEFIELD * SMALLINT;                                                

.DML LABEL INSERT_TABLE;                                                                                      
       INSERT INTO mydb.mytable --Load target table
       ( TABLE_ID
       ,TABLE_DESCR
       ,TABLE_NBR )                                                                                                
       VALUES
       ( :TABLE_ID
       ,:TABLE_DESCR
       ,:TABLE_NBR )                                                                                        
       ;

.IMPORT                                                                                                    
       INFILE INFILE_FILENAME
       LAYOUT INFILE_LAYOUT
       APPLY INSERT_TABLE;

.DML LABEL UPSERT1
        DO INSERT FOR MISSING UPDATE ROWS;
        UPDATE   mydb.mytable
        SET TABLE_ID     = :TABLE_ID
        WHERE  TABLE_DESCR      =  :somedescription
        INSERT INTO    mydb.mytable
        (   TABLE_ID
          , TABLE_DESCR
       , TABLE_NBR)
        VALUES
        (   :TABLE_ID
          , :TABLE_DESCR
          , :TABLE_NBR );   

.DML Label Deletes;              
        Delete from Employee where EmpNo  = :EmpNo;

.IMPORT                                                                                                    
       INFILE INFILE_FILENAME                     
       LAYOUT INFILE_LAYOUT
       APPLY UPSERT1
       Apply Deletes;  

.END MLOAD;

.LOGOFF;
```

After running SnowConvert, the above file \(with an extension of .ml\) is converted to the following code in Python:

```python
import sys

import snowconvert_helpers

con = None

try:
   snowconvert_helpers.configure_log()
   con = snowconvert_helpers.log_on(sys.argv[1], sys.argv[2], sys.argv[3])
   # 
   # Conversion Note - Removed next statement, not applicable in Snowflake.
   # .LOGON tpid/TDUSER,PASSWORD

   # 
   # Conversion Note - Removed next statement, not applicable in Snowflake.
   # .LOGTABLE mydb.mytable_logtable

   snowconvert_helpers.execute_sql_statement("""
DROP TABLE mydb.PUBLIC.mytable_WORKTABLE""", con)
   # 
   # Conversion Note - Commented out code related with dropping error table.
   # DROP TABLE mydb.mytable_ERRORTABLE1

   # 
   # Conversion Note - Commented out code related with dropping error table.
   # DROP TABLE mydb.mytable_ERRORTABLE2

   # 
   # Conversion Note - Removed next statement, not applicable in Snowflake.
   # .BEGIN MLOAD
   #        TABLES mydb.mytable
   #        WORKTABLES  mydb.mytable_WORKTABLE
   #        ERRORTABLES mydb.mytable_ERRORTABLE1
   #                    mydb.mytable_ERRORTABLE2
   #        CHECKPOINT 10

   INFILE_LAYOUT = "INFILE_LAYOUT_TEMP_TABLE"
   sql = """CREATE TRANSIENT TABLE {}
    (
      TABLE_ID INTEGER,
      TABLE_DESCR CHAR(8),
      TABLE_NBR SMALLINT
      )""".format(INFILE_LAYOUT)
   snowconvert_helpers.execute_sql_statement(sql, con)
   snowconvert_helpers.import_data_to_temptable(INFILE_LAYOUT, "DATA PLACEHOLDER", con)

   def import_data_to_temptable(tempTableName, inputDataPlaceholder, con):
    sql = """COPY INTO {} FROM {} ON ERROR = CONTINUE""".
    format(tempTableName, inputDataPlaceholder)
    snowconvert_helpers.execute_sql_statement(sql, con)

   def INSERT_TABLE(tempTableName):
      sql = """INSERT INTO mydb.mytable (TABLE_ID, TABLE_DESCR, TABLE_NBR) SELECT * FROM {}""".format(tempTableName)
      snowconvert_helpers.execute_sql_statement(sql, con)
   INSERT_TABLE(INFILE_LAYOUT)

   def UPSERT1(tempTableName):
      sql = """MERGE INTO mydb.mytable USING {} WHEN MATCHED TABLE_DESCR = :somedescription THEN TABLE_ID = :TABLE_ID WHEN NOT MATCHED INSERT VALUES(:TABLE_ID, :TABLE_DESCR, :TABLE_NBR)""".format(tempTableName)
      snowconvert_helpers.execute_sql_statement(sql, con)
      sql = """DROP TABLE {}""".format(tempTableName)
      snowconvert_helpers.execute_sql_statement(sql, con)

   def Deletes(tempTableName):
      sql = """DELETE from mydb.PUBLIC.mytable where table_id  = 1"""
      snowconvert_helpers.execute_sql_statement(sql, con)
   UPSERT1(INFILE_LAYOUT)
   Deletes(INFILE_LAYOUT)
   snowconvert_helpers.drop_transient_table(INFILE_LAYOUT, con)
   # 
   # Conversion Note - Removed next statement, not applicable in Snowflake.
   # 
   # .END MLOAD

   # 
   # Conversion Note - Removed next statement, not applicable in Snowflake.
   # 
   # .LOGOFF

except Exception as e:
   print(e)
finally:

   if con is not None:
      con.close()


   snowconvert_helpers.quit_application()
```

If you have any additional questions regarding this documentation, head on over to the [SnowConvert section of the Mobilize.Net forums](https://forums.mobilize.net/forum/23-snowconvert/). The forums are monitored by our team of engineers and developers, and any recommendation you may have can only improve our documentation. You can also reach out to support@mobilize.net.

Thanks for reading through the documentation on SnowConvert.

