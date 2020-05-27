---
description: Stay up to date with the latest from the development team.
---

# Release Notes

Here is the latest info on fixes and bugs. If you encounter a problem that you think should be addressed, please reach out to us on our community forums.

{% embed url="https://forums.mobilize.net/forum/23-snowconvert/" %}

And if you don't want to post, email us at [support@mobilize.net](mailto:support@mobilize.net). 

## Release 1.0.1

### GUI

* Adding support for Multiload, FastLoad and Tpump extensions for SnowConvert.

### Conversion Core 1.0.1

#### **Added:**

* Do not generate empty python methods when doing BTEQ migrations.
* Support BT and ET as Begin Transaction and End Transaction in BTEQ.
* Transformation for TITLE and NAMED data attributes in Procedures.
* Params are escaped inside Procedures transformations.
* Updated the definition of log\_on method in snowconvert\_helpers.py.
* The helper file snowconvert\_helpers.py is copied only at the root directory.

#### **Fixed:**

* View ordering and missing schemas.
* "NOT CASESPECIFIC" does not work in Snowflake.
* Not forwarding alias names defined as "NAMED AS".
* FOR LOOP statement not parsed correctly.
* Reported conversion coverage and assessment coverage being different.
* Some Multiset tables not parsing correctly.
* TRIM \(BOTH FROM ' TEST '\) is copied as it is.
* Comments removed from View definition and losing formatting for all SQLs.
* Reported errors in snowconvert\_helpers.py python file.
* Exception in the Symbol's table when files in different directories have the same name.
* Some BTEQ statements being trimmed in the converted coded.
* Keep Label ordering in conversion of BTEQ to Python.
* CURRENT\_DATE statement is being converted with quotes "CURRENT\_DATE".\`
* Wrong Parenthesis after migration for DT statement.
* Many errors in the conversion of SET and DECLARE statements in Procedures.
* String Terminator in BTEQ Queries is being concatenated with another string terminator.

## Release 1.0.0

### GUI

* Using Product Licensing API 3.0.0.
* Update crash screen to allow users to send crash report information.

### Conversion Core 1.0.0

* Conversion rate for Macros now counts partial conversions.
* Added support for ABORT statements conversion.
* Added support for ECHO statement conversion.
* Fixed some procedures being converted with an empty body.
* Fixed a reported error that prevented the conversion of Multiload files.
* Fixed missing parameter types in converted Procedures/Macros.

## Release 0.5.8

### Conversion Core 0.3.5

* Added support for conversion of REPEAT and LOOP statements.
* Added support for multiple DECLARE HANDLER in a Stored Procedure.
* Bugfixes in the conversion of Stored Procedures.
* Added support for conversion of GLOBAL TEMPORARY TABLE.

## Release 0.5.6

### GUI

* Check for updates menu option available.
* Modals styles improvements.
* Display execution mode when running in Assessment Mode in SnowConvert UI.
* New Mobilize.Net SnowConvert logo.

### Conversion Core 0.3.4

* Fixed parsing and conversion errors in Stored Procedures.
* Fixed an issue when reordering tables related to the syntax analysis.
* Added support for DECLARE EXIT HANDLER conversion.
* When Dynamic SQL cannot be converted it's now marked as a warning and not a conversion error.

## Release 0.5.5

* New version notification on first run.
* Improved user experience in settings window.
* Improved user experience for critial errors.

## Release 0.5.4

### Conversion Core 0.3.0

* Added support for TPUMP conversion to Python.
* Added support for SQL conversion of JOIN INDEX to MATERIALIZED VIEW.
* Added generation of Assessment Report.
* Improved preserving of comments in BTEQ and Stored Procedures.

## Release 0.5.3

* Fixed a problem with license validation on unusual environments.

## Release 0.5.2

* Added user guide in Help menu.
* Disabled some menu options while the conversion is being executed.
* Performance improvements and bugfixes.

## Release 0.5.1

### GUI

* Added Settings button in the Start Conversion window.
* Performance improvements and bugfixes.

### Conversion Core 0.3.0

* Support for FastLoad conversion.
* Support for MultiLoad conversion.
* Parsing improvements for Procedures.
* Initial version of the Assessment CSV generation.

## Release 0.4.8

* Bugfixes and performance improvements.

## Release 0.4.6

### GUI

* \#244583: Improvements in network detection.
* \#245246: Improvements in license validation.

### Conversion Core 0.2.3

* \#244662: DEL FROM statement converted to DELETE FROM.
* \#244974: Support TRYCAST function.
* \#244977: Support INSTR function.
* \#244818: Exception when converting MERGE sequences.
* \#244044: Conversion of FOR-CURSOR-FOR loops to Snowflake JS.
* \#244987: Upsert statement.
* \#244939: Implement support for LEAVE statements.
* \#245288: SELECT Statement.
* \#246072: DATE not converting to CURRENT\_DATE.
* \#246171: DML - LOCKING with INSERT.
* \#244666: INS statement converted to INSERT INTO.
* \#246202: DML - View prefix identifies.

## Release 0.4.4

### GUI

* \#242955: Update logo and icons \(App and Installer\).
* \#229548: Restart Conversion. Configuration file added.

### Conversion Core 0.2.1

* \#242787: Parameter Type Translation to SP.
* \#242568: Escaping Comments in Commented Code for Procedures. Different loc are now being counted when marked as partially.
* \#238699: Generate new version of details.csv for 'Create Table'.
* \#238704: Generate new version of details.csv for 'Create View'.
* \#243162: Show warning messages in the issues file.
* \#242788: CALL statement for macros/stored procs.
* \#238705: Generate new version of details.csv for 'Create Procedure'.
* \#242790 SET statement with sql.
* \#244348: Input/Output parameters transformation.

## Release 0.4.3

### GUI

* \#240348: Initial splash and loading screen added. Async application start and performance improvements related with electron-cgi connection initialization.
* \#240165: Special characters support bug solved.
* \#237676: Fixed standarization of error name.

### Conversion Core 0.2.0

* \#242598: License Text update to current year.
* \#242619: \[BTEQ\] Helpers is not being copied.
* \#242573: Support for Execute statements added.
* \#242168: For the -m option \(to comment out objects with missing dependencies\) reverse default.
* \#240369: Transformation for Conditional Handler added.

## Release 0.4.0

### GUI

* License modal added to show the license information to the user.
* Notification added to warm the user if the output folder isn't empty.
* Added support to specify input folder error for network directory format.
* Added about menu option to show conversion tool version.
* Conversion Settings Prompt.
* Adding support for auto update notifications mechanism.

### Conversion Core 0.1.3

* \#239790: BTEQ Parsing: If Statement.
* \#239809: BTEQ Parsing: LOGON.
* \#239792: BTEQ Parsing: BEGIN TRANSACTION and END TRANSACTION.
* \#240240: BTEQ Parsing: HELP, SET, DATABASE Stats.
* \#239424: Support to all flavors of identity.
* \#239415: Added validation for parenthesized expression in partition by.
* \#240182: Robustness. SnowConvert should be able to run in any environment that blocks Mobilize's DLLs from execution.
* \#240371: Support for Transformation of Bteq If Statements with Comparisons.
* \#241277: Report and Log name's timestamp format should be the same.
* \#238506: Update Ansi, T12, SnowSql and Python to .NET Standard.
* \#241249: Converting Non Converted statements to JavaScript Comments.
* \#237472: Name columns data insertion improvement.

## Release 0.3.0

### GUI

* Fixed bug related with undetected license when a new version is installed.
* Main menu implementation. \(Options development is in progress\)
* Maximum activations per license.
* Auto populate output folder after selecting the input folder.
* Fixes in license validation process and infinite looping avoided during validation.
* Fixes in labels.

## Release 0.2.0

### GUI

* Fixed bug related with undetected license when a new version is installed.
* Main menu implementation. \(Options development is in progress\)
* Maximum activations per license.
* Auto populate output folder after selecting the input folder.
* Fixes in license validation process and infinite looping avoided during validation.
* Fixes in labels.

### Conversion Core 0.1.2

* \#40568: Added List separator based on the region default delimiter.
* \#237780: Fix of Warning message of Default Session.
* \#238554: Fixed Infinite loops in conversion \(related to forward alias reference issue\).
* \#238517: Circular Reference Alias is being counted as transformation error.
* \#229628: Added validation to avoid symbol table crash in views with circular reference.
* \#237916: Added conversion rule for Expressions like: AnyExpr \(TITLE 'anyString'\) to AnyExpr AS anyString.
* \#237777: Collate clause removed for Variant columns and warning added.
* Parsing Echo, Abort, Select And Consume, Updatability Clause.
* Parsing Ins in Insert, CharacterString Hexadecimal Literals, Character Substring Function.
* \#237780: Fix of Warning message of Default Session.
* \#238554: Fixed Infinite loops in conversion \(related to forward alias reference issue\).
* \#40607: Circular Reference Alias is being counted as transformation error.
* \#40671: Added validation to avoid symbol table crash in views with circular reference.



