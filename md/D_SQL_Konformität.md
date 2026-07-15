# Anhang D. SQL-Konformität

Dieser Abschnitt versucht zu umreißen, in welchem Umfang PostgreSQL dem aktuellen SQL-Standard entspricht. Die folgenden Informationen sind keine vollständige Konformitätserklärung, stellen aber die wichtigsten Themen so detailliert dar, wie es für Benutzer sinnvoll und nützlich ist.

Der formale Name des SQL-Standards lautet ISO/IEC 9075 „Database Language SQL“. Von Zeit zu Zeit erscheint eine überarbeitete Version des Standards; die jüngste Aktualisierung erschien 2023. Die Version von 2023 wird als ISO/IEC 9075:2023 oder einfach als SQL:2023 bezeichnet. Frühere Versionen waren SQL:2016, SQL:2011, SQL:2008, SQL:2006, SQL:2003, SQL:1999 und SQL-92. Jede Version ersetzt die vorherige, sodass Konformitätsansprüche zu älteren Versionen keinen offiziellen Wert haben. Die Entwicklung von PostgreSQL zielt auf Konformität mit der neuesten offiziellen Version des Standards, soweit diese Konformität nicht traditionellen Funktionen oder gesundem Menschenverstand widerspricht. Viele der vom SQL-Standard geforderten Funktionen werden unterstützt, wenn auch manchmal mit leicht abweichender Syntax oder Funktion. Weitere Schritte in Richtung Konformität sind im Laufe der Zeit zu erwarten.

SQL-92 definierte drei Funktionsmengen für Konformität: Entry, Intermediate und Full. Die meisten Datenbankmanagementsysteme, die SQL-Standardkonformität beanspruchten, erfüllten nur das Entry-Level, weil die gesamte Menge der Funktionen auf Intermediate- und Full-Level entweder zu umfangreich war oder mit Legacy-Verhalten in Konflikt stand.

Seit SQL:1999 definiert der SQL-Standard statt der ineffektiv breiten drei Ebenen eine große Menge einzelner Funktionen. Eine große Teilmenge dieser Funktionen bildet die Core-Funktionen, die jede konforme SQL-Implementierung bereitstellen muss. Die übrigen Funktionen sind rein optional.

Der Standard ist in mehrere Teile gegliedert, die jeweils auch einen Kurznamen haben:

- ISO/IEC 9075-1 Framework (SQL/Framework)
- ISO/IEC 9075-2 Foundation (SQL/Foundation)
- ISO/IEC 9075-3 Call Level Interface (SQL/CLI)
- ISO/IEC 9075-4 Persistent Stored Modules (SQL/PSM)
- ISO/IEC 9075-9 Management of External Data (SQL/MED)
- ISO/IEC 9075-10 Object Language Bindings (SQL/OLB)
- ISO/IEC 9075-11 Information and Definition Schemas (SQL/Schemata)
- ISO/IEC 9075-13 Routines and Types using the Java Language (SQL/JRT)
- ISO/IEC 9075-14 XML-related specifications (SQL/XML)
- ISO/IEC 9075-15 Multi-dimensional arrays (SQL/MDA)
- ISO/IEC 9075-16 Property Graph Queries (SQL/PGQ)

Einige Teilenummern werden nicht oder nicht mehr verwendet.

Der PostgreSQL-Kern deckt die Teile 1, 2, 9, 11 und 14 ab. Teil 3 wird vom ODBC-Treiber abgedeckt, und Teil 13 vom PL/Java-Plug-in; die genaue Konformität dieser Komponenten wird derzeit jedoch nicht geprüft. Für die Teile 4, 10, 15 und 16 gibt es derzeit keine PostgreSQL-Implementierungen.

PostgreSQL unterstützt die meisten wichtigen Funktionen von SQL:2023. Von 177 verpflichtenden Funktionen, die für vollständige Core-Konformität erforderlich sind, erfüllt PostgreSQL mindestens 170. Zusätzlich gibt es eine lange Liste unterstützter optionaler Funktionen. Bemerkenswert ist, dass zum Zeitpunkt der Erstellung keine aktuelle Version eines Datenbankmanagementsystems vollständige Konformität zu Core SQL:2023 beansprucht.

In den folgenden beiden Abschnitten wird zuerst eine Liste der von PostgreSQL unterstützten Funktionen gezeigt, danach eine Liste der in SQL:2023 definierten Funktionen, die PostgreSQL noch nicht unterstützt. Beide Listen sind Näherungen: Bei einer als unterstützt aufgeführten Funktion können kleine Details nicht konform sein, und von einer nicht unterstützten Funktion können große Teile tatsächlich implementiert sein. Der Hauptteil der Dokumentation enthält stets die genauesten Informationen darüber, was funktioniert und was nicht.

> **Hinweis:** Funktionscodes mit Bindestrich sind Unterfunktionen. Wenn eine bestimmte Unterfunktion nicht unterstützt wird, wird daher die Hauptfunktion als nicht unterstützt aufgeführt, selbst wenn einige andere Unterfunktionen unterstützt werden.

### D.1. Unterstützte Funktionen

| Kennung | Core? | Beschreibung | Kommentar |
| --- | --- | --- | --- |
| `B012` |  | Embedded C |  |
| `B021` |  | Direct SQL |  |
| `B128` |  | Routine language SQL |  |
| `E011` | Core | Numeric data types |  |
| `E011-01` | Core | INTEGER and SMALLINT data types |  |
| `E011-02` | Core | REAL, DOUBLE PRECISION, and FLOAT data types |  |
| `E011-03` | Core | DECIMAL and NUMERIC data types |  |
| `E011-04` | Core | Arithmetic operators |  |
| `E011-05` | Core | Numeric comparison |  |
| `E011-06` | Core | Implicit casting among the numeric data types |  |
| `E021` | Core | Character data types |  |
| `E021-01` | Core | CHARACTER data type |  |
| `E021-02` | Core | CHARACTER VARYING data type |  |
| `E021-03` | Core | Character literals |  |
| `E021-04` | Core | CHARACTER_LENGTH function | trims trailing spaces from CHARACTER values before counting |
| `E021-05` | Core | OCTET_LENGTH function |  |
| `E021-06` | Core | SUBSTRING function |  |
| `E021-07` | Core | Character concatenation |  |
| `E021-08` | Core | UPPER and LOWER functions |  |
| `E021-09` | Core | TRIM function |  |
| `E021-10` | Core | Implicit casting among the character string types |  |
| `E021-11` | ore | POSITION function |  |
| `E021-12` | ore | Character comparison |  |
| `E031` | ore | Identifiers |  |
| `E031-01` | ore | Delimited identifiers |  |
| `E031-02` | ore | Lower case identifiers |  |
| `E031-03` | ore | Trailing underscore |  |
| `E051` | ore | Basic query specification |  |
| `E051-01` | ore | SELECT DISTINCT |  |
| `E051-02` | ore | GROUP BY clause |  |
| `E051-04` | ore | GROUP BY can contain columns not in <select list> |  |
| `E051-05` | ore | Select list items can be renamed |  |
| `E051-06` | ore | HAVING clause |  |
| `E051-07` | ore | Qualified * in select list |  |
| `E051-08` | ore | Correlation names in the FROM clause |  |
| `E051-09` | ore | Rename columns in the FROM clause |  |
| `E061` | ore | Basic predicates and search conditions |  |
| `E061-01` | ore | Comparison predicate |  |
| `E061-02` | ore | BETWEEN predicate |  |
| `E061-03` | ore | IN predicate with list of values |  |
| `E061-04` | ore | LIKE predicate |  |
| `E061-05` | ore | LIKE predicate ESCAPE clause |  |
| `E061-06` | ore | NULL predicate |  |
| `E061-07` | ore | Quantified comparison predicate |  |
| `E061-08` | ore | EXISTS predicate |  |
| `E061-09` | ore | Subqueries in comparison predicate |  |
| `E061-11` | ore | Subqueries in IN predicate |  |
| `E061-12` | ore | Subqueries in quantified comparison predicate |  |
| `E061-13` | ore | Correlated subqueries |  |
| `E061-14` | ore | Search condition |  |
| `E071` | ore | Basic query expressions |  |
| `E071-01` | ore | UNION DISTINCT table operator |  |
| `E071-02` | ore | UNION ALL table operator |  |
| `E071-03` | ore | EXCEPT DISTINCT table operator |  |
| `E071-05` | ore | Columns combined via table operators need not have exactly the same data type |  |
| `E071-06` | ore | Table operators in subqueries |  |
| `E081` | ore | Basic Privileges |  |
| `E081-01` | ore | SELECT privilege |  |
| `E081-02` | ore | DELETE privilege |  |
| `E081-03` | ore | INSERT privilege at the table level |  |
| `E081-04` | ore | UPDATE privilege at the table level |  |
| `E081-05` | ore | UPDATE privilege at the column level |  |
| `E081-06` | ore | REFERENCES privilege at the table level |  |
| `E081-07` | ore | REFERENCES privilege at the column level |  |
| `E081-08` | ore | WITH GRANT OPTION |  |
| `E081-09` | ore | USAGE privilege |  |
| `E081-10` | ore | EXECUTE privilege |  |
| `E091` | ore | Set functions |  |
| `E091-01` | ore | AVG |  |
| `E091-02` | ore | COUNT |  |
| `E091-03` | ore | MAX |  |
| `E091-04` | ore | MIN |  |
| `E091-05` | ore | SUM |  |
| `E091-06` | ore | ALL quantifier |  |
| `E091-07` | ore | DISTINCT quantifier |  |
| `E101` | ore | Basic data manipulation |  |
| `E101-01` | ore | INSERT statement |  |
| `E101-03` | ore | Searched UPDATE statement |  |
| `E101-04` | ore | Searched DELETE statement |  |
| `E111` | ore | Single row SELECT statement |  |
| `E121` | ore | Basic cursor support |  |
| `E121-01` | ore | DECLARE CURSOR |  |
| `E121-02` | ore | ORDER BY columns need not be in select list |  |
| `E121-03` | ore | Value expressions in ORDER BY clause |  |
| `E121-04` | ore | OPEN statement |  |
| `E121-06` | ore | Positioned UPDATE statement |  |
| `E121-07` | ore | Positioned DELETE statement |  |
| `E121-08` | ore | CLOSE statement |  |
| `E121-10` | ore | FETCH statement implicit NEXT |  |
| `E121-17` | ore | WITH HOLD cursors |  |
| `E131` | ore | Null value support (nulls in lieu of values) |  |
| `E141` | ore | Basic integrity constraints |  |
| `E141-01` | ore | NOT NULL constraints |  |
| `E141-02` | ore | UNIQUE constraints of NOT NULL columns |  |
| `E141-03` | ore | PRIMARY KEY constraints |  |
| `E141-04` | ore | Basic FOREIGN KEY constraint with the NO ACTION default for both referential delete action and referential update action |  |
| `E141-06` | ore | CHECK constraints |  |
| `E141-07` | ore | Column defaults |  |
| `E141-08` | ore | NOT NULL inferred on PRIMARY KEY |  |
| `E141-10` | ore | Names in a foreign key can be specified in any order |  |
| `E151` | ore | Transaction support |  |
| `E151-01` | ore | COMMIT statement |  |
| `E151-02` | ore | ROLLBACK statement |  |
| `E152` | ore | Basic SET TRANSACTION statement |  |
| `E152-01` | ore | SET TRANSACTION statement: ISOLATION LEVEL SERIALIZABLE clause |  |
| `E152-02` | ore | SET TRANSACTION statement: READ ONLY and READ WRITE clauses |  |
| `E153` | ore | Updatable queries with subqueries |  |
| `E161` | ore | SQL comments using leading double minus |  |
| `E171` | ore | SQLSTATE support |  |
| `E182` | ore | Host language binding |  |
| `F021` | ore | Basic information schema |  |
| `F021-01` | ore | COLUMNS view |  |
| `F021-02` | ore | TABLES view |  |
| `F021-03` | ore | VIEWS view |  |
| `F021-04` | ore | TABLE_CONSTRAINTS view |  |
| `F021-05` | ore | REFERENTIAL_CONSTRAINTS view |  |
| `F021-06` | ore | CHECK_CONSTRAINTS view |  |
| `F031` | ore | Basic schema manipulation |  |
| `F031-01` | ore | CREATE TABLE statement to create persistent base tables |  |
| `F031-02` | ore | CREATE VIEW statement |  |
| `F031-03` | ore | GRANT statement |  |
| `F031-04` | ore | ALTER TABLE statement: ADD COLUMN clause |  |
| `F031-13` | ore | DROP TABLE statement: RESTRICT clause |  |
| `F031-16` | ore | DROP VIEW statement: RESTRICT clause |  |
| `F031-19` | ore | REVOKE statement: RESTRICT clause |  |
| `F032` |  | CASCADE drop behavior |  |
| `F033` |  | ALTER TABLE statement: DROP COLUMN clause |  |
| `F034` |  | Extended REVOKE statement |  |
| `F035` |  | REVOKE with CASCADE |  |
| `F036` |  | REVOKE statement performed by non-owner |  |
| `F037` |  | REVOKE statement: GRANT OPTION FOR clause |  |
| `F038` |  | REVOKE of a WITH GRANT OPTION privilege |  |
| `F041` | ore | Basic joined table |  |
| `F041-01` | ore | Inner join (but not necessarily the INNER keyword) |  |
| `F041-02` | ore | INNER keyword |  |
| `F041-03` | ore | LEFT OUTER JOIN |  |
| `F041-04` | ore | RIGHT OUTER JOIN |  |
| `F041-05` | ore | Outer joins can be nested |  |
| `F041-07` | ore | The inner table in a left or right outer join can also be used in an inner join |  |
| `F041-08` | ore | All comparison operators are supported (rather than just =) |  |
| `F051` | ore | Basic date and time |  |
| `F051-01` | ore | DATE data type (including support of DATE literal) |  |
| `F051-02` | ore | TIME data type (including support of TIME literal) with fractional seconds precision of at least 0 |  |
| `F051-03` | ore | TIMESTAMP data type (including support of TIMESTAMP literal) with fractional seconds precision of at least 0 and 6 |  |
| `F051-04` | ore | Comparison predicate on DATE, TIME, and TIMESTAMP data types |  |
| `F051-05` | ore | Explicit CAST between datetime types and character string types |  |
| `F051-06` | ore | CURRENT_DATE |  |
| `F051-07` | ore | LOCALTIME |  |
| `F051-08` | ore | LOCALTIMESTAMP |  |
| `F052` |  | Intervals and datetime arithmetic |  |
| `F053` |  | OVERLAPS predicate |  |
| `F081` | ore | UNION and EXCEPT in views |  |
| `F111` |  | Isolation levels other than SERIALIZABLE |  |
| `F112` |  | Isolation level READ UNCOMMITTED |  |
| `F113` |  | Isolation level READ COMMITTED |  |
| `F114` |  | Isolation level REPEATABLE READ |  |
| `F131` | ore | Grouped operations |  |
| `F131-01` | ore | WHERE, GROUP BY, and HAVING clauses supported in queries with grouped views |  |
| `F131-02` | ore | Multiple tables supported in queries with grouped views |  |
| `F131-03` | ore | Set functions supported in queries with grouped views |  |
| `F131-04` | ore | Subqueries with GROUP BY and HAVING clauses and grouped views |  |
| `F131-05` | ore | Single row SELECT with GROUP BY and HAVING clauses and grouped views |  |
| `F171` |  | Multiple schemas per user |  |
| `F181` | ore | Multiple module support |  |
| `F191` |  | Referential delete actions |  |
| `F200` |  | TRUNCATE TABLE statement |  |
| `F201` | ore | CAST function |  |
| `F202` |  | TRUNCATE TABLE: identity column restart option |  |
| `F221` | ore | Explicit defaults |  |
| `F222` |  | INSERT statement: DEFAULT VALUES clause |  |
| `F231` |  | Privilege tables |  |
| `F251` |  | Domain support |  |
| `F261` | ore | CASE expression |  |
| `F261-01` | ore | Simple CASE |  |
| `F261-02` | ore | Searched CASE |  |
| `F261-03` | ore | NULLIF |  |
| `F261-04` | ore | COALESCE |  |
| `F262` |  | Extended CASE expression |  |
| `F271` |  | Compound character literals |  |
| `F281` |  | LIKE enhancements |  |
| `F292` |  | UNIQUE null treatment |  |
| `F302` |  | INTERSECT table operator |  |
| `F303` |  | INTERSECT DISTINCT table operator |  |
| `F304` |  | EXCEPT ALL table operator |  |
| `F305` |  | INTERSECT ALL table operator |  |
| `F311` | ore | Schema definition statement |  |
| `F311-01` | ore | CREATE SCHEMA |  |
| `F311-02` | ore | CREATE TABLE for persistent base tables |  |
| `F311-03` | ore | CREATE VIEW |  |
| `F311-04` | ore | CREATE VIEW: WITH CHECK OPTION |  |
| `F311-05` | ore | GRANT statement |  |
| `F312` |  | MERGE statement |  |
| `F313` |  | Enhanced MERGE statement |  |
| `F314` |  | MERGE statement with DELETE branch |  |
| `F321` |  | User authorization |  |
| `F341` |  | Usage tables |  |
| `F361` |  | Subprogram support |  |
| `F381` |  | Extended schema manipulation |  |
| `F382` |  | Alter column data type |  |
| `F383` |  | Set column not null clause |  |
| `F384` |  | Drop identity property clause |  |
| `F385` |  | Drop column generation expression clause |  |
| `F386` |  | Set identity column generation clause |  |
| `F387` |  | ALTER TABLE statement: ALTER COLUMN clause |  |
| `F388` |  | ALTER TABLE statement: ADD/DROP CONSTRAINT clause |  |
| `F391` |  | Long identifiers |  |
| `F392` |  | Unicode escapes in identifiers |  |
| `F393` |  | Unicode escapes in literals |  |
| `F394` |  | Optional normal form specification |  |
| `F401` |  | Extended joined table |  |
| `F402` |  | Named column joins for LOBs, arrays, and multisets |  |
| `F404` |  | Range variable for common column names |  |
| `F405` |  | NATURAL JOIN |  |
| `F406` |  | FULL OUTER JOIN |  |
| `F407` |  | CROSS JOIN |  |
| `F411` |  | Time zone specification | differences regarding literal interpretation |
| `F421` |  | National character |  |
| `F431` |  | Read-only scrollable cursors |  |
| `F432` |  | FETCH with explicit NEXT |  |
| `F433` |  | FETCH FIRST |  |
| `F434` |  | FETCH LAST |  |
| `F435` |  | FETCH PRIOR |  |
| `F436` |  | FETCH ABSOLUTE |  |
| `F437` |  | FETCH RELATIVE |  |
| `F438` |  | Scrollable cursors |  |
| `F441` |  | Extended set function support |  |
| `F442` |  | Mixed column references in set functions |  |
| `F471` | ore | Scalar subquery values |  |
| `F481` | ore | Expanded NULL predicate |  |
| `F491` |  | Constraint management |  |
| `F492` |  | Optional table constraint enforcement | except not-null constraints |
| `F501` | ore | Features and conformance views |  |
| `F501-01` | ore | SQL_FEATURES view |  |
| `F501-02` | ore | SQL_SIZING view |  |
| `F502` |  | Enhanced documentation tables |  |
| `F531` |  | Temporary tables |  |
| `F555` |  | Enhanced seconds precision |  |
| `F561` |  | Full value expressions |  |
| `F571` |  | Truth value tests |  |
| `F591` |  | Derived tables |  |
| `F611` |  | Indicator data types |  |
| `F641` |  | Row and table constructors |  |
| `F651` |  | Catalog name qualifiers |  |
| `F661` |  | Simple tables |  |
| `F672` |  | Retrospective CHECK constraints |  |
| `F690` |  | Collation support |  |
| `F692` |  | Extended collation support |  |
| `F701` |  | Referential update actions |  |
| `F711` |  | ALTER domain |  |
| `F731` |  | INSERT column privileges |  |
| `F751` |  | View CHECK enhancements |  |
| `F761` |  | Session management |  |
| `F762` |  | CURRENT_CATALOG |  |
| `F763` |  | CURRENT_SCHEMA |  |
| `F771` |  | Connection management |  |
| `F781` |  | Self-referencing operations |  |
| `F791` |  | Insensitive cursors |  |
| `F801` |  | Full set function |  |
| `F850` |  | Top-level ORDER BY in query expression |  |
| `F851` |  | ORDER BY in subqueries |  |
| `F852` |  | Top-level ORDER BY in views |  |
| `F855` |  | Nested ORDER BY in query expression |  |
| `F856` |  | Nested FETCH FIRST in query expression |  |
| `F857` |  | Top-level FETCH FIRST in query expression |  |
| `F858` |  | FETCH FIRST in subqueries |  |
| `F859` |  | Top-level FETCH FIRST in views |  |
| `F860` |  | Dynamic FETCH FIRST row count |  |
| `F861` |  | Top-level OFFSET in query expression |  |
| `F862` |  | OFFSET in subqueries |  |
| `F863` |  | Nested OFFSET in query expression |  |
| `F864` |  | Top-level OFFSET in views |  |
| `F865` |  | Dynamic offset row count in OFFSET |  |
| `F867` |  | FETCH FIRST clause: WITH TIES option |  |
| `F868` |  | ORDER BY in grouped table |  |
| `F869` |  | SQL implementation info population |  |
| `S071` |  | SQL paths in function and type name resolution |  |
| `S090` |  | Minimal array support |  |
| `S092` |  | Arrays of user-defined types |  |
| `S095` |  | Array constructors by query |  |
| `S096` |  | Optional array bounds |  |
| `S098` |  | ARRAY_AGG |  |
| `S099` |  | Array expressions |  |
| `S111` |  | ONLY in query expressions |  |
| `S201` |  | SQL-invoked routines on arrays |  |
| `S203` |  | Array parameters |  |
| `S204` |  | Array as result type of functions |  |
| `S211` |  | User-defined cast functions |  |
| `S301` |  | Enhanced UNNEST |  |
| `S404` |  | TRIM_ARRAY |  |
| `T031` |  | BOOLEAN data type |  |
| `T054` |  | GREATEST and LEAST | different null handling |
| `T055` |  | String padding functions |  |
| `T056` |  | Multi-character TRIM functions |  |
| `T061` |  | UCS support |  |
| `T071` |  | BIGINT data type |  |
| `T081` |  | Optional string types maximum length |  |
| `T121` |  | WITH (excluding RECURSIVE) in query expression |  |
| `T122` |  | WITH (excluding RECURSIVE) in subquery |  |
| `T131` |  | Recursive query |  |
| `T132` |  | Recursive query in subquery |  |
| `T133` |  | Enhanced cycle mark values |  |
| `T141` |  | SIMILAR predicate |  |
| `T151` |  | DISTINCT predicate |  |
| `T152` |  | DISTINCT predicate with negation |  |
| `T171` |  | LIKE clause in table definition |  |
| `T172` |  | AS subquery clause in table definition |  |
| `T173` |  | Extended LIKE clause in table definition |  |
| `T174` |  | Identity columns |  |
| `T177` |  | Sequence generator support: simple restart option |  |
| `T178` |  | Identity columns: simple restart option |  |
| `T191` |  | Referential action RESTRICT |  |
| `T201` |  | Comparable data types for referential constraints |  |
| `T212` |  | Enhanced trigger capability |  |
| `T213` |  | INSTEAD OF triggers |  |
| `T214` |  | BEFORE triggers |  |
| `T215` |  | AFTER triggers |  |
| `T216` |  | Ability to require true search condition before trigger is invoked |  |
| `T217` |  | TRIGGER privilege |  |
| `T241` |  | START TRANSACTION statement |  |
| `T261` |  | Chained transactions |  |
| `T271` |  | Savepoints |  |
| `T281` |  | SELECT privilege with column granularity |  |
| `T285` |  | Enhanced derived column names |  |
| `T312` |  | OVERLAY function |  |
| `T321-01` | ore | User-defined functions with no overloading |  |
| `T321-02` | ore | User-defined stored procedures with no overloading |  |
| `T321-03` | ore | Function invocation |  |
| `T321-04` | ore | CALL statement |  |
| `T321-05` | ore | RETURN statement |  |
| `T321-06` | ore | ROUTINES view |  |
| `T321-07` | ore | PARAMETERS view |  |
| `T323` |  | Explicit security for external routines |  |
| `T325` |  | Qualified SQL parameter references |  |
| `T331` |  | Basic roles |  |
| `T332` |  | Extended roles |  |
| `T341` |  | Overloading of SQL-invoked functions and SQL-invoked procedures |  |
| `T351` |  | Bracketed comments |  |
| `T431` |  | Extended grouping capabilities |  |
| `T432` |  | Nested and concatenated GROUPING SETS |  |
| `T433` |  | Multi-argument GROUPING function |  |
| `T434` |  | GROUP BY DISTINCT |  |
| `T441` |  | ABS and MOD functions |  |
| `T461` |  | Symmetric BETWEEN predicate |  |
| `T491` |  | LATERAL derived table |  |
| `T501` |  | Enhanced EXISTS predicate |  |
| `T521` |  | Named arguments in CALL statement |  |
| `T523` |  | Default values for INOUT parameters of SQL-invoked procedures |  |
| `T524` |  | Named arguments in routine invocations other than a CALL statement |  |
| `T525` |  | Default values for parameters of SQL-invoked functions |  |
| `T551` |  | Optional key words for default syntax |  |
| `T581` |  | Regular expression substring function |  |
| `T591` |  | UNIQUE constraints of possibly null columns |  |
| `T611` |  | Elementary OLAP operations |  |
| `T612` |  | Advanced OLAP operations |  |
| `T613` |  | Sampling |  |
| `T614` |  | NTILE function |  |
| `T615` |  | LEAD and LAG functions |  |
| `T617` |  | FIRST_VALUE and LAST_VALUE functions |  |
| `T620` |  | WINDOW clause: GROUPS option |  |
| `T621` |  | Enhanced numeric functions |  |
| `T622` |  | Trigonometric functions |  |
| `T623` |  | General logarithm functions |  |
| `T624` |  | Common logarithm functions |  |
| `T626` |  | ANY_VALUE |  |
| `T627` |  | Window framed COUNT DISTINCT |  |
| `T631` | ore | IN predicate with one list element |  |
| `T651` |  | SQL-schema statements in SQL routines |  |
| `T653` |  | SQL-schema statements in external routines |  |
| `T655` |  | Cyclically dependent routines |  |
| `T661` |  | Non-decimal integer literals |  |
| `T662` |  | Underscores in numeric literals |  |
| `T670` |  | Schema and data statement mixing |  |
| `T803` |  | String-based JSON |  |
| `T811` |  | Basic SQL/JSON constructor functions |  |
| `T812` |  | SQL/JSON: JSON_OBJECTAGG |  |
| `T813` |  | SQL/JSON: JSON_ARRAYAGG with ORDER BY |  |
| `T814` |  | Colon in JSON_OBJECT or JSON_OBJECTAGG |  |
| `T821` |  | Basic SQL/JSON query operators |  |
| `T822` |  | SQL/JSON: IS JSON WITH UNIQUE KEYS predicate |  |
| `T823` |  | SQL/JSON: PASSING clause |  |
| `T825` |  | SQL/JSON: ON EMPTY and ON ERROR clauses |  |
| `T826` |  | General value expression in ON ERROR or ON EMPTY clauses |  |
| `T827` |  | JSON_TABLE: sibling NESTED COLUMNS clauses |  |
| `T828` |  | JSON_QUERY |  |
| `T829` |  | JSON_QUERY: array wrapper options |  |
| `T830` |  | Enforcing unique keys in SQL/JSON constructor functions |  |
| `T831` |  | SQL/JSON path language: strict mode |  |
| `T832` |  | SQL/JSON path language: item method |  |
| `T833` |  | SQL/JSON path language: multiple subscripts |  |
| `T834` |  | SQL/JSON path language: wildcard member accessor |  |
| `T835` |  | SQL/JSON path language: filter expressions |  |
| `T836` |  | SQL/JSON path language: starts with predicate |  |
| `T837` |  | SQL/JSON path language: regex_like predicate |  |
| `T840` |  | Hex integer literals in SQL/JSON path language |  |
| `T851` |  | SQL/JSON: optional keywords for default syntax |  |
| `T865` |  | SQL/JSON item method: bigint() |  |
| `T866` |  | SQL/JSON item method: boolean() |  |
| `T867` |  | SQL/JSON item method: date() |  |
| `T868` |  | SQL/JSON item method: decimal() |  |
| `T869` |  | SQL/JSON item method: decimal() with precision and scale |  |
| `T870` |  | SQL/JSON item method: integer() |  |
| `T871` |  | SQL/JSON item method: number() |  |
| `T872` |  | SQL/JSON item method: string() |  |
| `T873` |  | SQL/JSON item method: time() |  |
| `T874` |  | SQL/JSON item method: time_tz() |  |
| `T875` |  | SQL/JSON item method: time precision |  |
| `T876` |  | SQL/JSON item method: timestamp() |  |
| `T877` |  | SQL/JSON item method: timestamp_tz() |  |
| `T878` |  | SQL/JSON item method: timestamp precision |  |
| `T879` |  | JSON in equality operations | with jsonb |
| `T880` |  | JSON in grouping operations | with jsonb |
| `X010` |  | XML type |  |
| `X011` |  | Arrays of XML type |  |
| `X014` |  | Attributes of XML type |  |
| `X016` |  | Persistent XML values |  |
| `X020` |  | XMLConcat |  |
| `X031` |  | XMLElement |  |
| `X032` |  | XMLForest |  |
| `X034` |  | XMLAgg |  |
| `X035` |  | XMLAgg: ORDER BY option |  |
| `X036` |  | XMLComment |  |
| `X037` |  | XMLPI |  |
| `X038` |  | XMLText | supported except for RETURNING |
| `X040` |  | Basic table mapping |  |
| `X041` |  | Basic table mapping: null absent |  |
| `X042` |  | Basic table mapping: null as nil |  |
| `X043` |  | Basic table mapping: table as forest |  |
| `X044` |  | Basic table mapping: table as element |  |
| `X045` |  | Basic table mapping: with target namespace |  |
| `X046` |  | Basic table mapping: data mapping |  |
| `X047` |  | Basic table mapping: metadata mapping |  |
| `X048` |  | Basic table mapping: base64 encoding of binary strings |  |
| `X049` |  | Basic table mapping: hex encoding of binary strings |  |
| `X050` |  | Advanced table mapping |  |
| `X051` |  | Advanced table mapping: null absent |  |
| `X052` |  | Advanced table mapping: null as nil |  |
| `X053` |  | Advanced table mapping: table as forest |  |
| `X054` |  | Advanced table mapping: table as element |  |
| `X055` |  | Advanced table mapping: with target namespace |  |
| `X056` |  | Advanced table mapping: data mapping |  |
| `X057` |  | Advanced table mapping: metadata mapping |  |
| `X058` |  | Advanced table mapping: base64 encoding of binary strings |  |
| `X059` |  | Advanced table mapping: hex encoding of binary strings |  |
| `X060` |  | XMLParse: character string input and CONTENT option |  |
| `X061` |  | XMLParse: character string input and DOCUMENT option |  |
| `X069` |  | XMLSerialize: INDENT |  |
| `X070` |  | XMLSerialize: character string serialization and CONTENT option |  |
| `X071` |  | XMLSerialize: character string serialization and DOCUMENT option |  |
| `X072` |  | XMLSerialize: character string serialization |  |
| `X090` |  | XML document predicate |  |
| `X120` |  | XML parameters in SQL routines |  |
| `X121` |  | XML parameters in external routines |  |
| `X221` |  | XML passing mechanism BY VALUE |  |
| `X301` |  | XMLTable: derived column list option |  |
| `X302` |  | XMLTable: ordinality column option |  |
| `X303` |  | XMLTable: column default option |  |
| `X304` |  | XMLTable: passing a context item | must be XML DOCUMENT |
| `X400` |  | Name and identifier mapping |  |
| `X410` |  | Alter column data type: XML type |  |

### D.2. Nicht unterstützte Funktionen

Die folgenden in SQL:2023 definierten Funktionen sind in diesem PostgreSQL-Release nicht implementiert. In einigen Fällen ist äquivalente Funktionalität verfügbar.

| Kennung | Core? | Beschreibung | Kommentar |
| --- | --- | --- | --- |
| `B011` |  | Embedded Ada |  |
| `B013` |  | Embedded COBOL |  |
| `B014` |  | Embedded Fortran |  |
| `B015` |  | Embedded MUMPS |  |
| `B016` |  | Embedded Pascal |  |
| `B017` |  | Embedded PL/I |  |
| `B030` |  | Enhanced dynamic SQL |  |
| `B031` |  | Basic dynamic SQL |  |
| `B032` |  | Extended dynamic SQL |  |
| `B033` |  | Untyped SQL-invoked function arguments |  |
| `B034` |  | Dynamic specification of cursor attributes |  |
| `B035` |  | Non-extended descriptor names |  |
| `B036` |  | Describe input statement |  |
| `B041` |  | Extensions to embedded SQL exception declarations |  |
| `B051` |  | Enhanced execution rights |  |
| `B111` |  | Module language Ada |  |
| `B112` |  | Module language C |  |
| `B113` |  | Module language COBOL |  |
| `B114` |  | Module language Fortran |  |
| `B115` |  | Module language MUMPS |  |
| `B116` |  | Module language Pascal |  |
| `B117` |  | Module language PL/I |  |
| `B121` |  | Routine language Ada |  |
| `B122` |  | Routine language C |  |
| `B123` |  | Routine language COBOL |  |
| `B124` |  | Routine language Fortran |  |
| `B125` |  | Routine language MUMPS |  |
| `B126` |  | Routine language Pascal |  |
| `B127` |  | Routine language PL/I |  |
| `B200` |  | Polymorphic table functions |  |
| `B201` |  | More than one PTF generic table parameter |  |
| `B202` |  | PTF copartitioning |  |
| `B203` |  | More than one copartition specification |  |
| `B204` |  | PRUNE WHEN EMPTY |  |
| `B205` |  | Pass-through columns |  |
| `B206` |  | PTF descriptor parameters |  |
| `B207` |  | Cross products of partitionings |  |
| `B208` |  | PTF component procedure interface |  |
| `B209` |  | PTF extended names |  |
| `B211` |  | Module language Ada: VARCHAR and NUMERIC support |  |
| `B221` |  | Routine language Ada: VARCHAR and NUMERIC support |  |
| `F054` |  | TIMESTAMP in DATE type precedence list |  |
| `F120` |  | Get diagnostics statement |  |
| `F121` |  | Basic diagnostics management |  |
| `F122` |  | Enhanced diagnostics management |  |
| `F123` |  | All diagnostics |  |
| `F124` |  | SET TRANSACTION statement: DIAGNOSTICS SIZE clause |  |
| `F263` |  | Comma-separated predicates in simple CASE expression |  |
| `F291` |  | UNIQUE predicate |  |
| `F301` |  | CORRESPONDING in query expressions |  |
| `F403` |  | Partitioned join tables |  |
| `F451` |  | Character set definition |  |
| `F461` |  | Named character sets |  |
| `F521` |  | Assertions |  |
| `F671` |  | Subqueries in CHECK constraints | intentionally omitted |
| `F673` |  | Reads SQL-data routine invocations in CHECK constraints |  |
| `F693` |  | SQL-session and client module collations |  |
| `F695` |  | Translation support |  |
| `F696` |  | Additional translation documentation |  |
| `F721` |  | Deferrable constraints | foreign and unique keys only |
| `F741` |  | Referential MATCH types | no partial match yet |
| `F812` |  | Basic flagging |  |
| `F813` |  | Extended flagging |  |
| `F821` |  | Local table references |  |
| `F831` |  | Full cursor update |  |
| `F832` |  | Updatable scrollable cursors |  |
| `F833` |  | Updatable ordered cursors |  |
| `F841` |  | LIKE_REGEX predicate | consider regexp_like() |
| `F842` |  | OCCURRENCES_REGEX function | consider regexp_matches() |
| `F843` |  | POSITION_REGEX function | consider regexp_instr() |
| `F844` |  | SUBSTRING_REGEX function | consider regexp_substr() |
| `F845` |  | TRANSLATE_REGEX function | consider regexp_replace() |
| `F846` |  | Octet support in regular expression operators |  |
| `F847` |  | Non-constant regular expressions |  |
| `F866` |  | FETCH FIRST clause: PERCENT option |  |
| `R010` |  | Row pattern recognition: FROM clause |  |
| `R020` |  | Row pattern recognition: WINDOW clause |  |
| `R030` |  | Row pattern recognition: full aggregate support |  |
| `S011` | ore | Distinct data types |  |
| `S011-01` | ore | USER_DEFINED_TYPES view |  |
| `S023` |  | Basic structured types |  |
| `S024` |  | Enhanced structured types |  |
| `S025` |  | Final structured types |  |
| `S026` |  | Self-referencing structured types |  |
| `S027` |  | Create method by specific method name |  |
| `S028` |  | Permutable UDT options list |  |
| `S041` |  | Basic reference types |  |
| `S043` |  | Enhanced reference types |  |
| `S051` |  | Create table of type | partially supported |
| `S081` |  | Subtables |  |
| `S091` |  | Basic array support | partially supported |
| `S093` |  | Arrays of distinct types |  |
| `S094` |  | Arrays of reference types |  |
| `S097` |  | Array element assignment |  |
| `S151` |  | Type predicate | see pg_typeof() |
| `S161` |  | Subtype treatment |  |
| `S162` |  | Subtype treatment for references |  |
| `S202` |  | SQL-invoked routines on multisets |  |
| `S231` |  | Structured type locators |  |
| `S232` |  | Array locators |  |
| `S233` |  | Multiset locators |  |
| `S241` |  | Transform functions |  |
| `S242` |  | Alter transform statement |  |
| `S251` |  | User-defined orderings |  |
| `S261` |  | Specific type method |  |
| `S271` |  | Basic multiset support |  |
| `S272` |  | Multisets of user-defined types |  |
| `S274` |  | Multisets of reference types |  |
| `S275` |  | Advanced multiset support |  |
| `S281` |  | Nested collection types |  |
| `S291` |  | Unique constraint on entire row |  |
| `S401` |  | Distinct types based on array types |  |
| `S402` |  | Distinct types based on multiset types |  |
| `S403` |  | ARRAY_MAX_CARDINALITY |  |
| `T011` |  | Timestamp in Information Schema |  |
| `T021` |  | BINARY and VARBINARY data types |  |
| `T022` |  | Advanced support for BINARY and VARBINARY data types |  |
| `T023` |  | Compound binary literals |  |
| `T024` |  | Spaces in binary literals |  |
| `T039` |  | CLOB locator: non-holdable |  |
| `T040` |  | Concatenation of CLOBs |  |
| `T041` |  | Basic LOB data type support |  |
| `T042` |  | Extended LOB data type support |  |
| `T043` |  | Multiplier T |  |
| `T044` |  | Multiplier P |  |
| `T045` |  | BLOB data type |  |
| `T046` |  | CLOB data type |  |
| `T047` |  | POSITION, OCTET_LENGTH, TRIM, and SUBSTRING for BLOBs |  |
| `T048` |  | Concatenation of BLOBs |  |
| `T049` |  | BLOB locator: non-holdable |  |
| `T050` |  | POSITION, CHAR_LENGTH, OCTET_LENGTH, LOWER, TRIM, UPPER, and SUBSTRING for CLOBs |  |
| `T051` |  | Row types |  |
| `T053` |  | Explicit aliases for all-fields reference |  |
| `T062` |  | Character length units |  |
| `T076` |  | DECFLOAT data type |  |
| `T101` |  | Enhanced nullability determination |  |
| `T111` |  | Updatable joins, unions, and columns |  |
| `T175` |  | Generated columns | mostly supported |
| `T176` |  | Sequence generator support | supported except for NEXT VALUE FOR |
| `T180` |  | System-versioned tables |  |
| `T181` |  | Application-time period tables |  |
| `T200` |  | Trigger DDL | similar but not fully compatible |
| `T211` |  | Basic trigger capability |  |
| `T218` |  | Multiple triggers for the same event executed in the order created | intentionally omitted |
| `T231` |  | Sensitive cursors |  |
| `T251` |  | SET TRANSACTION statement: LOCAL option |  |
| `T262` |  | Multiple server transactions |  |
| `T272` |  | Enhanced savepoint management |  |
| `T301` |  | Functional dependencies | partially supported |
| `T321` | ore | Basic SQL-invoked routines | partially supported |
| `T322` |  | Declared data type attributes |  |
| `T324` |  | Explicit security for SQL routines |  |
| `T326` |  | Table functions |  |
| `T471` |  | Result sets return value |  |
| `T472` |  | DESCRIBE CURSOR |  |
| `T495` |  | Combined data change and retrieval | different syntax |
| `T502` |  | Period predicates |  |
| `T511` |  | Transaction counts |  |
| `T522` |  | Default values for IN parameters of SQL-invoked pro- sup cedures FAU inv | ported except DELT key word in ocation |
| `T561` |  | Holdable locators |  |
| `T571` |  | Array-returning external SQL-invoked functions |  |
| `T572` |  | Multiset-returning external SQL-invoked functions |  |
| `T601` |  | Local cursor references |  |
| `T616` |  | Null treatment option for LEAD and LAG functions |  |
| `T618` |  | NTH_VALUE function | function exists, but some options missing |
| `T619` |  | Nested window functions |  |
| `T625` |  | LISTAGG |  |
| `T641` |  | Multiple column assignment | only some syntax variants supported |
| `T652` |  | SQL-dynamic statements in SQL routines |  |
| `T654` |  | SQL-dynamic statements in external routines |  |
| `T801` |  | JSON data type |  |
| `T802` |  | Enhanced JSON data type |  |
| `T824` |  | JSON_TABLE: specific PLAN clause |  |
| `T838` |  | JSON_TABLE: PLAN DEFAULT clause |  |
| `T839` |  | Formatted cast of datetimes to/from character strings |  |
| `T860` |  | SQL/JSON simplified accessor: column reference only |  |
| `T861` |  | SQL/JSON simplified accessor: case-sensitive JSON member accessor |  |
| `T862` |  | SQL/JSON simplified accessor: wildcard member accessor |  |
| `T863` |  | SQL/JSON simplified accessor: single-quoted string literal as member accessor |  |
| `T864` |  | SQL/JSON simplified accessor |  |
| `T881` |  | JSON in ordering operations | with jsonb, partially supported |
| `T882` |  | JSON in multiset element grouping operations |  |
| `M001` |  | Datalinks |  |
| `M002` |  | Datalinks via SQL/CLI |  |
| `M003` |  | Datalinks via Embedded SQL |  |
| `M004` |  | Foreign data support | partially supported |
| `M005` |  | Foreign schema support |  |
| `M006` |  | GetSQLString routine |  |
| `M007` |  | TransmitRequest |  |
| `M009` |  | GetOpts and GetStatistics routines |  |
| `M010` |  | Foreign-data wrapper support | different API |
| `M011` |  | Datalinks via Ada |  |
| `M012` |  | Datalinks via C |  |
| `M013` |  | Datalinks via COBOL |  |
| `M014` |  | Datalinks via Fortran |  |
| `M015` |  | Datalinks via M |  |
| `M016` |  | Datalinks via Pascal |  |
| `M017` |  | Datalinks via PL/I |  |
| `M018` |  | Foreign-data wrapper interface routines in Ada |  |
| `M019` |  | Foreign-data wrapper interface routines in C | different API |
| `M020` |  | Foreign-data wrapper interface routines in COBOL |  |
| `M021` |  | Foreign-data wrapper interface routines in Fortran |  |
| `M022` |  | Foreign-data wrapper interface routines in MUMPS |  |
| `M023` |  | Foreign-data wrapper interface routines in Pascal |  |
| `M024` |  | Foreign-data wrapper interface routines in PL/I |  |
| `M030` |  | SQL-server foreign data support |  |
| `M031` |  | Foreign-data wrapper general routines |  |
| `X012` |  | Multisets of XML type |  |
| `X013` |  | Distinct types of XML type |  |
| `X015` |  | Fields of XML type |  |
| `X025` |  | XMLCast |  |
| `X030` |  | XMLDocument |  |
| `X065` |  | XMLParse: binary string input and CONTENT option |  |
| `X066` |  | XMLParse: binary string input and DOCUMENT option |  |
| `X068` |  | XMLSerialize: BOM |  |
| `X073` |  | XMLSerialize: binary string serialization and CONTENT option |  |
| `X074` |  | XMLSerialize: binary string serialization and DOCUMENT option |  |
| `X075` |  | XMLSerialize: binary string serialization |  |
| `X076` |  | XMLSerialize: VERSION |  |
| `X077` |  | XMLSerialize: explicit ENCODING option |  |
| `X078` |  | XMLSerialize: explicit XML declaration |  |
| `X080` |  | Namespaces in XML publishing |  |
| `X081` |  | Query-level XML namespace declarations |  |
| `X082` |  | XML namespace declarations in DML |  |
| `X083` |  | XML namespace declarations in DDL |  |
| `X084` |  | XML namespace declarations in compound statements |  |
| `X085` |  | Predefined namespace prefixes |  |
| `X086` |  | XML namespace declarations in XMLTable |  |
| `X091` |  | XML content predicate |  |
| `X096` |  | XMLExists | XPath 1.0 only |
| `X100` |  | Host language support for XML: CONTENT option |  |
| `X101` |  | Host language support for XML: DOCUMENT option |  |
| `X110` |  | Host language support for XML: VARCHAR mapping |  |
| `X111` |  | Host language support for XML: CLOB mapping |  |
| `X112` |  | Host language support for XML: BLOB mapping |  |
| `X113` |  | Host language support for XML: STRIP WHITESPACE option |  |
| `X114` |  | Host language support for XML: PRESERVE WHITESPACE option |  |
| `X131` |  | Query-level XMLBINARY clause |  |
| `X132` |  | XMLBINARY clause in DML |  |
| `X133` |  | XMLBINARY clause in DDL |  |
| `X134` |  | XMLBINARY clause in compound statements |  |
| `X135` |  | XMLBINARY clause in subqueries |  |
| `X141` |  | IS VALID predicate: data-driven case |  |
| `X142` |  | IS VALID predicate: ACCORDING TO clause |  |
| `X143` |  | IS VALID predicate: ELEMENT clause |  |
| `X144` |  | IS VALID predicate: schema location |  |
| `X145` |  | IS VALID predicate outside check constraints |  |
| `X151` |  | IS VALID predicate: with DOCUMENT option |  |
| `X152` |  | IS VALID predicate: with CONTENT option |  |
| `X153` |  | IS VALID predicate: with SEQUENCE option |  |
| `X155` |  | IS VALID predicate: NAMESPACE without ELEMENT clause |  |
| `X157` |  | IS VALID predicate: NO NAMESPACE with ELEMENT clause |  |
| `X160` |  | Basic Information Schema for registered XML schemas |  |
| `X161` |  | Advanced Information Schema for registered XML schemas |  |
| `X170` |  | XML null handling options |  |
| `X171` |  | NIL ON NO CONTENT option |  |
| `X181` |  | XML(DOCUMENT(UNTYPED)) type |  |
| `X182` |  | XML(DOCUMENT(ANY)) type |  |
| `X190` |  | XML(SEQUENCE) type |  |
| `X191` |  | XML(DOCUMENT(XMLSCHEMA)) type |  |
| `X192` |  | XML(CONTENT(XMLSCHEMA)) type |  |
| `X200` |  | XMLQuery |  |
| `X201` |  | XMLQuery: RETURNING CONTENT |  |
| `X202` |  | XMLQuery: RETURNING SEQUENCE |  |
| `X203` |  | XMLQuery: passing a context item |  |
| `X204` |  | XMLQuery: initializing an XQuery variable |  |
| `X205` |  | XMLQuery: EMPTY ON EMPTY option |  |
| `X206` |  | XMLQuery: NULL ON EMPTY option |  |
| `X211` |  | XML 1.1 support |  |
| `X222` |  | XML passing mechanism BY REF | parser accepts BY REF but ignores it; passing is always BY VALUE |
| `X231` |  | XML(CONTENT(UNTYPED)) type |  |
| `X232` |  | XML(CONTENT(ANY)) type |  |
| `X241` |  | RETURNING CONTENT in XML publishing |  |
| `X242` |  | RETURNING SEQUENCE in XML publishing |  |
| `X251` |  | Persistent XML values of XML(DOCUMENT(UNTYPED)) type |  |
| `X252` |  | Persistent XML values of XML(DOCUMENT(ANY)) type |  |
| `X253` |  | Persistent XML values of XML(CONTENT(UNTYPED)) type |  |
| `X254` |  | Persistent XML values of XML(CONTENT(ANY)) type |  |
| `X255` |  | Persistent XML values of XML(SEQUENCE) type |  |
| `X256` |  | Persistent XML values of XML(DOCUMENT(XMLSCHEMA)) type |  |
| `X257` |  | Persistent XML values of XML(CONTENT(XMLSCHEMA)) type |  |
| `X260` |  | XML type: ELEMENT clause |  |
| `X261` |  | XML type: NAMESPACE without ELEMENT clause |  |
| `X263` |  | XML type: NO NAMESPACE with ELEMENT clause |  |
| `X264` |  | XML type: schema location |  |
| `X271` |  | XMLValidate: data-driven case |  |
| `X272` |  | XMLValidate: ACCORDING TO clause |  |
| `X273` |  | XMLValidate: ELEMENT clause |  |
| `X274` |  | XMLValidate: schema location |  |
| `X281` |  | XMLValidate with DOCUMENT option |  |
| `X282` |  | XMLValidate with CONTENT option |  |
| `X283` |  | XMLValidate with SEQUENCE option |  |
| `X284` |  | XMLValidate: NAMESPACE without ELEMENT clause |  |
| `X286` |  | XMLValidate: NO NAMESPACE with ELEMENT clause |  |
| `X300` |  | XMLTable | XPath 1.0 only |
| `X305` |  | XMLTable: initializing an XQuery variable |  |

### D.3. XML-Grenzen und Konformität zu SQL/XML

Mit SQL:2006 wurden erhebliche Überarbeitungen an den XML-bezogenen Spezifikationen in ISO/IEC 9075-14 (SQL/XML) eingeführt. PostgreSQLs Implementierung des Datentyps `xml` und der zugehörigen Funktionen folgt weitgehend der früheren Ausgabe von 2003, mit einigen Übernahmen aus späteren Ausgaben. Insbesondere gilt:

- Wo der aktuelle Standard eine Familie von XML-Datentypen bereitstellt, um „document“ oder „content“ in untypisierten oder XML-Schema-typisierten Varianten zu halten, sowie einen Typ `XML(SEQUENCE)` für beliebige XML-Inhaltsstücke, stellt PostgreSQL den einzelnen Typ `xml` bereit, der „document“ oder „content“ halten kann. Es gibt kein Gegenstück zum „sequence“-Typ des Standards.

- PostgreSQL stellt zwei mit SQL:2006 eingeführte Funktionen bereit, jedoch in Varianten, die die Sprache XPath 1.0 verwenden, statt XML Query, wie es der Standard für sie spezifiziert.

- PostgreSQL unterstützt die Klauseln `RETURNING CONTENT` und `RETURNING SEQUENCE` nicht; Funktionen, die laut Spezifikation solche Klauseln haben, geben implizit Content zurück.

Dieser Abschnitt stellt einige der daraus entstehenden Unterschiede vor, auf die Sie stoßen können.

#### D.3.1. Abfragen sind auf XPath 1.0 beschränkt

Die PostgreSQL-spezifischen Funktionen `xpath()` und `xpath_exists()` fragen XML-Dokumente mit der Sprache XPath ab. PostgreSQL stellt außerdem XPath-only-Varianten der Standardfunktionen `XMLEXISTS` und `XMLTABLE` bereit, die offiziell die Sprache XQuery verwenden. Für alle diese Funktionen stützt sich PostgreSQL auf die Bibliothek `libxml2`, die nur XPath 1.0 bereitstellt.

Zwischen der Sprache XQuery und XPath ab Version 2.0 gibt es eine starke Verbindung: Jeder Ausdruck, der in beiden syntaktisch gültig ist und erfolgreich ausgeführt wird, liefert dasselbe Ergebnis, mit einer kleinen Ausnahme für Ausdrücke mit numerischen Zeichenreferenzen oder vordefinierten Entitätsreferenzen, die XQuery durch das entsprechende Zeichen ersetzt, während XPath sie unverändert lässt. Zwischen diesen Sprachen und XPath 1.0 gibt es keine solche Verbindung; XPath 1.0 ist eine frühere Sprache und unterscheidet sich in vielerlei Hinsicht.

Es sind zwei Kategorien von Einschränkungen zu beachten: die Einschränkung von XQuery auf XPath für die im SQL-Standard spezifizierten Funktionen und die Einschränkung von XPath auf Version 1.0 sowohl für die Standardfunktionen als auch für die PostgreSQL-spezifischen Funktionen.

##### D.3.1.1. Einschränkung von XQuery auf XPath

Funktionen von XQuery, die über XPath hinausgehen, umfassen:

- XQuery-Ausdrücke können neue XML-Knoten konstruieren und zurückgeben, zusätzlich zu allen möglichen XPath-Werten. XPath kann Werte atomarer Typen erzeugen und zurückgeben, etwa Zahlen und Zeichenketten, aber nur XML-Knoten zurückgeben, die bereits in den als Eingabe gelieferten Dokumenten vorhanden waren.
- XQuery hat Steuerkonstrukte für Iteration, Sortierung und Gruppierung.
- XQuery erlaubt die Deklaration und Verwendung lokaler Funktionen.

Neuere XPath-Versionen beginnen, sich mit diesen Fähigkeiten zu überschneiden, zum Beispiel durch funktionales `for-each` und Sortieren, anonyme Funktionen und `parse-xml`, um aus einer Zeichenkette einen Knoten zu erzeugen. Solche Funktionen waren vor XPath 3.0 jedoch nicht verfügbar.

##### D.3.1.2. Einschränkung von XPath auf 1.0

Für Entwickler, die mit XQuery und XPath 2.0 oder später vertraut sind, bringt XPath 1.0 eine Reihe von Unterschieden mit sich:

- Der grundlegende Typ eines XQuery/XPath-Ausdrucks, die Sequenz, die XML-Knoten, atomare Werte oder beides enthalten kann, existiert in XPath 1.0 nicht. Ein 1.0-Ausdruck kann nur eine Knotenmenge mit null oder mehr XML-Knoten oder einen einzelnen atomaren Wert erzeugen.
- Anders als eine XQuery/XPath-Sequenz, die beliebige Items in beliebiger Reihenfolge enthalten kann, hat eine XPath-1.0-Knotenmenge keine garantierte Reihenfolge und erlaubt wie jede Menge nicht, dass dasselbe Item mehrfach vorkommt.

> **Hinweis:** Die Bibliothek `libxml2` scheint Knotenmenge an PostgreSQL stets in derselben relativen Reihenfolge zurückzugeben, in der ihre Mitglieder im Eingabedokument standen. Ihre Dokumentation verpflichtet sich jedoch nicht auf dieses Verhalten, und ein XPath-1.0-Ausdruck kann es nicht steuern.

- Während XQuery/XPath alle in XML Schema definierten Typen sowie viele Operatoren und Funktionen über diesen Typen bereitstellt, hat XPath 1.0 nur Knotenmenge und die drei atomaren Typen `boolean`, `double` und `string`.
- XPath 1.0 hat keinen bedingten Operator. Ein XQuery/XPath-Ausdruck wie `if ( hat ) then hat/@size else "no hat"` hat kein XPath-1.0-Gegenstück.
- XPath 1.0 hat keinen Ordnungsvergleichsoperator für Zeichenketten. Sowohl `"cat" < "dog"` als auch `"cat" > "dog"` sind falsch, weil beides numerische Vergleiche zweier NaNs sind. Dagegen vergleichen `=` und `!=` die Zeichenketten als Zeichenketten.
- XPath 1.0 verwischt die Unterscheidung zwischen Wertvergleichen und allgemeinen Vergleichen, wie XQuery/XPath sie definieren. Sowohl `sale/@hatsize = 7` als auch `sale/@customer = "alice"` sind existenziell quantifizierte Vergleiche, die wahr sind, wenn es irgendeinen `sale` mit dem angegebenen Attributwert gibt. `sale/@taxable = false()` dagegen ist ein Wertvergleich mit dem effektiven booleschen Wert einer ganzen Knotenmenge. Er ist nur dann wahr, wenn kein `sale` überhaupt ein `taxable`-Attribut hat.
- Im XQuery/XPath-Datenmodell kann ein Dokumentknoten entweder Dokumentform haben, also genau ein Top-Level-Element mit nur Kommentaren und Processing Instructions außerhalb davon, oder Content-Form, bei der diese Einschränkungen gelockert sind. Sein Gegenstück in XPath 1.0, der Wurzelknoten, kann nur Dokumentform haben. Das ist ein Teil des Grundes, warum ein `xml`-Wert, der einer XPath-basierten PostgreSQL-Funktion als Kontextitem übergeben wird, Dokumentform haben muss.

Die hier hervorgehobenen Unterschiede sind nicht vollständig. In XQuery und den Versionen 2.0 und später von XPath gibt es einen XPath-1.0-Kompatibilitätsmodus, und die W3C-Listen zu Änderungen der Funktionsbibliothek und Sprachänderungen in diesem Modus geben eine vollständigere, wenn auch immer noch nicht erschöpfende, Darstellung der Unterschiede. Der Kompatibilitätsmodus kann die späteren Sprachen nicht exakt äquivalent zu XPath 1.0 machen.

##### D.3.1.3. Abbildungen zwischen SQL- und XML-Datentypen und -Werten

In SQL:2006 und später sind beide Konvertierungsrichtungen zwischen Standard-SQL-Datentypen und XML-Schema-Typen genau spezifiziert. Die Regeln sind jedoch mit den Typen und der Semantik von XQuery/XPath ausgedrückt und lassen sich nicht direkt auf das andere Datenmodell von XPath 1.0 anwenden.

Wenn PostgreSQL SQL-Datenwerte nach XML abbildet, etwa in `xmlelement`, oder XML nach SQL, etwa in den Ausgabespalten von `xmltable`, nimmt PostgreSQL, abgesehen von einigen speziell behandelten Fällen, einfach an, dass die XPath-1.0-Stringform des XML-Datentyps als Texteingabeform des SQL-Datentyps gültig ist, und umgekehrt. Diese Regel hat den Vorteil der Einfachheit und liefert für viele Datentypen Ergebnisse, die den im Standard spezifizierten Abbildungen ähneln.

Wenn Interoperabilität mit anderen Systemen wichtig ist, kann es für einige Datentypen nötig sein, Datentyp-Formatierungsfunktionen wie die in [Abschnitt 9.8](09_Funktionen_und_Operatoren.md#98-formatierungsfunktionen-für-datentypen) ausdrücklich zu verwenden, um die Standardabbildungen zu erzeugen.

#### D.3.2. Beiläufige Grenzen der Implementierung

Dieser Abschnitt betrifft Grenzen, die nicht der Bibliothek `libxml2` innewohnen, sondern für die aktuelle Implementierung in PostgreSQL gelten.

##### D.3.2.1. Nur der Übergabemechanismus BY VALUE wird unterstützt

Der SQL-Standard definiert zwei Übergabemechanismen, die beim Übergeben eines XML-Arguments von SQL an eine XML-Funktion oder beim Empfangen eines Ergebnisses gelten: `BY REF`, bei dem ein bestimmter XML-Wert seine Knotenidentität behält, und `BY VALUE`, bei dem der Inhalt des XML übergeben wird, die Knotenidentität aber nicht erhalten bleibt. Ein Mechanismus kann vor einer Parameterliste als Standardmechanismus für alle Parameter angegeben werden oder nach einem einzelnen Parameter, um den Standard zu überschreiben.

Zur Veranschaulichung des Unterschieds würden diese beiden Abfragen, wenn `x` ein XML-Wert ist, in einer SQL:2006-Umgebung `true` bzw. `false` liefern:

```sql
SELECT XMLQUERY('$a is $b' PASSING BY REF x AS a, x AS b NULL ON EMPTY);
SELECT XMLQUERY('$a is $b' PASSING BY VALUE x AS a, x AS b NULL ON EMPTY);
```

PostgreSQL akzeptiert `BY VALUE` oder `BY REF` in einem `XMLEXISTS`- oder `XMLTABLE`-Konstrukt, ignoriert sie aber. Der Datentyp `xml` hält eine serialisierte Zeichenkettenrepräsentation, daher gibt es keine Knotenidentität zu bewahren, und Übergabe ist effektiv immer `BY VALUE`.

##### D.3.2.2. Benannte Parameter können nicht an Abfragen übergeben werden

Die XPath-basierten Funktionen unterstützen die Übergabe eines Parameters, der als Kontextitem des XPath-Ausdrucks dient, unterstützen aber keine zusätzlichen Werte, die dem Ausdruck als benannte Parameter zur Verfügung stehen.

##### D.3.2.3. Kein Typ XML(SEQUENCE)

Der PostgreSQL-Datentyp `xml` kann nur einen Wert in `DOCUMENT`- oder `CONTENT`-Form halten. Ein Kontextitem eines XQuery/XPath-Ausdrucks muss ein einzelner XML-Knoten oder atomarer Wert sein, aber XPath 1.0 schränkt dies weiter auf einen XML-Knoten ein und hat keinen Knotentyp, der `CONTENT` erlaubt. Das Ergebnis ist, dass ein wohlgeformtes `DOCUMENT` die einzige Form von XML-Wert ist, die PostgreSQL als XPath-Kontextitem bereitstellen kann.
