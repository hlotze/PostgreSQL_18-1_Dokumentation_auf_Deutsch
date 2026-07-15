# Anhang C. SQL-Schlüsselwörter

Tabelle C.1 listet alle Tokens auf, die im SQL-Standard und in PostgreSQL 18.1 Schlüsselwörter sind. Hintergrundinformationen finden sich in Abschnitt 4.1.1. Aus Platzgründen sind nur die beiden neuesten Versionen des SQL-Standards sowie SQL-92 zum historischen Vergleich enthalten. Die Unterschiede zwischen diesen und den anderen dazwischenliegenden Standardversionen sind klein.

SQL unterscheidet zwischen reservierten und nicht reservierten Schlüsselwörtern. Nach dem Standard sind reservierte Schlüsselwörter die einzigen echten Schlüsselwörter; sie dürfen niemals als Bezeichner verwendet werden. Nicht reservierte Schlüsselwörter haben nur in bestimmten Kontexten eine besondere Bedeutung und können in anderen Kontexten als Bezeichner verwendet werden. Die meisten nicht reservierten Schlüsselwörter sind tatsächlich Namen eingebauter Tabellen und Funktionen, die SQL spezifiziert. Das Konzept nicht reservierter Schlüsselwörter existiert im Wesentlichen nur, um festzuhalten, dass einem Wort in einigen Kontexten eine vordefinierte Bedeutung zugeordnet ist.

Im PostgreSQL-Parser ist die Lage etwas komplizierter. Es gibt mehrere Klassen von Tokens, von solchen, die niemals als Bezeichner verwendet werden können, bis zu solchen, die im Parser keinerlei besonderen Status haben, aber als gewöhnliche Bezeichner betrachtet werden. Letzteres ist normalerweise bei Funktionen der Fall, die SQL spezifiziert. Selbst reservierte Schlüsselwörter sind in PostgreSQL nicht vollständig reserviert, sondern können als Spaltenlabels verwendet werden, zum Beispiel `SELECT 55 AS CHECK`, obwohl `CHECK` ein reserviertes Schlüsselwort ist.

In Tabelle C.1 klassifiziert die PostgreSQL-Spalte als „non-reserved“ jene Schlüsselwörter, die dem Parser ausdrücklich bekannt sind, aber als Spalten- oder Tabellennamen erlaubt sind. Einige ansonsten nicht reservierte Schlüsselwörter können nicht als Funktions- oder Datentypnamen verwendet werden und sind entsprechend markiert. Die meisten dieser Wörter stehen für eingebaute Funktionen oder Datentypen mit besonderer Syntax. Die Funktion oder der Typ ist weiterhin verfügbar, kann aber vom Benutzer nicht neu definiert werden. Als „reserved“ markiert sind jene Tokens, die nicht als Spalten- oder Tabellennamen erlaubt sind. Einige reservierte Schlüsselwörter sind als Namen für Funktionen oder Datentypen erlaubt; auch das zeigt die Tabelle. Wenn dies nicht markiert ist, ist ein reserviertes Schlüsselwort nur als Spaltenlabel erlaubt. Ein leerer Eintrag in dieser Spalte bedeutet, dass PostgreSQL das Wort als gewöhnlichen Bezeichner behandelt.

Außerdem können die meisten Schlüsselwörter als bloße Spaltenlabels verwendet werden, ohne davor `AS` zu schreiben, wie in Abschnitt 7.3.2 beschrieben. Einige wenige erfordern jedoch ein führendes `AS`, um Mehrdeutigkeit zu vermeiden. Diese sind in der Tabelle als „requires AS“ markiert.

Als allgemeine Regel gilt: Wenn Sie unerwartete Parserfehler bei Befehlen erhalten, die eines der aufgeführten Schlüsselwörter als Bezeichner verwenden, sollten Sie versuchen, den Bezeichner zu quoten, um zu prüfen, ob das Problem dadurch verschwindet.

Vor dem Studium von Tabelle C.1 ist wichtig zu verstehen, dass ein in PostgreSQL nicht reserviertes Schlüsselwort nicht bedeutet, dass die zugehörige Funktion nicht implementiert ist. Umgekehrt zeigt das Vorhandensein eines Schlüsselworts nicht an, dass eine Funktion existiert.

## Tabelle C.1. SQL-Schlüsselwörter

### A

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `A` |  | nicht reserviert | nicht reserviert |  |
| `ABORT` | nicht reserviert |  |  |  |
| `ABS` |  | reserviert | reserviert |  |
| `ABSENT` | nicht reserviert | reserviert | reserviert |  |
| `ABSOLUTE` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `ACCESS` | nicht reserviert |  |  |  |
| `ACCORDING` |  | nicht reserviert | nicht reserviert |  |
| `ACOS` |  | reserviert | reserviert |  |
| `ACTION` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `ADD` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `ADMIN` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `AFTER` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `AGGREGATE` | nicht reserviert |  |  |  |
| `ALL` | reserviert | reserviert | reserviert | reserviert |
| `ALLOCATE` |  | reserviert | reserviert | reserviert |
| `ALSO` | nicht reserviert |  |  |  |
| `ALTER` | nicht reserviert | reserviert | reserviert | reserviert |
| `ALWAYS` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `ANALYSE` | reserviert |  |  |  |
| `ANALYZE` | reserviert |  |  |  |
| `AND` | reserviert | reserviert | reserviert | reserviert |
| `ANY` | reserviert | reserviert | reserviert | reserviert |
| `ANY_VALUE` |  | reserviert |  |  |
| `ARE` |  | reserviert | reserviert | reserviert |
| `ARRAY` | reserviert; erfordert `AS` | reserviert | reserviert |  |
| `ARRAY_AGG` |  | reserviert | reserviert |  |
| `ARRAY_MAX_CARDINALITY` |  | reserviert | reserviert |  |
| `AS` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `ASC` | reserviert | nicht reserviert | nicht reserviert | reserviert |
| `ASENSITIVE` | nicht reserviert | reserviert | reserviert |  |
| `ASIN` |  | reserviert | reserviert |  |
| `ASSERTION` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `ASSIGNMENT` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `ASYMMETRIC` | reserviert | reserviert | reserviert |  |
| `AT` | nicht reserviert | reserviert | reserviert | reserviert |
| `ATAN` |  | reserviert | reserviert |  |
| `ATOMIC` | nicht reserviert | reserviert | reserviert |  |
| `ATTACH` | nicht reserviert |  |  |  |
| `ATTRIBUTE` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `ATTRIBUTES` |  | nicht reserviert | nicht reserviert |  |
| `AUTHORIZATION` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `AVG` |  | reserviert | reserviert | reserviert |

### B

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `BACKWARD` | nicht reserviert |  |  |  |
| `BASE64` |  | nicht reserviert | nicht reserviert |  |
| `BEFORE` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `BEGIN` | nicht reserviert | reserviert | reserviert | reserviert |
| `BEGIN_FRAME` |  | reserviert | reserviert |  |
| `BERNOULLI` |  | nicht reserviert | nicht reserviert |  |
| `BETWEEN` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `BIGINT` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `BINARY` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert |  |
| `BIT` | nicht reserviert; kann nicht Funktion oder Typ sein |  |  | reserviert |
| `BIT_LENGTH` |  |  |  | reserviert |
| `BLOB` |  | reserviert | reserviert |  |
| `BLOCKED` |  | nicht reserviert | nicht reserviert |  |
| `BOM` |  | nicht reserviert | nicht reserviert |  |
| `BOOLEAN` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `BOTH` | reserviert | reserviert | reserviert | reserviert |
| `BREADTH` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `BTRIM` |  | reserviert |  |  |
| `BY` | nicht reserviert | reserviert | reserviert | reserviert |

### C

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `C` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `CACHE` | nicht reserviert |  |  |  |
| `CALL` | nicht reserviert | reserviert | reserviert |  |
| `CALLED` | nicht reserviert | reserviert | reserviert |  |
| `CARDINALITY` |  | reserviert | reserviert |  |
| `CASCADE` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `CASCADED` | nicht reserviert | reserviert | reserviert | reserviert |
| `CASE` | reserviert | reserviert | reserviert | reserviert |
| `CAST` | reserviert | reserviert | reserviert | reserviert |
| `CATALOG` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `CATALOG_NAME` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `CEIL` |  | reserviert | reserviert |  |
| `CEILING` |  | reserviert | reserviert |  |
| `CHAIN` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `CHAINING` |  | nicht reserviert | nicht reserviert |  |
| `CHAR` | nicht reserviert; kann nicht Funktion oder Typ sein; erfordert `AS` | reserviert | reserviert | reserviert |
| `CHARACTERISTICS` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `CHARACTERS` |  | nicht reserviert | nicht reserviert |  |
| `CHARACTER_LENGTH` |  | reserviert | reserviert | reserviert |
| `CHARACTER_SET_CATALOG` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `CHARACTER_SET_NAME` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `CHARACTER_SET_SCHEMA` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `CHAR_LENGTH` |  | reserviert | reserviert | reserviert |
| `CHECK` | reserviert | reserviert | reserviert | reserviert |
| `CHECKPOINT` | nicht reserviert |  |  |  |
| `CLASS` | nicht reserviert |  |  |  |
| `CLASSIFIER` |  | reserviert | reserviert |  |
| `CLASS_ORIGIN` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `CLOB` |  | reserviert | reserviert |  |
| `CLOSE` | nicht reserviert | reserviert | reserviert | reserviert |
| `CLUSTER` | nicht reserviert |  |  |  |
| `COALESCE` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `COBOL` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `COLLATE` | reserviert | reserviert | reserviert | reserviert |
| `COLLATION` | reserviert; kann Funktion oder Typ sein | nicht reserviert | nicht reserviert | reserviert |
| `COLLATION_CATALOG` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `COLLATION_NAME` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `COLLATION_SCHEMA` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `COLLECT` |  | reserviert | reserviert |  |
| `COLUMN` | reserviert | reserviert | reserviert | reserviert |
| `COLUMNS` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `COLUMN_NAME` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `COMMAND_FUNCTION` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `COMMAND_FUNCTION_CODE` |  | nicht reserviert | nicht reserviert |  |
| `COMMENT` | nicht reserviert |  |  |  |
| `COMMENTS` | nicht reserviert |  |  |  |
| `COMMIT` | nicht reserviert | reserviert | reserviert | reserviert |
| `COMMITTED` | nicht reserviert | nicht reserviert | nicht reserviert | nicht reserviert |
| `COMPRESSION` | nicht reserviert |  |  |  |
| `CONCURRENTLY` | reserviert; kann Funktion oder Typ sein |  |  |  |
| `CONDITION` |  | reserviert | reserviert |  |
| `CONDITIONAL` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `CONDITION_NUMBER` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `CONFLICT` | nicht reserviert |  |  |  |
| `CONNECT` |  | reserviert | reserviert | reserviert |
| `CONNECTION` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `CONNECTION_NAME` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `CONSTRAINT` | reserviert | reserviert | reserviert | reserviert |
| `CONSTRAINTS` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `CONSTRAINT_CATALOG` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `CONSTRAINT_NAME` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `CONSTRAINT_SCHEMA` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `CONSTRUCTOR` |  | nicht reserviert | nicht reserviert |  |
| `CONTAINS` |  | reserviert | reserviert |  |
| `CONTENT` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `CONTINUE` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `CONTROL` |  | nicht reserviert | nicht reserviert |  |
| `CONVERSION` | nicht reserviert |  |  |  |
| `CONVERT` |  | reserviert | reserviert | reserviert |
| `COPARTITION` |  | nicht reserviert |  |  |
| `COPY` | nicht reserviert | reserviert | reserviert |  |
| `CORR` |  | reserviert | reserviert |  |
| `CORRESPONDING` |  | reserviert | reserviert | reserviert |
| `COS` |  | reserviert | reserviert |  |
| `COSH` |  | reserviert | reserviert |  |
| `COST` | nicht reserviert |  |  |  |
| `COUNT` |  | reserviert | reserviert | reserviert |
| `COVAR_POP` |  | reserviert | reserviert |  |
| `COVAR_SAMP` |  | reserviert | reserviert |  |
| `CREATE` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `CROSS` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `CSV` | nicht reserviert |  |  |  |
| `CUBE` | nicht reserviert | reserviert | reserviert |  |
| `CUME_DIST` |  | reserviert | reserviert |  |
| `CURRENT` | nicht reserviert | reserviert | reserviert | reserviert |
| `CURRENT_CATALOG` | reserviert | reserviert | reserviert |  |
| `CURRENT_DATE` | reserviert | reserviert | reserviert | reserviert |
| `CURRENT_DEFAULT_TRANS-FORM_GROUP` |  | reserviert | reserviert |  |
| `CURRENT_PATH` |  | reserviert | reserviert |  |
| `CURRENT_ROLE` | reserviert | reserviert | reserviert |  |
| `CURRENT_SCHEMA` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert |  |
| `CURRENT_TIME` | reserviert | reserviert | reserviert | reserviert |
| `CURRENT_TIMESTAMP` | reserviert | reserviert | reserviert | reserviert |
| `CURRENT_TRANSFORM_GROUP_FOR_TYPE` |  | reserviert | reserviert |  |
| `CURRENT_USER` | reserviert | reserviert | reserviert | reserviert |
| `CURSOR` | nicht reserviert | reserviert | reserviert | reserviert |
| `CURSOR_NAME` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `CYCLE` | nicht reserviert | reserviert | reserviert |  |

### D

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `DATA` | nicht reserviert | nicht reserviert | nicht reserviert | nicht reserviert |
| `DATABASE` | nicht reserviert |  |  |  |
| `DATALINK` |  | reserviert | reserviert |  |
| `DATE` |  | reserviert | reserviert | reserviert |
| `DATETIME_INTERVAL_CODE` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `DATETIME_INTERVAL_PRECISION` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `DAY` | nicht reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `DB` |  | nicht reserviert | nicht reserviert |  |
| `DEALLOCATE` | nicht reserviert | reserviert | reserviert | reserviert |
| `DEC` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `DECFLOAT` |  | reserviert | reserviert |  |
| `DECIMAL` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `DECLARE` | nicht reserviert | reserviert | reserviert | reserviert |
| `DEFAULT` | reserviert | reserviert | reserviert | reserviert |
| `DEFAULTS` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `DEFERRABLE` | reserviert | nicht reserviert | nicht reserviert | reserviert |
| `DEFERRED` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `DEFINE` |  | reserviert | reserviert |  |
| `DEFINED` |  | nicht reserviert | nicht reserviert |  |
| `DEFINER` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `DEGREE` |  | nicht reserviert | nicht reserviert |  |
| `DELETE` | nicht reserviert | reserviert | reserviert | reserviert |
| `DELIMITER` | nicht reserviert |  |  |  |
| `DELIMITERS` | nicht reserviert |  |  |  |
| `DENSE_RANK` |  | reserviert | reserviert |  |
| `DEPENDS` | nicht reserviert |  |  |  |
| `DEREF` |  | reserviert | reserviert |  |
| `DERIVED` |  | nicht reserviert | nicht reserviert |  |
| `DESC` | reserviert | nicht reserviert | nicht reserviert | reserviert |
| `DESCRIBE` |  | reserviert | reserviert | reserviert |
| `DESCRIPTOR` |  | nicht reserviert | nicht reserviert | reserviert |
| `DETACH` | nicht reserviert |  |  |  |
| `DETERMINISTIC` |  | reserviert | reserviert |  |
| `DIAGNOSTICS` |  | nicht reserviert | nicht reserviert | reserviert |
| `DICTIONARY` | nicht reserviert |  |  |  |
| `DISABLE` | nicht reserviert |  |  |  |
| `DISCARD` | nicht reserviert |  |  |  |
| `DISCONNECT` |  | reserviert | reserviert | reserviert |
| `DISPATCH` |  | nicht reserviert | nicht reserviert |  |
| `DISTINCT` | reserviert | reserviert | reserviert | reserviert |
| `DLNEWCOPY` |  | reserviert | reserviert |  |
| `DLPREVIOUSCOPY` |  | reserviert | reserviert |  |
| `DLURLCOMPLETE` |  | reserviert | reserviert |  |
| `DLURLCOMPLETEONLY` |  | reserviert | reserviert |  |
| `DLURLCOMPLETEWRITE` |  | reserviert | reserviert |  |
| `DLURLPATH` |  | reserviert | reserviert |  |
| `DLURLPATHONLY` |  | reserviert | reserviert |  |
| `DLURLPATHWRITE` |  | reserviert | reserviert |  |
| `DLURLSCHEME` |  | reserviert | reserviert |  |
| `DLURLSERVER` |  | reserviert | reserviert |  |
| `DLVALUE` |  | reserviert | reserviert |  |
| `DO` | reserviert |  |  |  |
| `DOCUMENT` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `DOMAIN` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `DOUBLE` | nicht reserviert | reserviert | reserviert | reserviert |
| `DROP` | nicht reserviert | reserviert | reserviert | reserviert |
| `DYNAMIC` |  | reserviert | reserviert |  |
| `DYNAMIC_FUNCTION` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `DYNAMIC_FUNCTION_CODE` |  | nicht reserviert | nicht reserviert |  |

### E

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `EACH` | nicht reserviert | reserviert | reserviert |  |
| `ELEMENT` |  | reserviert | reserviert |  |
| `ELSE` | reserviert | reserviert | reserviert | reserviert |
| `EMPTY` | nicht reserviert | reserviert | reserviert |  |
| `ENABLE` | nicht reserviert |  |  |  |
| `ENCODING` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `ENCRYPTED` | nicht reserviert |  |  |  |
| `END-EXEC` |  | reserviert | reserviert | reserviert |
| `END_FRAME` |  | reserviert | reserviert |  |
| `END_PARTITION` |  | reserviert | reserviert |  |
| `ENFORCED` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `ENUM` | nicht reserviert |  |  |  |
| `EQUALS` |  | reserviert | reserviert |  |
| `ERROR` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `ESCAPE` | nicht reserviert | reserviert | reserviert | reserviert |
| `EVENT` | nicht reserviert |  |  |  |
| `EVERY` |  | reserviert | reserviert |  |
| `EXCEPT` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `EXCEPTION` |  |  |  | reserviert |
| `EXCLUDE` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `EXCLUDING` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `EXCLUSIVE` | nicht reserviert |  |  |  |
| `EXEC` |  | reserviert | reserviert | reserviert |
| `EXECUTE` | nicht reserviert | reserviert | reserviert | reserviert |
| `EXISTS` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `EXP` |  | reserviert | reserviert |  |
| `EXPLAIN` | nicht reserviert |  |  |  |
| `EXPRESSION` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `EXTENSION` | nicht reserviert |  |  |  |
| `EXTERNAL` | nicht reserviert | reserviert | reserviert | reserviert |
| `EXTRACT` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |

### F

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `FALSE` | reserviert | reserviert | reserviert | reserviert |
| `FAMILY` | nicht reserviert |  |  |  |
| `FETCH` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `FILE` |  | nicht reserviert | nicht reserviert |  |
| `FILTER` | nicht reserviert; erfordert `AS` | reserviert | reserviert |  |
| `FINAL` |  | nicht reserviert | nicht reserviert |  |
| `FINALIZE` | nicht reserviert |  |  |  |
| `FINISH` |  | nicht reserviert | nicht reserviert |  |
| `FIRST` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `FIRST_VALUE` |  | reserviert | reserviert |  |
| `FLOAT` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `FLOOR` |  | reserviert | reserviert |  |
| `FOLLOWING` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `FOR` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `FORCE` | nicht reserviert |  |  |  |
| `FOREIGN` | reserviert | reserviert | reserviert | reserviert |
| `FORMAT` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `FORTRAN` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `FORWARD` | nicht reserviert |  |  |  |
| `FOUND` |  | nicht reserviert | nicht reserviert | reserviert |
| `FRAME_ROW` |  | reserviert | reserviert |  |
| `FREE` |  | reserviert | reserviert |  |
| `FREEZE` | reserviert; kann Funktion oder Typ sein |  |  |  |
| `FROM` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `FS` |  | nicht reserviert | nicht reserviert |  |
| `FULFILL` |  | nicht reserviert | nicht reserviert |  |
| `FULL` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `FUNCTION` | nicht reserviert | reserviert | reserviert |  |
| `FUNCTIONS` | nicht reserviert |  |  |  |
| `FUSION` |  | reserviert | reserviert |  |

### G

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `G` |  | nicht reserviert | nicht reserviert |  |
| `GENERAL` |  | nicht reserviert | nicht reserviert |  |
| `GENERATED` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `GET` |  | reserviert | reserviert | reserviert |
| `GLOBAL` | nicht reserviert | reserviert | reserviert | reserviert |
| `GO` |  | nicht reserviert | nicht reserviert | reserviert |
| `GOTO` |  | nicht reserviert | nicht reserviert | reserviert |
| `GRANT` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `GRANTED` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `GREATEST` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert |  |  |
| `GROUP` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `GROUPS` | nicht reserviert | reserviert | reserviert |  |

### H

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `HANDLER` | nicht reserviert |  |  |  |
| `HAVING` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `HEADER` | nicht reserviert |  |  |  |
| `HEX` |  | nicht reserviert | nicht reserviert |  |
| `HIERARCHY` |  | nicht reserviert | nicht reserviert |  |
| `HOLD` | nicht reserviert | reserviert | reserviert |  |
| `HOUR` | nicht reserviert; erfordert `AS` | reserviert | reserviert | reserviert |

### I

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `ID` |  | nicht reserviert | nicht reserviert |  |
| `IDENTITY` | nicht reserviert | reserviert | reserviert | reserviert |
| `IF` | nicht reserviert |  |  |  |
| `IGNORE` |  | nicht reserviert | nicht reserviert |  |
| `ILIKE` | reserviert; kann Funktion oder Typ sein |  |  |  |
| `IMMEDIATE` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `IMMEDIATELY` |  | nicht reserviert | nicht reserviert |  |
| `IMMUTABLE` | nicht reserviert |  |  |  |
| `IMPLEMENTATION` |  | nicht reserviert | nicht reserviert |  |
| `IMPLICIT` | nicht reserviert |  |  |  |
| `IMPORT` | nicht reserviert | reserviert | reserviert |  |
| `IN` | reserviert | reserviert | reserviert | reserviert |
| `INCLUDE` | nicht reserviert |  |  |  |
| `INCLUDING` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `INCREMENT` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `INDENT` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `INDEX` | nicht reserviert |  |  |  |
| `INDEXES` | nicht reserviert |  |  |  |
| `INDICATOR` |  | reserviert | reserviert | reserviert |
| `INHERIT` | nicht reserviert |  |  |  |
| `INHERITS` | nicht reserviert |  |  |  |
| `INITIAL` |  | reserviert | reserviert |  |
| `INITIALLY` | reserviert | nicht reserviert | nicht reserviert | reserviert |
| `INLINE` | nicht reserviert |  |  |  |
| `INNER` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `INOUT` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `INPUT` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `INSERT` | nicht reserviert | reserviert | reserviert | reserviert |
| `INSTANCE` |  | nicht reserviert | nicht reserviert |  |
| `INSTANTIABLE` |  | nicht reserviert | nicht reserviert |  |
| `INSTEAD` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `INT` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `INTEGER` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `INTEGRITY` |  | nicht reserviert | nicht reserviert |  |
| `INTERSECT` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `INTERSECTION` |  | reserviert | reserviert |  |
| `INTERVAL` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `INTO` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `INVOKER` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `IS` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `ISNULL` | reserviert; kann Funktion oder Typ sein; erfordert `AS` |  |  |  |
| `ISOLATION` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |

### J

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `JOIN` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `JSON` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert |  |  |
| `JSON_ARRAY` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `JSON_ARRAYAGG` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `JSON_EXISTS` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `JSON_OBJECT` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `JSON_QUERY` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `JSON_SCALAR` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert |  |  |
| `JSON_SERIALIZE` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert |  |  |
| `JSON_TABLE` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `JSON_TABLE_PRIMITIVE` |  | reserviert | reserviert |  |
| `JSON_VALUE` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |

### K

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `K` |  | nicht reserviert | nicht reserviert |  |
| `KEEP` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `KEY` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `KEYS` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `KEY_MEMBER` |  | nicht reserviert | nicht reserviert |  |
| `KEY_TYPE` |  | nicht reserviert | nicht reserviert |  |

### L

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `LABEL` | nicht reserviert |  |  |  |
| `LAG` |  | reserviert | reserviert |  |
| `LANGUAGE` | nicht reserviert | reserviert | reserviert | reserviert |
| `LARGE` | nicht reserviert | reserviert | reserviert |  |
| `LAST` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `LAST_VALUE` |  | reserviert | reserviert |  |
| `LATERAL` | reserviert | reserviert | reserviert |  |
| `LEAD` |  | reserviert | reserviert |  |
| `LEADING` | reserviert | reserviert | reserviert | reserviert |
| `LEAKPROOF` | nicht reserviert |  |  |  |
| `LEAST` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert |  |  |
| `LEFT` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `LENGTH` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `LEVEL` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `LIBRARY` |  | nicht reserviert | nicht reserviert |  |
| `LIKE` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `LIKE_REGEX` |  | reserviert | reserviert |  |
| `LINK` |  | nicht reserviert | nicht reserviert |  |
| `LISTAGG` |  | reserviert | reserviert |  |
| `LISTEN` | nicht reserviert |  |  |  |
| `LN` |  | reserviert | reserviert |  |
| `LOAD` | nicht reserviert |  |  |  |
| `LOCAL` | nicht reserviert | reserviert | reserviert | reserviert |
| `LOCALTIME` | reserviert | reserviert | reserviert |  |
| `LOCALTIMESTAMP` | reserviert | reserviert | reserviert |  |
| `LOCATION` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `LOCATOR` |  | nicht reserviert | nicht reserviert |  |
| `LOCK` | nicht reserviert |  |  |  |
| `LOCKED` | nicht reserviert |  |  |  |
| `LOG` |  | reserviert | reserviert |  |
| `LOG10` |  | reserviert | reserviert |  |
| `LOGGED` | nicht reserviert |  |  |  |
| `LOWER` |  | reserviert | reserviert | reserviert |
| `LPAD` |  | reserviert |  |  |
| `LTRIM` |  | reserviert |  |  |

### M

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `M` |  | nicht reserviert | nicht reserviert |  |
| `MAP` |  | nicht reserviert | nicht reserviert |  |
| `MAPPING` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `MATCH` | nicht reserviert | reserviert | reserviert | reserviert |
| `MATCHED` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `MATCHES` |  | reserviert | reserviert |  |
| `MATCH_NUMBER` |  | reserviert | reserviert |  |
| `MATCH_RECOGNIZE` |  | reserviert | reserviert |  |
| `MATERIALIZED` | nicht reserviert |  |  |  |
| `MAX` |  | reserviert | reserviert | reserviert |
| `MAXVALUE` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `MEASURES` |  | nicht reserviert | nicht reserviert |  |
| `MEMBER` |  | reserviert | reserviert |  |
| `MERGE` | nicht reserviert | reserviert | reserviert |  |
| `MERGE_ACTION` | nicht reserviert; kann nicht Funktion oder Typ sein |  |  |  |
| `MESSAGE_LENGTH` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `MESSAGE_OCTET_LENGTH` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `MESSAGE_TEXT` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `METHOD` | nicht reserviert | reserviert | reserviert |  |
| `MIN` |  | reserviert | reserviert | reserviert |
| `MINVALUE` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `MOD` |  | reserviert | reserviert |  |
| `MODE` | nicht reserviert |  |  |  |
| `MODIFIES` |  | reserviert | reserviert |  |
| `MODULE` |  | reserviert | reserviert | reserviert |
| `MONTH` | nicht reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `MORE` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `MOVE` | nicht reserviert |  |  |  |
| `MULTISET` |  | reserviert | reserviert |  |
| `MUMPS` |  | nicht reserviert | nicht reserviert | nicht reserviert |

### N

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `NAME` | nicht reserviert | nicht reserviert | nicht reserviert | nicht reserviert |
| `NAMES` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `NAMESPACE` |  | nicht reserviert | nicht reserviert |  |
| `NATIONAL` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `NATURAL` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `NCHAR` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `NCLOB` |  | reserviert | reserviert |  |
| `NESTED` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `NESTING` |  | nicht reserviert | nicht reserviert |  |
| `NEW` | nicht reserviert | reserviert | reserviert |  |
| `NEXT` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `NFC` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `NFD` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `NFKC` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `NFKD` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `NIL` |  | nicht reserviert | nicht reserviert |  |
| `NO` | nicht reserviert | reserviert | reserviert | reserviert |
| `NONE` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `NORMALIZE` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `NORMALIZED` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `NOT` | reserviert | reserviert | reserviert | reserviert |
| `NOTHING` | nicht reserviert |  |  |  |
| `NOTNULL` | reserviert; kann Funktion oder Typ sein; erfordert `AS` |  |  |  |
| `NOWAIT` | nicht reserviert |  |  |  |
| `NTH_VALUE` |  | reserviert | reserviert |  |
| `NTILE` |  | reserviert | reserviert |  |
| `NULL` | reserviert | reserviert | reserviert | reserviert |
| `NULLABLE` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `NULLIF` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `NULLS` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `NULL_ORDERING` |  | nicht reserviert | nicht reserviert |  |
| `NUMBER` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `NUMERIC` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |

### O

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `OBJECT` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `OBJECTS` | nicht reserviert |  |  |  |
| `OCCURRENCE` |  | nicht reserviert | nicht reserviert |  |
| `OCCURRENCES_REGEX` |  | reserviert | reserviert |  |
| `OCTETS` |  | nicht reserviert | nicht reserviert |  |
| `OCTET_LENGTH` |  | reserviert | reserviert | reserviert |
| `OF` | nicht reserviert | reserviert | reserviert | reserviert |
| `OFF` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `OFFSET` | reserviert; erfordert `AS` | reserviert | reserviert |  |
| `OIDS` | nicht reserviert |  |  |  |
| `OLD` | nicht reserviert | reserviert | reserviert |  |
| `OMIT` | nicht reserviert | reserviert | reserviert |  |
| `ON` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `ONE` |  | reserviert | reserviert |  |
| `ONLY` | reserviert | reserviert | reserviert | reserviert |
| `OPEN` |  | reserviert | reserviert | reserviert |
| `OPERATOR` | nicht reserviert |  |  |  |
| `OPTION` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `OPTIONS` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `OR` | reserviert | reserviert | reserviert | reserviert |
| `ORDER` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `ORDERING` |  | nicht reserviert | nicht reserviert |  |
| `OTHERS` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `OUT` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `OUTER` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `OUTPUT` |  | nicht reserviert | nicht reserviert | reserviert |
| `OVER` | nicht reserviert; erfordert `AS` | reserviert | reserviert |  |
| `OVERFLOW` |  | nicht reserviert | nicht reserviert |  |
| `OVERLAPS` | reserviert; kann Funktion oder Typ sein; erfordert `AS` | reserviert | reserviert | reserviert |
| `OVERLAY` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `OVERRIDING` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `OWNED` | nicht reserviert |  |  |  |
| `OWNER` | nicht reserviert |  |  |  |

### P

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `P` |  | nicht reserviert | nicht reserviert |  |
| `PAD` |  | nicht reserviert | nicht reserviert | reserviert |
| `PARALLEL` | nicht reserviert |  |  |  |
| `PARAMETER` | nicht reserviert | reserviert | reserviert |  |
| `PARAMETER_MODE` |  | nicht reserviert | nicht reserviert |  |
| `PARAMETER_NAME` |  | nicht reserviert | nicht reserviert |  |
| `PARAMETER_ORDINAL_POSITION` |  | nicht reserviert | nicht reserviert |  |
| `PARAMETER_SPECIFIC_CAT-ALOG` |  | nicht reserviert | nicht reserviert |  |
| `PARAMETER_SPECIFIC_NAME` |  | nicht reserviert | nicht reserviert |  |
| `PARAMETER_SPECIFIC_SCHEMA` |  | nicht reserviert | nicht reserviert |  |
| `PARSER` | nicht reserviert |  |  |  |
| `PARTIAL` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `PARTITION` | nicht reserviert | reserviert | reserviert |  |
| `PASCAL` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `PASS` |  | nicht reserviert | nicht reserviert |  |
| `PASSING` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `PASSTHROUGH` |  | nicht reserviert | nicht reserviert |  |
| `PASSWORD` | nicht reserviert |  |  |  |
| `PAST` |  | nicht reserviert | nicht reserviert |  |
| `PATH` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `PATTERN` |  | reserviert | reserviert |  |
| `PERCENT` |  | reserviert | reserviert |  |
| `PERCENTILE_CONT` |  | reserviert | reserviert |  |
| `PERCENTILE_DISC` |  | reserviert | reserviert |  |
| `PERCENT_RANK` |  | reserviert | reserviert |  |
| `PERIOD` | nicht reserviert | reserviert | reserviert |  |
| `PERMISSION` |  | nicht reserviert | nicht reserviert |  |
| `PERMUTE` |  | nicht reserviert | nicht reserviert |  |
| `PIPE` |  | nicht reserviert | nicht reserviert |  |
| `PLACING` | reserviert | nicht reserviert | nicht reserviert |  |
| `PLAN` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `PLANS` | nicht reserviert |  |  |  |
| `PLI` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `POLICY` | nicht reserviert |  |  |  |
| `PORTION` |  | reserviert | reserviert |  |
| `POSITION` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `POSITION_REGEX` |  | reserviert | reserviert |  |
| `POWER` |  | reserviert | reserviert |  |
| `PRECEDES` |  | reserviert | reserviert |  |
| `PRECEDING` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `PRECISION` | nicht reserviert; kann nicht Funktion oder Typ sein; erfordert `AS` | reserviert | reserviert | reserviert |
| `PREPARE` | nicht reserviert | reserviert | reserviert | reserviert |
| `PREPARED` | nicht reserviert |  |  |  |
| `PRESERVE` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `PREV` |  | nicht reserviert | nicht reserviert |  |
| `PRIMARY` | reserviert | reserviert | reserviert | reserviert |
| `PRIOR` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `PRIVATE` |  | nicht reserviert | nicht reserviert |  |
| `PRIVILEGES` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `PROCEDURAL` | nicht reserviert |  |  |  |
| `PROCEDURE` | nicht reserviert | reserviert | reserviert | reserviert |
| `PROCEDURES` | nicht reserviert |  |  |  |
| `PROGRAM` | nicht reserviert |  |  |  |
| `PRUNE` |  | nicht reserviert | nicht reserviert |  |
| `PTF` |  | reserviert | reserviert |  |
| `PUBLIC` |  | nicht reserviert | nicht reserviert | reserviert |
| `PUBLICATION` | nicht reserviert |  |  |  |

### Q

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `QUOTE` | nicht reserviert |  |  |  |

### R

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `RANGE` | nicht reserviert | reserviert | reserviert |  |
| `RANK` |  | reserviert | reserviert |  |
| `READ` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `READS` |  | reserviert | reserviert |  |
| `REAL` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `REASSIGN` | nicht reserviert |  |  |  |
| `RECOVERY` |  | nicht reserviert | nicht reserviert |  |
| `RECURSIVE` | nicht reserviert | reserviert | reserviert |  |
| `REF` | nicht reserviert | reserviert | reserviert |  |
| `REFERENCES` | reserviert | reserviert | reserviert | reserviert |
| `REFERENCING` | nicht reserviert | reserviert | reserviert |  |
| `REFRESH` | nicht reserviert |  |  |  |
| `REGR_AVGX` |  | reserviert | reserviert |  |
| `REGR_AVGY` |  | reserviert | reserviert |  |
| `REGR_COUNT` |  | reserviert | reserviert |  |
| `REGR_INTERCEPT` |  | reserviert | reserviert |  |
| `REGR_R2` |  | reserviert | reserviert |  |
| `REGR_SLOPE` |  | reserviert | reserviert |  |
| `REGR_SXX` |  | reserviert | reserviert |  |
| `REGR_SXY` |  | reserviert | reserviert |  |
| `REGR_SYY` |  | reserviert | reserviert |  |
| `REINDEX` | nicht reserviert |  |  |  |
| `RELATIVE` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `RELEASE` | nicht reserviert | reserviert | reserviert |  |
| `RENAME` | nicht reserviert |  |  |  |
| `REPEATABLE` | nicht reserviert | nicht reserviert | nicht reserviert | nicht reserviert |
| `REPLACE` | nicht reserviert |  |  |  |
| `REPLICA` | nicht reserviert |  |  |  |
| `REQUIRING` |  | nicht reserviert | nicht reserviert |  |
| `RESET` | nicht reserviert |  |  |  |
| `RESPECT` |  | nicht reserviert | nicht reserviert |  |
| `RESTART` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `RESTORE` |  | nicht reserviert | nicht reserviert |  |
| `RESTRICT` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `RESULT` |  | reserviert | reserviert |  |
| `RETURN` | nicht reserviert | reserviert | reserviert |  |
| `RETURNED_CARDINALITY` |  | nicht reserviert | nicht reserviert |  |
| `RETURNED_LENGTH` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `RETURNED_SQLSTATE` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `RETURNING` | reserviert; erfordert `AS` | nicht reserviert | nicht reserviert |  |
| `RETURNS` | nicht reserviert | reserviert | reserviert |  |
| `REVOKE` | nicht reserviert | reserviert | reserviert | reserviert |
| `RIGHT` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `ROLE` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `ROLLBACK` | nicht reserviert | reserviert | reserviert | reserviert |
| `ROLLUP` | nicht reserviert | reserviert | reserviert |  |
| `ROUTINE` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `ROUTINES` | nicht reserviert |  |  |  |
| `ROUTINE_CATALOG` |  | nicht reserviert | nicht reserviert |  |
| `ROUTINE_NAME` |  | nicht reserviert | nicht reserviert |  |
| `ROUTINE_SCHEMA` |  | nicht reserviert | nicht reserviert |  |
| `ROW` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `ROWS` | nicht reserviert | reserviert | reserviert | reserviert |
| `ROW_COUNT` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `ROW_NUMBER` |  | reserviert | reserviert |  |
| `RPAD` |  | reserviert |  |  |
| `RTRIM` |  | reserviert |  |  |
| `RULE` | nicht reserviert |  |  |  |
| `RUNNING` |  | reserviert | reserviert |  |

### S

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `SAVEPOINT` | nicht reserviert | reserviert | reserviert |  |
| `SCALAR` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `SCALE` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `SCHEMA` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `SCHEMAS` | nicht reserviert |  |  |  |
| `SCHEMA_NAME` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `SCOPE` |  | reserviert | reserviert |  |
| `SCOPE_CATALOG` |  | nicht reserviert | nicht reserviert |  |
| `SCOPE_NAME` |  | nicht reserviert | nicht reserviert |  |
| `SCOPE_SCHEMA` |  | nicht reserviert | nicht reserviert |  |
| `SCROLL` | nicht reserviert | reserviert | reserviert | reserviert |
| `SEARCH` | nicht reserviert | reserviert | reserviert |  |
| `SECOND` | nicht reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `SECTION` |  | nicht reserviert | nicht reserviert | reserviert |
| `SECURITY` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `SELECT` | reserviert | reserviert | reserviert | reserviert |
| `SELECTIVE` |  | nicht reserviert | nicht reserviert |  |
| `SELF` |  | nicht reserviert | nicht reserviert |  |
| `SEMANTICS` |  | nicht reserviert | nicht reserviert |  |
| `SENSITIVE` |  | reserviert | reserviert |  |
| `SEQUENCE` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `SEQUENCES` | nicht reserviert |  |  |  |
| `SERIALIZABLE` | nicht reserviert | nicht reserviert | nicht reserviert | nicht reserviert |
| `SERVER` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `SERVER_NAME` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `SESSION` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `SESSION_USER` | reserviert | reserviert | reserviert | reserviert |
| `SET` | nicht reserviert | reserviert | reserviert | reserviert |
| `SETOF` | nicht reserviert; kann nicht Funktion oder Typ sein |  |  |  |
| `SETS` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `SHARE` | nicht reserviert |  |  |  |
| `SHOW` | nicht reserviert | reserviert | reserviert |  |
| `SIMILAR` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert |  |
| `SIMPLE` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `SIN` |  | reserviert | reserviert |  |
| `SINH` |  | reserviert | reserviert |  |
| `SIZE` |  | nicht reserviert | nicht reserviert | reserviert |
| `SKIP` | nicht reserviert | reserviert | reserviert |  |
| `SMALLINT` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `SNAPSHOT` | nicht reserviert |  |  |  |
| `SOME` | reserviert | reserviert | reserviert | reserviert |
| `SORT_DIRECTION` |  | nicht reserviert | nicht reserviert |  |
| `SOURCE` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `SPACE` |  | nicht reserviert | nicht reserviert | reserviert |
| `SPECIFIC` |  | reserviert | reserviert |  |
| `SPECIFICTYPE` |  | reserviert | reserviert |  |
| `SPECIFIC_NAME` |  | nicht reserviert | nicht reserviert |  |
| `SQL` | nicht reserviert | reserviert | reserviert | reserviert |
| `SQLCODE` |  |  |  | reserviert |
| `SQLERROR` |  |  |  | reserviert |
| `SQLEXCEPTION` |  | reserviert | reserviert |  |
| `SQLWARNING` |  | reserviert | reserviert |  |
| `SQRT` |  | reserviert | reserviert |  |
| `STABLE` | nicht reserviert |  |  |  |
| `STANDALONE` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `START` | nicht reserviert | reserviert | reserviert |  |
| `STATE` |  | nicht reserviert | nicht reserviert |  |
| `STATEMENT` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `STATIC` |  | reserviert | reserviert |  |
| `STATISTICS` | nicht reserviert |  |  |  |
| `STDDEV_POP` |  | reserviert | reserviert |  |
| `STDDEV_SAMP` |  | reserviert | reserviert |  |
| `STDIN` | nicht reserviert |  |  |  |
| `STDOUT` | nicht reserviert |  |  |  |
| `STORAGE` | nicht reserviert |  |  |  |
| `STORED` | nicht reserviert |  |  |  |
| `STRICT` | nicht reserviert |  |  |  |
| `STRING` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `STRIP` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `STRUCTURE` |  | nicht reserviert | nicht reserviert |  |
| `STYLE` |  | nicht reserviert | nicht reserviert |  |
| `SUBCLASS_ORIGIN` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `SUBMULTISET` |  | reserviert | reserviert |  |
| `SUBSCRIPTION` | nicht reserviert |  |  |  |
| `SUBSET` |  | reserviert | reserviert |  |
| `SUBSTRING` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `SUBSTRING_REGEX` |  | reserviert | reserviert |  |
| `SUCCEEDS` |  | reserviert | reserviert |  |
| `SUM` |  | reserviert | reserviert | reserviert |
| `SUPPORT` | nicht reserviert |  |  |  |
| `SYMMETRIC` | reserviert | reserviert | reserviert |  |
| `SYSID` | nicht reserviert |  |  |  |
| `SYSTEM` | nicht reserviert | reserviert | reserviert |  |
| `SYSTEM_TIME` |  | reserviert | reserviert |  |
| `SYSTEM_USER` | reserviert | reserviert | reserviert | reserviert |

### T

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `T` |  | nicht reserviert | nicht reserviert |  |
| `TABLE` | reserviert | reserviert | reserviert | reserviert |
| `TABLES` | nicht reserviert |  |  |  |
| `TABLESAMPLE` | reserviert; kann Funktion oder Typ sein | reserviert | reserviert |  |
| `TABLE_NAME` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `TAN` |  | reserviert | reserviert |  |
| `TANH` |  | reserviert | reserviert |  |
| `TARGET` | nicht reserviert |  |  |  |
| `TEMP` | nicht reserviert |  |  |  |
| `TEMPLATE` | nicht reserviert |  |  |  |
| `TEMPORARY` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `TEXT` | nicht reserviert |  |  |  |
| `THEN` | reserviert | reserviert | reserviert | reserviert |
| `THROUGH` |  | nicht reserviert | nicht reserviert |  |
| `TIES` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `TIME` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `TIMESTAMP` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `TIMEZONE_HOUR` |  | reserviert | reserviert | reserviert |
| `TIMEZONE_MINUTE` |  | reserviert | reserviert | reserviert |
| `TO` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `TOKEN` |  | nicht reserviert | nicht reserviert |  |
| `TOP_LEVEL_COUNT` |  | nicht reserviert | nicht reserviert |  |
| `TRAILING` | reserviert | reserviert | reserviert | reserviert |
| `TRANSACTION` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `TRANSACTIONS_COMMITTED` |  | nicht reserviert | nicht reserviert |  |
| `TRANSACTIONS_ROLLED_BACK` |  | nicht reserviert | nicht reserviert |  |
| `TRANSACTION_ACTIVE` |  | nicht reserviert | nicht reserviert |  |
| `TRANSFORM` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `TRANSFORMS` |  | nicht reserviert | nicht reserviert |  |
| `TRANSLATE` |  | reserviert | reserviert | reserviert |
| `TRANSLATE_REGEX` |  | reserviert | reserviert |  |
| `TRANSLATION` |  | reserviert | reserviert | reserviert |
| `TREAT` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `TRIGGER` | nicht reserviert | reserviert | reserviert |  |
| `TRIGGER_CATALOG` |  | nicht reserviert | nicht reserviert |  |
| `TRIGGER_NAME` |  | nicht reserviert | nicht reserviert |  |
| `TRIGGER_SCHEMA` |  | nicht reserviert | nicht reserviert |  |
| `TRIM_ARRAY` |  | reserviert | reserviert |  |
| `TRUE` | reserviert | reserviert | reserviert | reserviert |
| `TRUNCATE` | nicht reserviert | reserviert | reserviert |  |
| `TRUSTED` | nicht reserviert |  |  |  |
| `TYPE` | nicht reserviert | nicht reserviert | nicht reserviert | nicht reserviert |
| `TYPES` | nicht reserviert |  |  |  |

### U

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `UESCAPE` | nicht reserviert | reserviert | reserviert |  |
| `UNBOUNDED` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `UNCOMMITTED` | nicht reserviert | nicht reserviert | nicht reserviert | nicht reserviert |
| `UNCONDITIONAL` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `UNDER` |  | nicht reserviert | nicht reserviert |  |
| `UNENCRYPTED` | nicht reserviert |  |  |  |
| `UNION` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `UNIQUE` | reserviert | reserviert | reserviert | reserviert |
| `UNKNOWN` | nicht reserviert | reserviert | reserviert | reserviert |
| `UNLINK` |  | nicht reserviert | nicht reserviert |  |
| `UNLISTEN` | nicht reserviert |  |  |  |
| `UNLOGGED` | nicht reserviert |  |  |  |
| `UNMATCHED` |  | nicht reserviert | nicht reserviert |  |
| `UNNAMED` |  | nicht reserviert | nicht reserviert | nicht reserviert |
| `UNNEST` |  | reserviert | reserviert |  |
| `UNTIL` | nicht reserviert |  |  |  |
| `UNTYPED` |  | nicht reserviert | nicht reserviert |  |
| `UPDATE` | nicht reserviert | reserviert | reserviert | reserviert |
| `UPPER` |  | reserviert | reserviert | reserviert |
| `URI` |  | nicht reserviert | nicht reserviert |  |
| `USAGE` |  | nicht reserviert | nicht reserviert | reserviert |
| `USER` | reserviert | reserviert | reserviert | reserviert |
| `USER_DEFINED_TYPE_CATA-LOG` |  | nicht reserviert | nicht reserviert |  |
| `USER_DEFINED_TYPE_CODE` |  | nicht reserviert | nicht reserviert |  |
| `USER_DEFINED_TYPE_NAME` |  | nicht reserviert | nicht reserviert |  |
| `USER_DEFINED_TYPE_SCHEMA` |  | nicht reserviert | nicht reserviert |  |
| `USING` | reserviert | reserviert | reserviert | reserviert |
| `UTF16` |  | nicht reserviert | nicht reserviert |  |
| `UTF32` |  | nicht reserviert | nicht reserviert |  |
| `UTF8` |  | nicht reserviert | nicht reserviert |  |

### V

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `VACUUM` | nicht reserviert |  |  |  |
| `VALID` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `VALIDATOR` | nicht reserviert |  |  |  |
| `VALUE` | nicht reserviert | reserviert | reserviert | reserviert |
| `VALUES` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `VALUE_OF` |  | reserviert | reserviert |  |
| `VARBINARY` |  | reserviert | reserviert |  |
| `VARCHAR` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert | reserviert |
| `VARIADIC` | reserviert |  |  |  |
| `VARYING` | nicht reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `VAR_POP` |  | reserviert | reserviert |  |
| `VAR_SAMP` |  | reserviert | reserviert |  |
| `VERBOSE` | reserviert; kann Funktion oder Typ sein |  |  |  |
| `VERSION` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `VERSIONING` |  | reserviert | reserviert |  |
| `VIEW` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `VIEWS` | nicht reserviert |  |  |  |
| `VIRTUAL` | nicht reserviert |  |  |  |
| `VOLATILE` | nicht reserviert |  |  |  |

### W

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `WHEN` | reserviert | reserviert | reserviert | reserviert |
| `WHENEVER` |  | reserviert | reserviert | reserviert |
| `WHERE` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `WHITESPACE` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `WIDTH_BUCKET` |  | reserviert | reserviert |  |
| `WINDOW` | reserviert; erfordert `AS` | reserviert | reserviert |  |
| `WITH` | reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `WITHIN` | nicht reserviert; erfordert `AS` | reserviert | reserviert |  |
| `WITHOUT` | nicht reserviert; erfordert `AS` | reserviert | reserviert |  |
| `WORK` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
| `WRAPPER` | nicht reserviert | nicht reserviert | nicht reserviert |  |
| `WRITE` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |

### X

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `XML` | nicht reserviert | reserviert | reserviert |  |
| `XMLATTRIBUTES` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `XMLBINARY` |  | reserviert | reserviert |  |
| `XMLCAST` |  | reserviert | reserviert |  |
| `XMLCOMMENT` |  | reserviert | reserviert |  |
| `XMLCONCAT` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `XMLDECLARATION` |  | nicht reserviert | nicht reserviert |  |
| `XMLDOCUMENT` |  | reserviert | reserviert |  |
| `XMLELEMENT` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `XMLEXISTS` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `XMLFOREST` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `XMLITERATE` |  | reserviert | reserviert |  |
| `XMLNAMESPACES` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `XMLPARSE` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `XMLPI` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `XMLQUERY` |  | reserviert | reserviert |  |
| `XMLROOT` | nicht reserviert; kann nicht Funktion oder Typ sein |  |  |  |
| `XMLSCHEMA` |  | nicht reserviert | nicht reserviert |  |
| `XMLSERIALIZE` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `XMLTABLE` | nicht reserviert; kann nicht Funktion oder Typ sein | reserviert | reserviert |  |
| `XMLTEXT` |  | reserviert | reserviert |  |
| `XMLVALIDATE` |  | reserviert | reserviert |  |

### Y

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `YEAR` | nicht reserviert; erfordert `AS` | reserviert | reserviert | reserviert |
| `YES` | nicht reserviert | nicht reserviert | nicht reserviert |  |

### Z

| Schlüsselwort | PostgreSQL | SQL:2023 | SQL:2016 | SQL-92 |
| --- | --- | --- | --- | --- |
| `ZONE` | nicht reserviert | nicht reserviert | nicht reserviert | reserviert |
