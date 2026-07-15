# 35. Das Information Schema

Das Information Schema besteht aus einer Gruppe von Sichten, die Informationen über die in der aktuellen Datenbank definierten Objekte enthalten. Das Information Schema ist im SQL-Standard definiert und kann daher als portabel und stabil erwartet werden, anders als die Systemkataloge, die PostgreSQL-spezifisch sind und sich an Implementierungsbelangen orientieren. Die Sichten des Information Schemas enthalten jedoch keine Informationen über PostgreSQL-spezifische Funktionen; dafür müssen Sie die Systemkataloge oder andere PostgreSQL-spezifische Sichten abfragen.

> **Hinweis:** Wenn Sie die Datenbank nach Informationen über Constraints abfragen, kann eine standardkonforme Abfrage, die eine Zeile erwartet, mehrere Zeilen zurückgeben. Der Grund ist, dass der SQL-Standard verlangt, dass Constraint-Namen innerhalb eines Schemas eindeutig sind, PostgreSQL diese Einschränkung aber nicht erzwingt. Von PostgreSQL automatisch erzeugte Constraint-Namen vermeiden Duplikate im selben Schema, Benutzer können solche doppelten Namen jedoch explizit angeben.
>
> Dieses Problem kann bei Abfragen von Information-Schema-Sichten wie `check_constraint_routine_usage`, `check_constraints`, `domain_constraints` und `referential_constraints` auftreten. Einige andere Sichten haben ähnliche Probleme, enthalten aber den Tabellennamen, um doppelte Zeilen unterscheidbar zu machen, zum Beispiel `constraint_column_usage`, `constraint_table_usage` und `table_constraints`.

## 35.1. Das Schema

Das Information Schema selbst ist ein Schema namens `information_schema`. Dieses Schema existiert automatisch in allen Datenbanken. Eigentümer dieses Schemas ist der anfängliche Datenbankbenutzer im Cluster, und dieser Benutzer hat naturgemäß alle Privilegien auf diesem Schema, einschließlich der Möglichkeit, es zu löschen; die dadurch erzielte Platzeinsparung ist allerdings winzig.

Standardmäßig befindet sich das Information Schema nicht im Schema-Suchpfad, sodass Sie auf alle Objekte darin über qualifizierte Namen zugreifen müssen. Da einige Objekte im Information Schema generische Namen haben, die auch in Benutzeranwendungen vorkommen können, sollten Sie vorsichtig sein, wenn Sie das Information Schema in den Suchpfad aufnehmen möchten.

## 35.2. Datentypen

Die Spalten der Information-Schema-Sichten verwenden spezielle Datentypen, die im Information Schema definiert sind. Sie sind als einfache Domains über gewöhnlichen eingebauten Typen definiert. Sie sollten diese Typen nicht für Arbeiten außerhalb des Information Schemas verwenden, Ihre Anwendungen müssen aber damit umgehen können, wenn sie aus dem Information Schema lesen.

Diese Typen sind:

| Typ | Beschreibung |
| --- | --- |
| `cardinal_number` | Eine nichtnegative Ganzzahl. |
| `character_data` | Eine Zeichenkette ohne bestimmte maximale Länge. |
| `sql_identifier` | Eine Zeichenkette. Dieser Typ wird für SQL-Bezeichner verwendet; der Typ `character_data` wird für andere Arten von Textdaten verwendet. |
| `time_stamp` | Eine Domain über dem Typ `timestamp with time zone`. |
| `yes_or_no` | Eine Zeichenketten-Domain, die entweder `YES` oder `NO` enthält. Sie wird im Information Schema verwendet, um boolesche Daten (wahr/falsch) darzustellen. Das Information Schema wurde erfunden, bevor der Typ `boolean` in den SQL-Standard aufgenommen wurde; diese Konvention ist daher notwendig, um das Information Schema rückwärtskompatibel zu halten. |

Jede Spalte im Information Schema hat einen dieser fünf Typen.

## 35.3. information_schema_catalog_name

`information_schema_catalog_name` ist eine Tabelle, die immer eine Zeile und eine Spalte enthält. Diese Spalte enthält den Namen der aktuellen Datenbank (in der SQL-Terminologie: des aktuellen Katalogs).

**Tabelle 35.1. Spalten von `information_schema_catalog_name`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `catalog_name` | `sql_identifier` | Name der Datenbank, die dieses Information Schema enthält |

## 35.4. administrable_role_authorizations

Die Sicht `administrable_role_authorizations` identifiziert alle Rollen, für die der aktuelle Benutzer die Admin-Option hat.

**Tabelle 35.2. Spalten von `administrable_role_authorizations`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `grantee` | `sql_identifier` | Name der Rolle, der diese Rollenmitgliedschaft gewährt wurde; dies kann der aktuelle Benutzer oder, bei verschachtelten Rollenmitgliedschaften, eine andere Rolle sein |
| `role_name` | `sql_identifier` | Name einer Rolle |
| `is_grantable` | `yes_or_no` | Immer `YES` |

## 35.5. applicable_roles

Die Sicht `applicable_roles` identifiziert alle Rollen, deren Privilegien der aktuelle Benutzer verwenden kann. Das bedeutet, dass es eine Kette von Rollengewährungen vom aktuellen Benutzer zu der betreffenden Rolle gibt. Der aktuelle Benutzer selbst ist ebenfalls eine anwendbare Rolle. Die Menge der anwendbaren Rollen wird im Allgemeinen für Berechtigungsprüfungen verwendet.

**Tabelle 35.3. Spalten von `applicable_roles`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `grantee` | `sql_identifier` | Name der Rolle, der diese Rollenmitgliedschaft gewährt wurde; dies kann der aktuelle Benutzer oder, bei verschachtelten Rollenmitgliedschaften, eine andere Rolle sein |
| `role_name` | `sql_identifier` | Name einer Rolle |
| `is_grantable` | `yes_or_no` | `YES`, wenn der Empfänger die Admin-Option für die Rolle hat, sonst `NO` |

## 35.6. attributes

Die Sicht `attributes` enthält Informationen über die Attribute zusammengesetzter Datentypen, die in der Datenbank definiert sind. Beachten Sie, dass die Sicht keine Informationen über Tabellenspalten liefert, die in PostgreSQL-Zusammenhängen manchmal ebenfalls Attribute genannt werden. Angezeigt werden nur Attribute, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer des Typs ist oder weil er ein Privileg auf dem Typ besitzt.

**Tabelle 35.4. Spalten von `attributes`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `udt_catalog` | `sql_identifier` | Name der Datenbank, die den Datentyp enthält; immer die aktuelle Datenbank |
| `udt_schema` | `sql_identifier` | Name des Schemas, das den Datentyp enthält |
| `udt_name` | `sql_identifier` | Name des Datentyps |
| `attribute_name` | `sql_identifier` | Name des Attributs |
| `ordinal_position` | `cardinal_number` | Ordnungsposition des Attributs innerhalb des Datentyps; die Zählung beginnt bei 1 |
| `attribute_default` | `character_data` | Default-Ausdruck des Attributs |
| `is_nullable` | `yes_or_no` | `YES`, wenn das Attribut möglicherweise null sein kann, `NO`, wenn bekannt ist, dass es nicht null sein kann |
| `data_type` | `character_data` | Datentyp des Attributs, wenn es ein eingebauter Typ ist; `ARRAY`, wenn es ein Array ist (siehe in diesem Fall die Sicht `element_types`); andernfalls `USER-DEFINED` (in diesem Fall wird der Typ in `attribute_udt_name` und den zugehörigen Spalten identifiziert) |
| `character_maximum_length` | `cardinal_number` | Wenn `data_type` einen Zeichenketten- oder Bitketten-Typ identifiziert, die deklarierte maximale Länge; null für alle anderen Datentypen oder wenn keine maximale Länge deklariert wurde |
| `character_octet_length` | `cardinal_number` | Wenn `data_type` einen Zeichentyp identifiziert, die maximal mögliche Länge eines Werts in Oktetten (Bytes); null für alle anderen Datentypen. Die maximale Oktettlänge hängt von der deklarierten maximalen Zeichenlänge (siehe oben) und der Serverkodierung ab |
| `character_set_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `collation_catalog` | `sql_identifier` | Name der Datenbank, die die Sortierfolge des Attributs enthält; immer die aktuelle Datenbank. Null, wenn die Standardsortierfolge gilt oder der Datentyp des Attributs nicht sortierbar ist |
| `collation_schema` | `sql_identifier` | Name des Schemas, das die Sortierfolge des Attributs enthält; null, wenn die Standardsortierfolge gilt oder der Datentyp des Attributs nicht sortierbar ist |
| `collation_name` | `sql_identifier` | Name der Sortierfolge des Attributs; null, wenn die Standardsortierfolge gilt oder der Datentyp des Attributs nicht sortierbar ist |
| `numeric_precision` | `cardinal_number` | Wenn `data_type` einen numerischen Typ identifiziert, enthält diese Spalte die deklarierte oder implizite Genauigkeit des Typs für dieses Attribut. Die Genauigkeit gibt die Anzahl signifikanter Ziffern an. Sie kann dezimal (Basis 10) oder binär (Basis 2) ausgedrückt werden, wie in der Spalte `numeric_precision_radix` angegeben. Für alle anderen Datentypen ist diese Spalte null |
| `numeric_precision_radix` | `cardinal_number` | Wenn `data_type` einen numerischen Typ identifiziert, gibt diese Spalte an, in welcher Basis die Werte in den Spalten `numeric_precision` und `numeric_scale` ausgedrückt werden. Der Wert ist entweder 2 oder 10. Für alle anderen Datentypen ist diese Spalte null |
| `numeric_scale` | `cardinal_number` | Wenn `data_type` einen exakten numerischen Typ identifiziert, enthält diese Spalte die deklarierte oder implizite Skalierung des Typs für dieses Attribut. Die Skalierung gibt die Anzahl signifikanter Ziffern rechts vom Dezimalpunkt an. Sie kann dezimal (Basis 10) oder binär (Basis 2) ausgedrückt werden, wie in der Spalte `numeric_precision_radix` angegeben. Für alle anderen Datentypen ist diese Spalte null |
| `datetime_precision` | `cardinal_number` | Wenn `data_type` einen Datums-, Zeit-, Zeitstempel- oder Intervalltyp identifiziert, enthält diese Spalte die deklarierte oder implizite Sekundenbruchteilgenauigkeit des Typs für dieses Attribut, also die Anzahl der Dezimalstellen nach dem Dezimalpunkt im Sekundenwert. Für alle anderen Datentypen ist diese Spalte null |
| `interval_type` | `character_data` | Wenn `data_type` einen Intervalltyp identifiziert, enthält diese Spalte die Angabe, welche Felder die Intervalle für dieses Attribut enthalten, zum Beispiel `YEAR TO MONTH`, `DAY TO SECOND` usw. Wenn keine Feldbeschränkungen angegeben wurden, das Intervall also alle Felder akzeptiert, und für alle anderen Datentypen ist diese Spalte null |
| `interval_precision` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist; siehe `datetime_precision` für die Sekundenbruchteilgenauigkeit von Attributen mit Intervalltyp |
| `attribute_udt_catalog` | `sql_identifier` | Name der Datenbank, in der der Datentyp des Attributs definiert ist; immer die aktuelle Datenbank |
| `attribute_udt_schema` | `sql_identifier` | Name des Schemas, in dem der Datentyp des Attributs definiert ist |
| `attribute_udt_name` | `sql_identifier` | Name des Datentyps des Attributs |
| `scope_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `scope_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `scope_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `maximum_cardinality` | `cardinal_number` | Immer null, weil Arrays in PostgreSQL immer eine unbegrenzte maximale Kardinalität haben |
| `dtd_identifier` | `sql_identifier` | Ein Bezeichner des Datentypdeskriptors des Attributs, eindeutig unter den Datentypdeskriptoren, die zum zusammengesetzten Typ gehören. Dies ist vor allem für Joins mit anderen Instanzen solcher Bezeichner nützlich. Das genaue Format des Bezeichners ist nicht definiert und es wird nicht garantiert, dass es in zukünftigen Versionen gleich bleibt |
| `is_derived_reference_attribute` | `yes_or_no` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |

Siehe auch [Abschnitt 35.17](35_Das_Information_Schema.md#3517-columns), eine ähnlich strukturierte Sicht, für weitere Informationen zu einigen Spalten.

## 35.7. character_sets

Die Sicht `character_sets` identifiziert die in der aktuellen Datenbank verfügbaren Zeichensätze. Da PostgreSQL innerhalb einer Datenbank nicht mehrere Zeichensätze unterstützt, zeigt diese Sicht nur einen Eintrag: die Datenbankkodierung.

Beachten Sie, wie die folgenden Begriffe im SQL-Standard verwendet werden:

- `character repertoire`: Eine abstrakte Sammlung von Zeichen, zum Beispiel `UNICODE`, `UCS` oder `LATIN1`. Sie wird nicht als SQL-Objekt offengelegt, ist aber in dieser Sicht sichtbar.
- `character encoding form`: Eine Kodierung eines Zeichenvorrats. Die meisten älteren Zeichenvorräte verwenden nur eine Kodierungsform, daher gibt es für sie keine eigenen Namen; zum Beispiel ist `LATIN2` eine Kodierungsform, die auf den Zeichenvorrat `LATIN2` angewendet wird. Unicode hat dagegen Kodierungsformen wie `UTF8`, `UTF16` usw. (nicht alle werden von PostgreSQL unterstützt). Kodierungsformen werden nicht als SQL-Objekte offengelegt, sind aber in dieser Sicht sichtbar.
- `character set`: Ein benanntes SQL-Objekt, das einen Zeichenvorrat, eine Zeichenkodierung und eine Standardsortierfolge identifiziert. Ein vordefinierter Zeichensatz hätte typischerweise denselben Namen wie eine Kodierungsform, Benutzer könnten aber andere Namen definieren. Der Zeichensatz `UTF8` würde zum Beispiel typischerweise den Zeichenvorrat `UCS`, die Kodierungsform `UTF8` und eine Standardsortierfolge identifizieren.

Eine »Kodierung« in PostgreSQL können Sie entweder als Zeichensatz oder als Zeichenkodierungsform betrachten. Beide haben denselben Namen, und in einer Datenbank kann es nur eine davon geben.

**Tabelle 35.5. Spalten von `character_sets`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `character_set_catalog` | `sql_identifier` | Zeichensätze sind derzeit nicht als Schemaobjekte implementiert, daher ist diese Spalte null |
| `character_set_schema` | `sql_identifier` | Zeichensätze sind derzeit nicht als Schemaobjekte implementiert, daher ist diese Spalte null |
| `character_set_name` | `sql_identifier` | Name des Zeichensatzes; derzeit implementiert als Anzeige des Namens der Datenbankkodierung |
| `character_repertoire` | `sql_identifier` | Zeichenvorrat; zeigt `UCS`, wenn die Kodierung `UTF8` ist, sonst einfach den Kodierungsnamen |
| `form_of_use` | `sql_identifier` | Zeichenkodierungsform; entspricht der Datenbankkodierung |
| `default_collate_catalog` | `sql_identifier` | Name der Datenbank, die die Standardsortierfolge enthält; immer die aktuelle Datenbank, wenn eine Sortierfolge identifiziert wird |
| `default_collate_schema` | `sql_identifier` | Name des Schemas, das die Standardsortierfolge enthält |
| `default_collate_name` | `sql_identifier` | Name der Standardsortierfolge. Die Standardsortierfolge wird als die Sortierfolge identifiziert, die zu den Einstellungen `COLLATE` und `CTYPE` der aktuellen Datenbank passt. Wenn es keine solche Sortierfolge gibt, sind diese Spalte und die zugehörigen Schema- und Katalogspalten null |

## 35.8. check_constraint_routine_usage

Die Sicht `check_constraint_routine_usage` identifiziert Routinen (Funktionen und Prozeduren), die von einem Check-Constraint verwendet werden. Angezeigt werden nur Routinen, deren Eigentümer eine derzeit aktivierte Rolle ist.

**Tabelle 35.6. Spalten von `check_constraint_routine_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `constraint_catalog` | `sql_identifier` | Name der Datenbank, die das Constraint enthält; immer die aktuelle Datenbank |
| `constraint_schema` | `sql_identifier` | Name des Schemas, das das Constraint enthält |
| `constraint_name` | `sql_identifier` | Name des Constraints |
| `specific_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `specific_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `specific_name` | `sql_identifier` | Der »spezifische Name« der Funktion. Siehe [Abschnitt 35.45](35_Das_Information_Schema.md#3545-routines) für weitere Informationen. |

## 35.9. check_constraints

Die Sicht `check_constraints` enthält alle Check-Constraints, die entweder auf einer Tabelle oder auf einer Domain definiert sind und deren Eigentümer eine derzeit aktivierte Rolle ist. Der Eigentümer der Tabelle oder Domain ist der Eigentümer des Constraints.

Der SQL-Standard betrachtet Not-Null-Constraints als Check-Constraints mit einem Ausdruck `CHECK (column_name IS NOT NULL)`. Not-Null-Constraints sind daher ebenfalls hier enthalten und haben keine eigene Sicht.

**Tabelle 35.7. Spalten von `check_constraints`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `constraint_catalog` | `sql_identifier` | Name der Datenbank, die das Constraint enthält; immer die aktuelle Datenbank |
| `constraint_schema` | `sql_identifier` | Name des Schemas, das das Constraint enthält |
| `constraint_name` | `sql_identifier` | Name des Constraints |
| `check_clause` | `character_data` | Der Check-Ausdruck des Check-Constraints |

## 35.10. collations

Die Sicht `collations` enthält die in der aktuellen Datenbank verfügbaren Sortierfolgen.

**Tabelle 35.8. Spalten von `collations`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `collation_catalog` | `sql_identifier` | Name der Datenbank, die die Sortierfolge enthält; immer die aktuelle Datenbank |
| `collation_schema` | `sql_identifier` | Name des Schemas, das die Sortierfolge enthält |
| `collation_name` | `sql_identifier` | Name der Standardsortierfolge |
| `pad_attribute` | `character_data` | Immer `NO PAD`; die Alternative `PAD SPACE` wird von PostgreSQL nicht unterstützt |

## 35.11. collation_character_set_applicability

Die Sicht `collation_character_set_applicability` identifiziert, auf welchen Zeichensatz die verfügbaren Sortierfolgen anwendbar sind. In PostgreSQL gibt es pro Datenbank nur einen Zeichensatz (siehe die Erläuterung in [Abschnitt 35.7](35_Das_Information_Schema.md#357-charactersets)), daher liefert diese Sicht nur wenige nützliche Informationen.

**Tabelle 35.9. Spalten von `collation_character_set_applicability`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `collation_catalog` | `sql_identifier` | Name der Datenbank, die die Sortierfolge enthält; immer die aktuelle Datenbank |
| `collation_schema` | `sql_identifier` | Name des Schemas, das die Sortierfolge enthält |
| `collation_name` | `sql_identifier` | Name der Standardsortierfolge |
| `character_set_catalog` | `sql_identifier` | Zeichensätze sind derzeit nicht als Schemaobjekte implementiert, daher ist diese Spalte null |
| `character_set_schema` | `sql_identifier` | Zeichensätze sind derzeit nicht als Schemaobjekte implementiert, daher ist diese Spalte null |
| `character_set_name` | `sql_identifier` | Name des Zeichensatzes |

## 35.12. column_column_usage

Die Sicht `column_column_usage` identifiziert alle generierten Spalten, die von einer anderen Basisspalte in derselben Tabelle abhängen. Enthalten sind nur Tabellen, deren Eigentümer eine derzeit aktivierte Rolle ist.

**Tabelle 35.10. Spalten von `column_column_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält |
| `table_name` | `sql_identifier` | Name der Tabelle |
| `column_name` | `sql_identifier` | Name der Basisspalte, von der eine generierte Spalte abhängt |
| `dependent_column` | `sql_identifier` | Name der generierten Spalte |

## 35.13. column_domain_usage

Die Sicht `column_domain_usage` identifiziert alle Spalten einer Tabelle oder Sicht, die eine in der aktuellen Datenbank definierte Domain verwenden, deren Eigentümer eine derzeit aktivierte Rolle ist.

**Tabelle 35.11. Spalten von `column_domain_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `domain_catalog` | `sql_identifier` | Name der Datenbank, die die Domain enthält; immer die aktuelle Datenbank |
| `domain_schema` | `sql_identifier` | Name des Schemas, das die Domain enthält |
| `domain_name` | `sql_identifier` | Name der Domain |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält |
| `table_name` | `sql_identifier` | Name der Tabelle |
| `column_name` | `sql_identifier` | Name der Spalte |

## 35.14. column_options

Die Sicht `column_options` enthält alle Optionen, die für Spalten von Fremdtabellen in der aktuellen Datenbank definiert sind. Angezeigt werden nur solche Fremdtabellenspalten, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.12. Spalten von `column_options`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Fremdtabelle enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Fremdtabelle enthält |
| `table_name` | `sql_identifier` | Name der Fremdtabelle |
| `column_name` | `sql_identifier` | Name der Spalte |
| `option_name` | `sql_identifier` | Name einer Option |
| `option_value` | `character_data` | Wert der Option |

## 35.15. column_privileges

Die Sicht `column_privileges` identifiziert alle Privilegien, die auf Spalten an eine derzeit aktivierte Rolle oder von einer derzeit aktivierten Rolle gewährt wurden. Für jede Kombination aus Spalte, Gewährer und Empfänger gibt es eine Zeile.

Wenn ein Privileg für eine ganze Tabelle gewährt wurde, erscheint es in dieser Sicht als Gewährung für jede Spalte, allerdings nur für die Privilegtypen, bei denen Spaltengranularität möglich ist: `SELECT`, `INSERT`, `UPDATE`, `REFERENCES`.

**Tabelle 35.13. Spalten von `column_privileges`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `grantor` | `sql_identifier` | Name der Rolle, die das Privileg gewährt hat |
| `grantee` | `sql_identifier` | Name der Rolle, der das Privileg gewährt wurde |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält, welche die Spalte enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält, welche die Spalte enthält |
| `table_name` | `sql_identifier` | Name der Tabelle, die die Spalte enthält |
| `column_name` | `sql_identifier` | Name der Spalte |
| `privilege_type` | `character_data` | Typ des Privilegs: `SELECT`, `INSERT`, `UPDATE` oder `REFERENCES` |
| `is_grantable` | `yes_or_no` | `YES`, wenn das Privileg weitergewährt werden kann, sonst `NO` |

## 35.16. column_udt_usage

Die Sicht `column_udt_usage` identifiziert alle Spalten, die Datentypen verwenden, deren Eigentümer eine derzeit aktivierte Rolle ist. Beachten Sie, dass eingebaute Datentypen in PostgreSQL wie benutzerdefinierte Typen behandelt werden und daher ebenfalls hier enthalten sind. Siehe auch [Abschnitt 35.17](35_Das_Information_Schema.md#3517-columns) für Details.

**Tabelle 35.14. Spalten von `column_udt_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `udt_catalog` | `sql_identifier` | Name der Datenbank, in der der Datentyp der Spalte definiert ist; bei Domains der zugrunde liegende Typ, falls zutreffend; immer die aktuelle Datenbank |
| `udt_schema` | `sql_identifier` | Name des Schemas, in dem der Datentyp der Spalte definiert ist; bei Domains der zugrunde liegende Typ, falls zutreffend |
| `udt_name` | `sql_identifier` | Name des Datentyps der Spalte; bei Domains der zugrunde liegende Typ, falls zutreffend |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält |
| `table_name` | `sql_identifier` | Name der Tabelle |
| `column_name` | `sql_identifier` | Name der Spalte |

## 35.17. columns

Die Sicht `columns` enthält Informationen über alle Tabellenspalten oder Sichtspalten in der Datenbank. Systemspalten wie `ctid` usw. sind nicht enthalten. Angezeigt werden nur Spalten, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.15. Spalten von `columns`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält |
| `table_name` | `sql_identifier` | Name der Tabelle |
| `column_name` | `sql_identifier` | Name der Spalte |
| `ordinal_position` | `cardinal_number` | Ordnungsposition der Spalte innerhalb der Tabelle; die Zählung beginnt bei 1 |
| `column_default` | `character_data` | Default-Ausdruck der Spalte |
| `is_nullable` | `yes_or_no` | `YES`, wenn die Spalte möglicherweise null sein kann, `NO`, wenn bekannt ist, dass sie nicht null sein kann. Ein Not-Null-Constraint ist eine Möglichkeit, wie bekannt sein kann, dass eine Spalte nicht null sein kann, es kann aber weitere geben |
| `data_type` | `character_data` | Datentyp der Spalte, wenn es ein eingebauter Typ ist; `ARRAY`, wenn es ein Array ist (siehe in diesem Fall die Sicht `element_types`); andernfalls `USER-DEFINED` (in diesem Fall wird der Typ in `udt_name` und den zugehörigen Spalten identifiziert). Wenn die Spalte auf einer Domain basiert, bezieht sich diese Spalte auf den zugrunde liegenden Typ der Domain; die Domain wird in `domain_name` und den zugehörigen Spalten identifiziert |
| `character_maximum_length` | `cardinal_number` | Wenn `data_type` einen Zeichenketten- oder Bitketten-Typ identifiziert, die deklarierte maximale Länge; null für alle anderen Datentypen oder wenn keine maximale Länge deklariert wurde |
| `character_octet_length` | `cardinal_number` | Wenn `data_type` einen Zeichentyp identifiziert, die maximal mögliche Länge eines Werts in Oktetten (Bytes); null für alle anderen Datentypen. Die maximale Oktettlänge hängt von der deklarierten maximalen Zeichenlänge (siehe oben) und der Serverkodierung ab |
| `numeric_precision` | `cardinal_number` | Wenn `data_type` einen numerischen Typ identifiziert, enthält diese Spalte die deklarierte oder implizite Genauigkeit des Typs für diese Spalte. Die Genauigkeit gibt die Anzahl signifikanter Ziffern an. Sie kann dezimal (Basis 10) oder binär (Basis 2) ausgedrückt werden, wie in der Spalte `numeric_precision_radix` angegeben. Für alle anderen Datentypen ist diese Spalte null |
| `numeric_precision_radix` | `cardinal_number` | Wenn `data_type` einen numerischen Typ identifiziert, gibt diese Spalte an, in welcher Basis die Werte in den Spalten `numeric_precision` und `numeric_scale` ausgedrückt werden. Der Wert ist entweder 2 oder 10. Für alle anderen Datentypen ist diese Spalte null |
| `numeric_scale` | `cardinal_number` | Wenn `data_type` einen exakten numerischen Typ identifiziert, enthält diese Spalte die deklarierte oder implizite Skalierung des Typs für diese Spalte. Die Skalierung gibt die Anzahl signifikanter Ziffern rechts vom Dezimalpunkt an. Sie kann dezimal (Basis 10) oder binär (Basis 2) ausgedrückt werden, wie in der Spalte `numeric_precision_radix` angegeben. Für alle anderen Datentypen ist diese Spalte null |
| `datetime_precision` | `cardinal_number` | Wenn `data_type` einen Datums-, Zeit-, Zeitstempel- oder Intervalltyp identifiziert, enthält diese Spalte die deklarierte oder implizite Sekundenbruchteilgenauigkeit des Typs für diese Spalte, also die Anzahl der Dezimalstellen nach dem Dezimalpunkt im Sekundenwert. Für alle anderen Datentypen ist diese Spalte null |
| `interval_type` | `character_data` | Wenn `data_type` einen Intervalltyp identifiziert, enthält diese Spalte die Angabe, welche Felder die Intervalle für diese Spalte enthalten, zum Beispiel `YEAR TO MONTH`, `DAY TO SECOND` usw. Wenn keine Feldbeschränkungen angegeben wurden, das Intervall also alle Felder akzeptiert, und für alle anderen Datentypen ist diese Spalte null |
| `interval_precision` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist; siehe `datetime_precision` für die Sekundenbruchteilgenauigkeit von Spalten mit Intervalltyp |
| `character_set_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `collation_catalog` | `sql_identifier` | Name der Datenbank, die die Sortierfolge der Spalte enthält; immer die aktuelle Datenbank. Null, wenn die Standardsortierfolge gilt oder der Datentyp der Spalte nicht sortierbar ist |
| `collation_schema` | `sql_identifier` | Name des Schemas, das die Sortierfolge der Spalte enthält; null, wenn die Standardsortierfolge gilt oder der Datentyp der Spalte nicht sortierbar ist |
| `collation_name` | `sql_identifier` | Name der Sortierfolge der Spalte; null, wenn die Standardsortierfolge gilt oder der Datentyp der Spalte nicht sortierbar ist |
| `domain_catalog` | `sql_identifier` | Wenn die Spalte einen Domain-Typ hat, der Name der Datenbank, in der die Domain definiert ist; immer die aktuelle Datenbank. Andernfalls null |
| `domain_schema` | `sql_identifier` | Wenn die Spalte einen Domain-Typ hat, der Name des Schemas, in dem die Domain definiert ist. Andernfalls null |
| `domain_name` | `sql_identifier` | Wenn die Spalte einen Domain-Typ hat, der Name der Domain. Andernfalls null |
| `udt_catalog` | `sql_identifier` | Name der Datenbank, in der der Datentyp der Spalte definiert ist; bei Domains der zugrunde liegende Typ, falls zutreffend; immer die aktuelle Datenbank |
| `udt_schema` | `sql_identifier` | Name des Schemas, in dem der Datentyp der Spalte definiert ist; bei Domains der zugrunde liegende Typ, falls zutreffend |
| `udt_name` | `sql_identifier` | Name des Datentyps der Spalte; bei Domains der zugrunde liegende Typ, falls zutreffend |
| `scope_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `scope_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `scope_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `maximum_cardinality` | `cardinal_number` | Immer null, weil Arrays in PostgreSQL immer eine unbegrenzte maximale Kardinalität haben |
| `dtd_identifier` | `sql_identifier` | Ein Bezeichner des Datentypdeskriptors der Spalte, eindeutig unter den Datentypdeskriptoren, die zur Tabelle gehören. Dies ist vor allem für Joins mit anderen Instanzen solcher Bezeichner nützlich. Das genaue Format des Bezeichners ist nicht definiert und es wird nicht garantiert, dass es in zukünftigen Versionen gleich bleibt |
| `is_self_referencing` | `yes_or_no` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `is_identity` | `yes_or_no` | Wenn die Spalte eine Identity-Spalte ist, dann `YES`, sonst `NO` |
| `identity_generation` | `character_data` | Wenn die Spalte eine Identity-Spalte ist, dann `ALWAYS` oder `BY DEFAULT`, entsprechend der Definition der Spalte |
| `identity_start` | `character_data` | Wenn die Spalte eine Identity-Spalte ist, der Startwert der internen Sequenz, sonst null |
| `identity_increment` | `character_data` | Wenn die Spalte eine Identity-Spalte ist, die Schrittweite der internen Sequenz, sonst null |
| `identity_maximum` | `character_data` | Wenn die Spalte eine Identity-Spalte ist, der Maximalwert der internen Sequenz, sonst null |
| `identity_minimum` | `character_data` | Wenn die Spalte eine Identity-Spalte ist, der Minimalwert der internen Sequenz, sonst null |
| `identity_cycle` | `yes_or_no` | Wenn die Spalte eine Identity-Spalte ist, dann `YES`, wenn die interne Sequenz umläuft, oder `NO`, wenn nicht; andernfalls null |
| `is_generated` | `character_data` | Wenn die Spalte eine generierte Spalte ist, dann `ALWAYS`, sonst `NEVER` |
| `generation_expression` | `character_data` | Wenn die Spalte eine generierte Spalte ist, dann der Generierungsausdruck, sonst null |
| `is_updatable` | `yes_or_no` | `YES`, wenn die Spalte aktualisierbar ist, sonst `NO`; Spalten in Basistabellen sind immer aktualisierbar, Spalten in Sichten nicht unbedingt |

Da Datentypen in SQL auf verschiedene Arten definiert werden können und PostgreSQL zusätzliche Möglichkeiten zur Definition von Datentypen enthält, kann ihre Darstellung im Information Schema etwas schwierig sein. Die Spalte `data_type` soll den zugrunde liegenden eingebauten Typ der Spalte identifizieren. In PostgreSQL bedeutet das, dass der Typ im Systemkatalogschema `pg_catalog` definiert ist. Diese Spalte kann nützlich sein, wenn die Anwendung die bekannten eingebauten Typen speziell behandeln kann, zum Beispiel numerische Typen anders formatieren oder die Daten in den Genauigkeitsspalten verwenden kann.

Die Spalten `udt_name`, `udt_schema` und `udt_catalog` identifizieren immer den zugrunde liegenden Datentyp der Spalte, auch wenn die Spalte auf einer Domain basiert. Da PostgreSQL eingebaute Typen wie benutzerdefinierte Typen behandelt, erscheinen eingebaute Typen hier ebenfalls; dies ist eine Erweiterung des SQL-Standards. Diese Spalten sollten verwendet werden, wenn eine Anwendung Daten je nach Typ unterschiedlich verarbeiten möchte, weil es in diesem Fall keine Rolle spielt, ob die Spalte tatsächlich auf einer Domain basiert. Wenn die Spalte auf einer Domain basiert, wird die Identität der Domain in den Spalten `domain_name`, `domain_schema` und `domain_catalog` gespeichert. Wenn Sie Spalten ihren zugehörigen Datentypen zuordnen und Domains als eigene Typen behandeln möchten, könnten Sie zum Beispiel `coalesce(domain_name, udt_name)` usw. schreiben.

## 35.18. constraint_column_usage

Die Sicht `constraint_column_usage` identifiziert alle Spalten in der aktuellen Datenbank, die von einem Constraint verwendet werden. Angezeigt werden nur Spalten, die in einer Tabelle enthalten sind, deren Eigentümer eine derzeit aktivierte Rolle ist. Für ein Check-Constraint identifiziert diese Sicht die Spalten, die im Check-Ausdruck verwendet werden. Für ein Not-Null-Constraint identifiziert sie die Spalte, auf der das Constraint definiert ist. Für ein Fremdschlüssel-Constraint identifiziert sie die Spalten, auf die der Fremdschlüssel verweist. Für ein Unique- oder Primary-Key-Constraint identifiziert sie die eingeschränkten Spalten.

**Tabelle 35.16. Spalten von `constraint_column_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält, welche die von einem Constraint verwendete Spalte enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält, welche die von einem Constraint verwendete Spalte enthält |
| `table_name` | `sql_identifier` | Name der Tabelle, die die von einem Constraint verwendete Spalte enthält |
| `column_name` | `sql_identifier` | Name der Spalte, die von einem Constraint verwendet wird |
| `constraint_catalog` | `sql_identifier` | Name der Datenbank, die das Constraint enthält; immer die aktuelle Datenbank |
| `constraint_schema` | `sql_identifier` | Name des Schemas, das das Constraint enthält |
| `constraint_name` | `sql_identifier` | Name des Constraints |

## 35.19. constraint_table_usage

Die Sicht `constraint_table_usage` identifiziert alle Tabellen in der aktuellen Datenbank, die von einem Constraint verwendet werden und deren Eigentümer eine derzeit aktivierte Rolle ist. Dies unterscheidet sich von der Sicht `table_constraints`, die alle Tabellen-Constraints zusammen mit der Tabelle identifiziert, auf der sie definiert sind. Für ein Fremdschlüssel-Constraint identifiziert diese Sicht die Tabelle, auf die der Fremdschlüssel verweist. Für ein Unique- oder Primary-Key-Constraint identifiziert sie einfach die Tabelle, zu der das Constraint gehört. Check-Constraints und Not-Null-Constraints sind in dieser Sicht nicht enthalten.

**Tabelle 35.17. Spalten von `constraint_table_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die von einem Constraint verwendete Tabelle enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die von einem Constraint verwendete Tabelle enthält |
| `table_name` | `sql_identifier` | Name der Tabelle, die von einem Constraint verwendet wird |
| `constraint_catalog` | `sql_identifier` | Name der Datenbank, die das Constraint enthält; immer die aktuelle Datenbank |
| `constraint_schema` | `sql_identifier` | Name des Schemas, das das Constraint enthält |
| `constraint_name` | `sql_identifier` | Name des Constraints |

## 35.20. data_type_privileges

Die Sicht `data_type_privileges` identifiziert alle Datentypdeskriptoren, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer des beschriebenen Objekts ist oder weil er ein Privileg darauf besitzt. Ein Datentypdeskriptor wird immer dann erzeugt, wenn ein Datentyp in der Definition einer Tabellenspalte, einer Domain oder einer Funktion (als Parameter oder Rückgabetyp) verwendet wird, und speichert Informationen darüber, wie der Datentyp in dieser Instanz verwendet wird, zum Beispiel die deklarierte maximale Länge, falls zutreffend. Jedem Datentypdeskriptor wird ein beliebiger Bezeichner zugewiesen, der unter den Datentypdeskriptor-Bezeichnern eines Objekts (Tabelle, Domain, Funktion) eindeutig ist. Diese Sicht ist für Anwendungen vermutlich nicht nützlich, wird aber verwendet, um einige andere Sichten im Information Schema zu definieren.

**Tabelle 35.18. Spalten von `data_type_privileges`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `object_catalog` | `sql_identifier` | Name der Datenbank, die das beschriebene Objekt enthält; immer die aktuelle Datenbank |
| `object_schema` | `sql_identifier` | Name des Schemas, das das beschriebene Objekt enthält |
| `object_name` | `sql_identifier` | Name des beschriebenen Objekts |
| `object_type` | `character_data` | Typ des beschriebenen Objekts: einer von `TABLE` (der Datentypdeskriptor gehört zu einer Spalte dieser Tabelle), `DOMAIN` (der Datentypdeskriptor gehört zu dieser Domain), `ROUTINE` (der Datentypdeskriptor gehört zu einem Parameter oder zum Rückgabetyp dieser Funktion) |
| `dtd_identifier` | `sql_identifier` | Bezeichner des Datentypdeskriptors, der unter den Datentypdeskriptoren desselben Objekts eindeutig ist |

## 35.21. domain_constraints

Die Sicht `domain_constraints` enthält alle Constraints, die zu Domains gehören, die in der aktuellen Datenbank definiert sind. Angezeigt werden nur solche Domains, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.19. Spalten von `domain_constraints`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `constraint_catalog` | `sql_identifier` | Name der Datenbank, die das Constraint enthält; immer die aktuelle Datenbank |
| `constraint_schema` | `sql_identifier` | Name des Schemas, das das Constraint enthält |
| `constraint_name` | `sql_identifier` | Name des Constraints |
| `domain_catalog` | `sql_identifier` | Name der Datenbank, die die Domain enthält; immer die aktuelle Datenbank |
| `domain_schema` | `sql_identifier` | Name des Schemas, das die Domain enthält |
| `domain_name` | `sql_identifier` | Name der Domain |
| `is_deferrable` | `yes_or_no` | `YES`, wenn das Constraint aufschiebbar ist, sonst `NO` |
| `initially_deferred` | `yes_or_no` | `YES`, wenn das Constraint aufschiebbar und anfänglich aufgeschoben ist, sonst `NO` |

## 35.22. domain_udt_usage

Die Sicht `domain_udt_usage` identifiziert alle Domains, die auf Datentypen basieren, deren Eigentümer eine derzeit aktivierte Rolle ist. Beachten Sie, dass eingebaute Datentypen in PostgreSQL wie benutzerdefinierte Typen behandelt werden und daher ebenfalls hier enthalten sind.

**Tabelle 35.20. Spalten von `domain_udt_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `udt_catalog` | `sql_identifier` | Name der Datenbank, in der der Datentyp der Domain definiert ist; immer die aktuelle Datenbank |
| `udt_schema` | `sql_identifier` | Name des Schemas, in dem der Datentyp der Domain definiert ist |
| `udt_name` | `sql_identifier` | Name des Datentyps der Domain |
| `domain_catalog` | `sql_identifier` | Name der Datenbank, die die Domain enthält; immer die aktuelle Datenbank |
| `domain_schema` | `sql_identifier` | Name des Schemas, das die Domain enthält |
| `domain_name` | `sql_identifier` | Name der Domain |

## 35.23. domains

Die Sicht `domains` enthält alle Domains, die in der aktuellen Datenbank definiert sind. Angezeigt werden nur solche Domains, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.21. Spalten von `domains`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `domain_catalog` | `sql_identifier` | Name der Datenbank, die die Domain enthält; immer die aktuelle Datenbank |
| `domain_schema` | `sql_identifier` | Name des Schemas, das die Domain enthält |
| `domain_name` | `sql_identifier` | Name der Domain |
| `data_type` | `character_data` | Datentyp der Domain, wenn es ein eingebauter Typ ist; `ARRAY`, wenn es ein Array ist (siehe in diesem Fall die Sicht `element_types`); andernfalls `USER-DEFINED` (in diesem Fall wird der Typ in `udt_name` und den zugehörigen Spalten identifiziert) |
| `character_maximum_length` | `cardinal_number` | Wenn die Domain einen Zeichenketten- oder Bitketten-Typ hat, die deklarierte maximale Länge; null für alle anderen Datentypen oder wenn keine maximale Länge deklariert wurde |
| `character_octet_length` | `cardinal_number` | Wenn die Domain einen Zeichentyp hat, die maximal mögliche Länge eines Werts in Oktetten (Bytes); null für alle anderen Datentypen. Die maximale Oktettlänge hängt von der deklarierten maximalen Zeichenlänge (siehe oben) und der Serverkodierung ab |
| `character_set_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `collation_catalog` | `sql_identifier` | Name der Datenbank, die die Sortierfolge der Domain enthält; immer die aktuelle Datenbank. Null, wenn die Standardsortierfolge gilt oder der Datentyp der Domain nicht sortierbar ist |
| `collation_schema` | `sql_identifier` | Name des Schemas, das die Sortierfolge der Domain enthält; null, wenn die Standardsortierfolge gilt oder der Datentyp der Domain nicht sortierbar ist |
| `collation_name` | `sql_identifier` | Name der Sortierfolge der Domain; null, wenn die Standardsortierfolge gilt oder der Datentyp der Domain nicht sortierbar ist |
| `numeric_precision` | `cardinal_number` | Wenn die Domain einen numerischen Typ hat, enthält diese Spalte die deklarierte oder implizite Genauigkeit des Typs für diese Domain. Die Genauigkeit gibt die Anzahl signifikanter Ziffern an. Sie kann dezimal (Basis 10) oder binär (Basis 2) ausgedrückt werden, wie in der Spalte `numeric_precision_radix` angegeben. Für alle anderen Datentypen ist diese Spalte null |
| `numeric_precision_radix` | `cardinal_number` | Wenn die Domain einen numerischen Typ hat, gibt diese Spalte an, in welcher Basis die Werte in den Spalten `numeric_precision` und `numeric_scale` ausgedrückt werden. Der Wert ist entweder 2 oder 10. Für alle anderen Datentypen ist diese Spalte null |
| `numeric_scale` | `cardinal_number` | Wenn die Domain einen exakten numerischen Typ hat, enthält diese Spalte die deklarierte oder implizite Skalierung des Typs für diese Domain. Die Skalierung gibt die Anzahl signifikanter Ziffern rechts vom Dezimalpunkt an. Sie kann dezimal (Basis 10) oder binär (Basis 2) ausgedrückt werden, wie in der Spalte `numeric_precision_radix` angegeben. Für alle anderen Datentypen ist diese Spalte null |
| `datetime_precision` | `cardinal_number` | Wenn `data_type` einen Datums-, Zeit-, Zeitstempel- oder Intervalltyp identifiziert, enthält diese Spalte die deklarierte oder implizite Sekundenbruchteilgenauigkeit des Typs für diese Domain, also die Anzahl der Dezimalstellen nach dem Dezimalpunkt im Sekundenwert. Für alle anderen Datentypen ist diese Spalte null |
| `interval_type` | `character_data` | Wenn `data_type` einen Intervalltyp identifiziert, enthält diese Spalte die Angabe, welche Felder die Intervalle für diese Domain enthalten, zum Beispiel `YEAR TO MONTH`, `DAY TO SECOND` usw. Wenn keine Feldbeschränkungen angegeben wurden, das Intervall also alle Felder akzeptiert, und für alle anderen Datentypen ist diese Spalte null |
| `interval_precision` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist; siehe `datetime_precision` für die Sekundenbruchteilgenauigkeit von Domains mit Intervalltyp |
| `domain_default` | `character_data` | Default-Ausdruck der Domain |
| `udt_catalog` | `sql_identifier` | Name der Datenbank, in der der Datentyp der Domain definiert ist; immer die aktuelle Datenbank |
| `udt_schema` | `sql_identifier` | Name des Schemas, in dem der Datentyp der Domain definiert ist |
| `udt_name` | `sql_identifier` | Name des Datentyps der Domain |
| `scope_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `scope_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `scope_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `maximum_cardinality` | `cardinal_number` | Immer null, weil Arrays in PostgreSQL immer eine unbegrenzte maximale Kardinalität haben |
| `dtd_identifier` | `sql_identifier` | Ein Bezeichner des Datentypdeskriptors der Domain, eindeutig unter den Datentypdeskriptoren, die zur Domain gehören; das ist trivial, weil eine Domain nur einen Datentypdeskriptor enthält. Dies ist vor allem für Joins mit anderen Instanzen solcher Bezeichner nützlich. Das genaue Format des Bezeichners ist nicht definiert und es wird nicht garantiert, dass es in zukünftigen Versionen gleich bleibt |

## 35.24. element_types

Die Sicht `element_types` enthält die Datentypdeskriptoren der Elemente von Arrays. Wenn eine Tabellenspalte, ein Attribut eines zusammengesetzten Typs, eine Domain, ein Funktionsparameter oder der Rückgabewert einer Funktion als Array-Typ definiert ist, enthält die jeweilige Information-Schema-Sicht in der Spalte `data_type` nur `ARRAY`. Um Informationen über den Elementtyp des Arrays zu erhalten, können Sie die jeweilige Sicht mit dieser Sicht verbinden. Um zum Beispiel die Spalten einer Tabelle mit Datentypen und gegebenenfalls Array-Elementtypen anzuzeigen, könnten Sie schreiben:

```sql
SELECT c.column_name, c.data_type, e.data_type AS element_type
FROM information_schema.columns c LEFT JOIN
 information_schema.element_types e
     ON ((c.table_catalog, c.table_schema, c.table_name, 'TABLE',
 c.dtd_identifier)
       = (e.object_catalog, e.object_schema, e.object_name,
 e.object_type, e.collection_type_identifier))
WHERE c.table_schema = '...' AND c.table_name = '...'
ORDER BY c.ordinal_position;
```

Diese Sicht enthält nur Objekte, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.22. Spalten von `element_types`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `object_catalog` | `sql_identifier` | Name der Datenbank, die das Objekt enthält, das das beschriebene Array verwendet; immer die aktuelle Datenbank |
| `object_schema` | `sql_identifier` | Name des Schemas, das das Objekt enthält, das das beschriebene Array verwendet |
| `object_name` | `sql_identifier` | Name des Objekts, das das beschriebene Array verwendet |
| `object_type` | `character_data` | Typ des Objekts, das das beschriebene Array verwendet: einer von `TABLE` (das Array wird von einer Spalte dieser Tabelle verwendet), `USER-DEFINED TYPE` (das Array wird von einem Attribut dieses zusammengesetzten Typs verwendet), `DOMAIN` (das Array wird von dieser Domain verwendet), `ROUTINE` (das Array wird von einem Parameter oder vom Rückgabetyp dieser Funktion verwendet) |
| `collection_type_identifier` | `sql_identifier` | Bezeichner des Datentypdeskriptors des beschriebenen Arrays. Verwenden Sie diesen Wert, um mit den `dtd_identifier`-Spalten anderer Information-Schema-Sichten zu verbinden |
| `data_type` | `character_data` | Datentyp der Array-Elemente, wenn es ein eingebauter Typ ist; andernfalls `USER-DEFINED` (in diesem Fall wird der Typ in `udt_name` und den zugehörigen Spalten identifiziert) |
| `character_maximum_length` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Datentypen von Array-Elementen angewendet wird |
| `character_octet_length` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Datentypen von Array-Elementen angewendet wird |
| `character_set_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `collation_catalog` | `sql_identifier` | Name der Datenbank, die die Sortierfolge des Elementtyps enthält; immer die aktuelle Datenbank. Null, wenn die Standardsortierfolge gilt oder der Datentyp des Elements nicht sortierbar ist |
| `collation_schema` | `sql_identifier` | Name des Schemas, das die Sortierfolge des Elementtyps enthält; null, wenn die Standardsortierfolge gilt oder der Datentyp des Elements nicht sortierbar ist |
| `collation_name` | `sql_identifier` | Name der Sortierfolge des Elementtyps; null, wenn die Standardsortierfolge gilt oder der Datentyp des Elements nicht sortierbar ist |
| `numeric_precision` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Datentypen von Array-Elementen angewendet wird |
| `numeric_precision_radix` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Datentypen von Array-Elementen angewendet wird |
| `numeric_scale` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Datentypen von Array-Elementen angewendet wird |
| `datetime_precision` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Datentypen von Array-Elementen angewendet wird |
| `interval_type` | `character_data` | Immer null, da diese Information in PostgreSQL nicht auf Datentypen von Array-Elementen angewendet wird |
| `interval_precision` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Datentypen von Array-Elementen angewendet wird |
| `udt_catalog` | `sql_identifier` | Name der Datenbank, in der der Datentyp der Elemente definiert ist; immer die aktuelle Datenbank |
| `udt_schema` | `sql_identifier` | Name des Schemas, in dem der Datentyp der Elemente definiert ist |
| `udt_name` | `sql_identifier` | Name des Datentyps der Elemente |
| `scope_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `scope_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `scope_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `maximum_cardinality` | `cardinal_number` | Immer null, weil Arrays in PostgreSQL immer eine unbegrenzte maximale Kardinalität haben |
| `dtd_identifier` | `sql_identifier` | Ein Bezeichner des Datentypdeskriptors des Elements. Dies ist derzeit nicht nützlich |

## 35.25. enabled_roles

Die Sicht `enabled_roles` identifiziert die derzeit »aktivierten Rollen«. Die aktivierten Rollen sind rekursiv definiert als der aktuelle Benutzer zusammen mit allen Rollen, die den aktivierten Rollen mit automatischer Vererbung gewährt wurden. Anders gesagt sind dies alle Rollen, in denen der aktuelle Benutzer direkt oder indirekt automatisch vererbende Mitgliedschaft hat.

Für Berechtigungsprüfungen wird die Menge der »anwendbaren Rollen« verwendet, die größer sein kann als die Menge der aktivierten Rollen. Im Allgemeinen ist es daher besser, statt dieser Sicht die Sicht `applicable_roles` zu verwenden; siehe [Abschnitt 35.5](35_Das_Information_Schema.md#355-applicableroles) für Details zur Sicht `applicable_roles`.

**Tabelle 35.23. Spalten von `enabled_roles`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `role_name` | `sql_identifier` | Name einer Rolle |

## 35.26. foreign_data_wrapper_options

Die Sicht `foreign_data_wrapper_options` enthält alle Optionen, die für Foreign-Data Wrapper in der aktuellen Datenbank definiert sind. Angezeigt werden nur solche Foreign-Data Wrapper, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.24. Spalten von `foreign_data_wrapper_options`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `foreign_data_wrapper_catalog` | `sql_identifier` | Name der Datenbank, in der der Foreign-Data Wrapper definiert ist; immer die aktuelle Datenbank |
| `foreign_data_wrapper_name` | `sql_identifier` | Name des Foreign-Data Wrappers |
| `option_name` | `sql_identifier` | Name einer Option |
| `option_value` | `character_data` | Wert der Option |

## 35.27. foreign_data_wrappers

Die Sicht `foreign_data_wrappers` enthält alle Foreign-Data Wrapper, die in der aktuellen Datenbank definiert sind. Angezeigt werden nur solche Foreign-Data Wrapper, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.25. Spalten von `foreign_data_wrappers`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `foreign_data_wrapper_catalog` | `sql_identifier` | Name der Datenbank, die den Foreign-Data Wrapper enthält; immer die aktuelle Datenbank |
| `foreign_data_wrapper_name` | `sql_identifier` | Name des Foreign-Data Wrappers |
| `authorization_identifier` | `sql_identifier` | Name des Eigentümers des Foreign Servers |
| `library_name` | `character_data` | Dateiname der Bibliothek, die diesen Foreign-Data Wrapper implementiert |
| `foreign_data_wrapper_language` | `character_data` | Sprache, mit der dieser Foreign-Data Wrapper implementiert wurde |

## 35.28. foreign_server_options

Die Sicht `foreign_server_options` enthält alle Optionen, die für Foreign Servers in der aktuellen Datenbank definiert sind. Angezeigt werden nur solche Foreign Servers, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.26. Spalten von `foreign_server_options`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `foreign_server_catalog` | `sql_identifier` | Name der Datenbank, in der der Foreign Server definiert ist; immer die aktuelle Datenbank |
| `foreign_server_name` | `sql_identifier` | Name des Foreign Servers |
| `option_name` | `sql_identifier` | Name einer Option |
| `option_value` | `character_data` | Wert der Option |

## 35.29. foreign_servers

Die Sicht `foreign_servers` enthält alle Foreign Servers, die in der aktuellen Datenbank definiert sind. Angezeigt werden nur solche Foreign Servers, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.27. Spalten von `foreign_servers`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `foreign_server_catalog` | `sql_identifier` | Name der Datenbank, in der der Foreign Server definiert ist; immer die aktuelle Datenbank |
| `foreign_server_name` | `sql_identifier` | Name des Foreign Servers |
| `foreign_data_wrapper_catalog` | `sql_identifier` | Name der Datenbank, die den vom Foreign Server verwendeten Foreign-Data Wrapper enthält; immer die aktuelle Datenbank |
| `foreign_data_wrapper_name` | `sql_identifier` | Name des vom Foreign Server verwendeten Foreign-Data Wrappers |
| `foreign_server_type` | `character_data` | Typinformation des Foreign Servers, falls bei der Erstellung angegeben |
| `foreign_server_version` | `character_data` | Versionsinformation des Foreign Servers, falls bei der Erstellung angegeben |
| `authorization_identifier` | `sql_identifier` | Name des Eigentümers des Foreign Servers |

## 35.30. foreign_table_options

Die Sicht `foreign_table_options` enthält alle Optionen, die für Fremdtabellen in der aktuellen Datenbank definiert sind. Angezeigt werden nur solche Fremdtabellen, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.28. Spalten von `foreign_table_options`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `foreign_table_catalog` | `sql_identifier` | Name der Datenbank, die die Fremdtabelle enthält; immer die aktuelle Datenbank |
| `foreign_table_schema` | `sql_identifier` | Name des Schemas, das die Fremdtabelle enthält |
| `foreign_table_name` | `sql_identifier` | Name der Fremdtabelle |
| `option_name` | `sql_identifier` | Name einer Option |
| `option_value` | `character_data` | Wert der Option |

## 35.31. foreign_tables

Die Sicht `foreign_tables` enthält alle Fremdtabellen, die in der aktuellen Datenbank definiert sind. Angezeigt werden nur solche Fremdtabellen, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.29. Spalten von `foreign_tables`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `foreign_table_catalog` | `sql_identifier` | Name der Datenbank, in der die Fremdtabelle definiert ist; immer die aktuelle Datenbank |
| `foreign_table_schema` | `sql_identifier` | Name des Schemas, das die Fremdtabelle enthält |
| `foreign_table_name` | `sql_identifier` | Name der Fremdtabelle |
| `foreign_server_catalog` | `sql_identifier` | Name der Datenbank, in der der Foreign Server definiert ist; immer die aktuelle Datenbank |
| `foreign_server_name` | `sql_identifier` | Name des Foreign Servers |

## 35.32. key_column_usage

Die Sicht `key_column_usage` identifiziert alle Spalten in der aktuellen Datenbank, die durch ein Unique-, Primary-Key- oder Fremdschlüssel-Constraint eingeschränkt sind. Check-Constraints sind in dieser Sicht nicht enthalten. Angezeigt werden nur Spalten, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.30. Spalten von `key_column_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `constraint_catalog` | `sql_identifier` | Name der Datenbank, die das Constraint enthält; immer die aktuelle Datenbank |
| `constraint_schema` | `sql_identifier` | Name des Schemas, das das Constraint enthält |
| `constraint_name` | `sql_identifier` | Name des Constraints |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält, welche die durch dieses Constraint eingeschränkte Spalte enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält, welche die durch dieses Constraint eingeschränkte Spalte enthält |
| `table_name` | `sql_identifier` | Name der Tabelle, die die durch dieses Constraint eingeschränkte Spalte enthält |
| `column_name` | `sql_identifier` | Name der Spalte, die durch dieses Constraint eingeschränkt ist |
| `ordinal_position` | `cardinal_number` | Ordnungsposition der Spalte innerhalb des Constraint-Schlüssels; die Zählung beginnt bei 1 |
| `position_in_unique_constraint` | `cardinal_number` | Bei einem Fremdschlüssel-Constraint die Ordnungsposition der referenzierten Spalte innerhalb ihres Unique-Constraints; die Zählung beginnt bei 1. Andernfalls null |

## 35.33. parameters

Die Sicht `parameters` enthält Informationen über die Parameter (Argumente) aller Funktionen in der aktuellen Datenbank. Angezeigt werden nur Funktionen, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.31. Spalten von `parameters`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `specific_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `specific_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `specific_name` | `sql_identifier` | Der »spezifische Name« der Funktion. Siehe [Abschnitt 35.45](35_Das_Information_Schema.md#3545-routines) für weitere Informationen. |
| `ordinal_position` | `cardinal_number` | Ordnungsposition des Parameters in der Argumentliste der Funktion; die Zählung beginnt bei 1 |
| `parameter_mode` | `character_data` | `IN` für Eingabeparameter, `OUT` für Ausgabeparameter und `INOUT` für Ein-/Ausgabeparameter |
| `is_result` | `yes_or_no` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `as_locator` | `yes_or_no` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `parameter_name` | `sql_identifier` | Name des Parameters oder null, wenn der Parameter keinen Namen hat |
| `data_type` | `character_data` | Datentyp des Parameters, wenn es ein eingebauter Typ ist; `ARRAY`, wenn es ein Array ist (siehe in diesem Fall die Sicht `element_types`); andernfalls `USER-DEFINED` (in diesem Fall wird der Typ in `udt_name` und den zugehörigen Spalten identifiziert) |
| `character_maximum_length` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Parameterdatentypen angewendet wird |
| `character_octet_length` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Parameterdatentypen angewendet wird |
| `character_set_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `collation_catalog` | `sql_identifier` | Immer null, da diese Information in PostgreSQL nicht auf Parameterdatentypen angewendet wird |
| `collation_schema` | `sql_identifier` | Immer null, da diese Information in PostgreSQL nicht auf Parameterdatentypen angewendet wird |
| `collation_name` | `sql_identifier` | Immer null, da diese Information in PostgreSQL nicht auf Parameterdatentypen angewendet wird |
| `numeric_precision` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Parameterdatentypen angewendet wird |
| `numeric_precision_radix` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Parameterdatentypen angewendet wird |
| `numeric_scale` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Parameterdatentypen angewendet wird |
| `datetime_precision` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Parameterdatentypen angewendet wird |
| `interval_type` | `character_data` | Immer null, da diese Information in PostgreSQL nicht auf Parameterdatentypen angewendet wird |
| `interval_precision` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Parameterdatentypen angewendet wird |
| `udt_catalog` | `sql_identifier` | Name der Datenbank, in der der Datentyp des Parameters definiert ist; immer die aktuelle Datenbank |
| `udt_schema` | `sql_identifier` | Name des Schemas, in dem der Datentyp des Parameters definiert ist |
| `udt_name` | `sql_identifier` | Name des Datentyps des Parameters |
| `scope_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `scope_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `scope_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `maximum_cardinality` | `cardinal_number` | Immer null, weil Arrays in PostgreSQL immer eine unbegrenzte maximale Kardinalität haben |
| `dtd_identifier` | `sql_identifier` | Ein Bezeichner des Datentypdeskriptors des Parameters, eindeutig unter den Datentypdeskriptoren, die zur Funktion gehören. Dies ist vor allem für Joins mit anderen Instanzen solcher Bezeichner nützlich. Das genaue Format des Bezeichners ist nicht definiert und es wird nicht garantiert, dass es in zukünftigen Versionen gleich bleibt |
| `parameter_default` | `character_data` | Default-Ausdruck des Parameters oder null, wenn keiner vorhanden ist oder wenn die Funktion nicht einer derzeit aktivierten Rolle gehört |

## 35.34. referential_constraints

Die Sicht `referential_constraints` enthält alle referenziellen Constraints (Fremdschlüssel-Constraints) in der aktuellen Datenbank. Angezeigt werden nur Constraints, für die der aktuelle Benutzer Schreibzugriff auf die referenzierende Tabelle hat, entweder weil er Eigentümer ist oder weil er ein anderes Privileg als `SELECT` besitzt.

**Tabelle 35.32. Spalten von `referential_constraints`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `constraint_catalog` | `sql_identifier` | Name der Datenbank, die das Constraint enthält; immer die aktuelle Datenbank |
| `constraint_schema` | `sql_identifier` | Name des Schemas, das das Constraint enthält |
| `constraint_name` | `sql_identifier` | Name des Constraints |
| `unique_constraint_catalog` | `sql_identifier` | Name der Datenbank, die das Unique- oder Primary-Key-Constraint enthält, auf das das Fremdschlüssel-Constraint verweist; immer die aktuelle Datenbank |
| `unique_constraint_schema` | `sql_identifier` | Name des Schemas, das das Unique- oder Primary-Key-Constraint enthält, auf das das Fremdschlüssel-Constraint verweist |
| `unique_constraint_name` | `sql_identifier` | Name des Unique- oder Primary-Key-Constraints, auf das das Fremdschlüssel-Constraint verweist |
| `match_option` | `character_data` | Match-Option des Fremdschlüssel-Constraints: `FULL`, `PARTIAL` oder `NONE` |
| `update_rule` | `character_data` | Aktualisierungsregel des Fremdschlüssel-Constraints: `CASCADE`, `SET NULL`, `SET DEFAULT`, `RESTRICT` oder `NO ACTION` |
| `delete_rule` | `character_data` | Löschregel des Fremdschlüssel-Constraints: `CASCADE`, `SET NULL`, `SET DEFAULT`, `RESTRICT` oder `NO ACTION` |

## 35.35. role_column_grants

Die Sicht `role_column_grants` identifiziert alle auf Spalten gewährten Privilegien, bei denen der Gewährer oder der Empfänger eine derzeit aktivierte Rolle ist. Weitere Informationen finden Sie unter `column_privileges`. Der einzige effektive Unterschied zwischen dieser Sicht und `column_privileges` besteht darin, dass diese Sicht Spalten auslässt, die dem aktuellen Benutzer durch eine Gewährung an `PUBLIC` zugänglich gemacht wurden.

**Tabelle 35.33. Spalten von `role_column_grants`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `grantor` | `sql_identifier` | Name der Rolle, die das Privileg gewährt hat |
| `grantee` | `sql_identifier` | Name der Rolle, der das Privileg gewährt wurde |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält, welche die Spalte enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält, welche die Spalte enthält |
| `table_name` | `sql_identifier` | Name der Tabelle, die die Spalte enthält |
| `column_name` | `sql_identifier` | Name der Spalte |
| `privilege_type` | `character_data` | Typ des Privilegs: `SELECT`, `INSERT`, `UPDATE` oder `REFERENCES` |
| `is_grantable` | `yes_or_no` | `YES`, wenn das Privileg weitergewährt werden kann, sonst `NO` |

## 35.36. role_routine_grants

Die Sicht `role_routine_grants` identifiziert alle auf Funktionen gewährten Privilegien, bei denen der Gewährer oder der Empfänger eine derzeit aktivierte Rolle ist. Weitere Informationen finden Sie unter `routine_privileges`. Der einzige effektive Unterschied zwischen dieser Sicht und `routine_privileges` besteht darin, dass diese Sicht Funktionen auslässt, die dem aktuellen Benutzer durch eine Gewährung an `PUBLIC` zugänglich gemacht wurden.

**Tabelle 35.34. Spalten von `role_routine_grants`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `grantor` | `sql_identifier` | Name der Rolle, die das Privileg gewährt hat |
| `grantee` | `sql_identifier` | Name der Rolle, der das Privileg gewährt wurde |
| `specific_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `specific_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `specific_name` | `sql_identifier` | Der »spezifische Name« der Funktion. Siehe [Abschnitt 35.45](35_Das_Information_Schema.md#3545-routines) für weitere Informationen. |
| `routine_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `routine_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `routine_name` | `sql_identifier` | Name der Funktion; kann bei Überladung mehrfach vorkommen |
| `privilege_type` | `character_data` | Immer `EXECUTE`, der einzige Privilegtyp für Funktionen |
| `is_grantable` | `yes_or_no` | `YES`, wenn das Privileg weitergewährt werden kann, sonst `NO` |

## 35.37. role_table_grants

Die Sicht `role_table_grants` identifiziert alle auf Tabellen oder Sichten gewährten Privilegien, bei denen der Gewährer oder der Empfänger eine derzeit aktivierte Rolle ist. Weitere Informationen finden Sie unter `table_privileges`. Der einzige effektive Unterschied zwischen dieser Sicht und `table_privileges` besteht darin, dass diese Sicht Tabellen auslässt, die dem aktuellen Benutzer durch eine Gewährung an `PUBLIC` zugänglich gemacht wurden.

**Tabelle 35.35. Spalten von `role_table_grants`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `grantor` | `sql_identifier` | Name der Rolle, die das Privileg gewährt hat |
| `grantee` | `sql_identifier` | Name der Rolle, der das Privileg gewährt wurde |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält |
| `table_name` | `sql_identifier` | Name der Tabelle |
| `privilege_type` | `character_data` | Typ des Privilegs: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES` oder `TRIGGER` |
| `is_grantable` | `yes_or_no` | `YES`, wenn das Privileg weitergewährt werden kann, sonst `NO` |
| `with_hierarchy` | `yes_or_no` | Im SQL-Standard ist `WITH HIERARCHY OPTION` ein eigenes Unterprivileg, das bestimmte Operationen auf Tabellenvererbungshierarchien erlaubt. In PostgreSQL ist dies im Privileg `SELECT` enthalten, daher zeigt diese Spalte `YES`, wenn das Privileg `SELECT` ist, sonst `NO` |

## 35.38. role_udt_grants

Die Sicht `role_udt_grants` soll `USAGE`-Privilegien auf benutzerdefinierten Typen identifizieren, bei denen der Gewährer oder der Empfänger eine derzeit aktivierte Rolle ist. Weitere Informationen finden Sie unter `udt_privileges`. Der einzige effektive Unterschied zwischen dieser Sicht und `udt_privileges` besteht darin, dass diese Sicht Objekte auslässt, die dem aktuellen Benutzer durch eine Gewährung an `PUBLIC` zugänglich gemacht wurden. Da Datentypen in PostgreSQL keine echten Privilegien haben, sondern nur eine implizite Gewährung an `PUBLIC`, ist diese Sicht leer.

**Tabelle 35.36. Spalten von `role_udt_grants`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `grantor` | `sql_identifier` | Name der Rolle, die das Privileg gewährt hat |
| `grantee` | `sql_identifier` | Name der Rolle, der das Privileg gewährt wurde |
| `udt_catalog` | `sql_identifier` | Name der Datenbank, die den Typ enthält; immer die aktuelle Datenbank |
| `udt_schema` | `sql_identifier` | Name des Schemas, das den Typ enthält |
| `udt_name` | `sql_identifier` | Name des Typs |
| `privilege_type` | `character_data` | Immer `TYPE USAGE` |
| `is_grantable` | `yes_or_no` | `YES`, wenn das Privileg weitergewährt werden kann, sonst `NO` |

## 35.39. role_usage_grants

Die Sicht `role_usage_grants` identifiziert `USAGE`-Privilegien auf verschiedenen Objektarten, bei denen der Gewährer oder der Empfänger eine derzeit aktivierte Rolle ist. Weitere Informationen finden Sie unter `usage_privileges`. Der einzige effektive Unterschied zwischen dieser Sicht und `usage_privileges` besteht darin, dass diese Sicht Objekte auslässt, die dem aktuellen Benutzer durch eine Gewährung an `PUBLIC` zugänglich gemacht wurden.

**Tabelle 35.37. Spalten von `role_usage_grants`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `grantor` | `sql_identifier` | Name der Rolle, die das Privileg gewährt hat |
| `grantee` | `sql_identifier` | Name der Rolle, der das Privileg gewährt wurde |
| `object_catalog` | `sql_identifier` | Name der Datenbank, die das Objekt enthält; immer die aktuelle Datenbank |
| `object_schema` | `sql_identifier` | Name des Schemas, das das Objekt enthält, falls zutreffend, sonst eine leere Zeichenkette |
| `object_name` | `sql_identifier` | Name des Objekts |
| `object_type` | `character_data` | `COLLATION`, `DOMAIN`, `FOREIGN DATA WRAPPER`, `FOREIGN SERVER` oder `SEQUENCE` |
| `privilege_type` | `character_data` | Immer `USAGE` |
| `is_grantable` | `yes_or_no` | `YES`, wenn das Privileg weitergewährt werden kann, sonst `NO` |

## 35.40. routine_column_usage

Die Sicht `routine_column_usage` identifiziert alle Spalten, die von einer Funktion oder Prozedur verwendet werden, entweder im SQL-Body oder in Parameter-Default-Ausdrücken. Dies funktioniert nur für nicht gequotete SQL-Bodys, nicht für gequotete Bodys oder Funktionen in anderen Sprachen. Eine Spalte ist nur enthalten, wenn ihre Tabelle einer derzeit aktivierten Rolle gehört.

**Tabelle 35.38. Spalten von `routine_column_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `specific_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `specific_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `specific_name` | `sql_identifier` | Der »spezifische Name« der Funktion. Siehe [Abschnitt 35.45](35_Das_Information_Schema.md#3545-routines) für weitere Informationen. |
| `routine_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `routine_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `routine_name` | `sql_identifier` | Name der Funktion; kann bei Überladung mehrfach vorkommen |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die von der Funktion verwendete Tabelle enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die von der Funktion verwendete Tabelle enthält |
| `table_name` | `sql_identifier` | Name der Tabelle, die von der Funktion verwendet wird |
| `column_name` | `sql_identifier` | Name der Spalte, die von der Funktion verwendet wird |

## 35.41. routine_privileges

Die Sicht `routine_privileges` identifiziert alle auf Funktionen gewährten Privilegien, die an eine derzeit aktivierte Rolle oder von einer derzeit aktivierten Rolle gewährt wurden. Für jede Kombination aus Funktion, Gewährer und Empfänger gibt es eine Zeile.

**Tabelle 35.39. Spalten von `routine_privileges`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `grantor` | `sql_identifier` | Name der Rolle, die das Privileg gewährt hat |
| `grantee` | `sql_identifier` | Name der Rolle, der das Privileg gewährt wurde |
| `specific_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `specific_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `specific_name` | `sql_identifier` | Der »spezifische Name« der Funktion. Siehe [Abschnitt 35.45](35_Das_Information_Schema.md#3545-routines) für weitere Informationen. |
| `routine_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `routine_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `routine_name` | `sql_identifier` | Name der Funktion; kann bei Überladung mehrfach vorkommen |
| `privilege_type` | `character_data` | Immer `EXECUTE`, der einzige Privilegtyp für Funktionen |
| `is_grantable` | `yes_or_no` | `YES`, wenn das Privileg weitergewährt werden kann, sonst `NO` |

## 35.42. routine_routine_usage

Die Sicht `routine_routine_usage` identifiziert alle Funktionen oder Prozeduren, die von einer anderen oder derselben Funktion oder Prozedur verwendet werden, entweder im SQL-Body oder in Parameter-Default-Ausdrücken. Dies funktioniert nur für nicht gequotete SQL-Bodys, nicht für gequotete Bodys oder Funktionen in anderen Sprachen. Ein Eintrag ist hier nur enthalten, wenn die verwendete Funktion einer derzeit aktivierten Rolle gehört; für die verwendende Funktion gibt es keine solche Einschränkung.

Beachten Sie, dass sich die Einträge für beide Funktionen in der Sicht auf den »spezifischen« Namen der Routine beziehen, auch wenn die Spaltennamen in einer Weise verwendet werden, die mit anderen Information-Schema-Sichten über Routinen inkonsistent ist. Dies entspricht dem SQL-Standard, ist aber wohl ein Fehlentwurf. Siehe [Abschnitt 35.45](35_Das_Information_Schema.md#3545-routines) für weitere Informationen über spezifische Namen.

**Tabelle 35.40. Spalten von `routine_routine_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `specific_catalog` | `sql_identifier` | Name der Datenbank, die die verwendende Funktion enthält; immer die aktuelle Datenbank |
| `specific_schema` | `sql_identifier` | Name des Schemas, das die verwendende Funktion enthält |
| `specific_name` | `sql_identifier` | Der »spezifische Name« der verwendenden Funktion |
| `routine_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält, die von der ersten Funktion verwendet wird; immer die aktuelle Datenbank |
| `routine_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält, die von der ersten Funktion verwendet wird |
| `routine_name` | `sql_identifier` | Der »spezifische Name« der Funktion, die von der ersten Funktion verwendet wird |

## 35.43. routine_sequence_usage

Die Sicht `routine_sequence_usage` identifiziert alle Sequenzen, die von einer Funktion oder Prozedur verwendet werden, entweder im SQL-Body oder in Parameter-Default-Ausdrücken. Dies funktioniert nur für nicht gequotete SQL-Bodys, nicht für gequotete Bodys oder Funktionen in anderen Sprachen. Eine Sequenz ist nur enthalten, wenn diese Sequenz einer derzeit aktivierten Rolle gehört.

**Tabelle 35.41. Spalten von `routine_sequence_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `specific_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `specific_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `specific_name` | `sql_identifier` | Der »spezifische Name« der Funktion. Siehe [Abschnitt 35.45](35_Das_Information_Schema.md#3545-routines) für weitere Informationen. |
| `routine_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `routine_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `routine_name` | `sql_identifier` | Name der Funktion; kann bei Überladung mehrfach vorkommen |
| `schema_catalog` | `sql_identifier` | Name der Datenbank, die die von der Funktion verwendete Sequenz enthält; immer die aktuelle Datenbank |
| `sequence_schema` | `sql_identifier` | Name des Schemas, das die von der Funktion verwendete Sequenz enthält |
| `sequence_name` | `sql_identifier` | Name der Sequenz, die von der Funktion verwendet wird |

## 35.44. routine_table_usage

Die Sicht `routine_table_usage` soll alle Tabellen identifizieren, die von einer Funktion oder Prozedur verwendet werden. Diese Information wird derzeit von PostgreSQL nicht verfolgt.

**Tabelle 35.42. Spalten von `routine_table_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `specific_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `specific_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `specific_name` | `sql_identifier` | Der »spezifische Name« der Funktion. Siehe [Abschnitt 35.45](35_Das_Information_Schema.md#3545-routines) für weitere Informationen. |
| `routine_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `routine_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `routine_name` | `sql_identifier` | Name der Funktion; kann bei Überladung mehrfach vorkommen |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die von der Funktion verwendete Tabelle enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die von der Funktion verwendete Tabelle enthält |
| `table_name` | `sql_identifier` | Name der Tabelle, die von der Funktion verwendet wird |

## 35.45. routines

Die Sicht `routines` enthält alle Funktionen und Prozeduren in der aktuellen Datenbank. Angezeigt werden nur Funktionen und Prozeduren, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.43. Spalten von `routines`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `specific_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `specific_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `specific_name` | `sql_identifier` | Der »spezifische Name« der Funktion. Dies ist ein Name, der die Funktion im Schema eindeutig identifiziert, auch wenn der echte Name der Funktion überladen ist. Das Format des spezifischen Namens ist nicht definiert; er sollte nur verwendet werden, um ihn mit anderen Instanzen spezifischer Routinenamen zu vergleichen |
| `routine_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `routine_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `routine_name` | `sql_identifier` | Name der Funktion; kann bei Überladung mehrfach vorkommen |
| `routine_type` | `character_data` | `FUNCTION` für eine Funktion, `PROCEDURE` für eine Prozedur |
| `module_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `module_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `module_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `udt_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `udt_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `udt_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `data_type` | `character_data` | Rückgabedatentyp der Funktion, wenn es ein eingebauter Typ ist; `ARRAY`, wenn es ein Array ist (siehe in diesem Fall die Sicht `element_types`); andernfalls `USER-DEFINED` (in diesem Fall wird der Typ in `type_udt_name` und den zugehörigen Spalten identifiziert). Null für eine Prozedur |
| `character_maximum_length` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Rückgabedatentypen angewendet wird |
| `character_octet_length` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Rückgabedatentypen angewendet wird |
| `character_set_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `collation_catalog` | `sql_identifier` | Immer null, da diese Information in PostgreSQL nicht auf Rückgabedatentypen angewendet wird |
| `collation_schema` | `sql_identifier` | Immer null, da diese Information in PostgreSQL nicht auf Rückgabedatentypen angewendet wird |
| `collation_name` | `sql_identifier` | Immer null, da diese Information in PostgreSQL nicht auf Rückgabedatentypen angewendet wird |
| `numeric_precision` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Rückgabedatentypen angewendet wird |
| `numeric_precision_radix` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Rückgabedatentypen angewendet wird |
| `numeric_scale` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Rückgabedatentypen angewendet wird |
| `datetime_precision` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Rückgabedatentypen angewendet wird |
| `interval_type` | `character_data` | Immer null, da diese Information in PostgreSQL nicht auf Rückgabedatentypen angewendet wird |
| `interval_precision` | `cardinal_number` | Immer null, da diese Information in PostgreSQL nicht auf Rückgabedatentypen angewendet wird |
| `type_udt_catalog` | `sql_identifier` | Name der Datenbank, in der der Rückgabedatentyp der Funktion definiert ist; immer die aktuelle Datenbank. Null für eine Prozedur |
| `type_udt_schema` | `sql_identifier` | Name des Schemas, in dem der Rückgabedatentyp der Funktion definiert ist. Null für eine Prozedur |
| `type_udt_name` | `sql_identifier` | Name des Rückgabedatentyps der Funktion. Null für eine Prozedur |
| `scope_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `scope_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `scope_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `maximum_cardinality` | `cardinal_number` | Immer null, weil Arrays in PostgreSQL immer eine unbegrenzte maximale Kardinalität haben |
| `dtd_identifier` | `sql_identifier` | Ein Bezeichner des Datentypdeskriptors des Rückgabedatentyps dieser Funktion, eindeutig unter den Datentypdeskriptoren, die zur Funktion gehören. Dies ist vor allem für Joins mit anderen Instanzen solcher Bezeichner nützlich. Das genaue Format des Bezeichners ist nicht definiert und es wird nicht garantiert, dass es in zukünftigen Versionen gleich bleibt |
| `routine_body` | `character_data` | Wenn die Funktion eine SQL-Funktion ist, dann `SQL`, sonst `EXTERNAL` |
| `routine_definition` | `character_data` | Der Quelltext der Funktion; null, wenn die Funktion nicht einer derzeit aktivierten Rolle gehört. Nach dem SQL-Standard ist diese Spalte nur anwendbar, wenn `routine_body` `SQL` ist, in PostgreSQL enthält sie jedoch den Quelltext, der beim Erstellen der Funktion angegeben wurde |
| `external_name` | `character_data` | Wenn diese Funktion eine C-Funktion ist, der externe Name (Link-Symbol) der Funktion, sonst null. Dies entspricht demselben Wert, der in `routine_definition` angezeigt wird |
| `external_language` | `character_data` | Die Sprache, in der die Funktion geschrieben ist |
| `parameter_style` | `character_data` | Immer `GENERAL`; der SQL-Standard definiert andere Parameterstile, die in PostgreSQL nicht verfügbar sind |
| `is_deterministic` | `yes_or_no` | Wenn die Funktion als immutable deklariert ist (im SQL-Standard deterministic genannt), dann `YES`, sonst `NO`. Die anderen in PostgreSQL verfügbaren Volatilitätsstufen können nicht über das Information Schema abgefragt werden |
| `sql_data_access` | `character_data` | Immer `MODIFIES`, was bedeutet, dass die Funktion möglicherweise SQL-Daten verändert. Diese Information ist für PostgreSQL nicht nützlich |
| `is_null_call` | `yes_or_no` | Wenn die Funktion automatisch null zurückgibt, sobald eines ihrer Argumente null ist, dann `YES`, sonst `NO`. Null für eine Prozedur |
| `sql_path` | `character_data` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `schema_level_routine` | `yes_or_no` | Immer `YES`; das Gegenteil wäre eine Methode eines benutzerdefinierten Typs, eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `max_dynamic_result_sets` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `is_user_defined_cast` | `yes_or_no` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `is_implicitly_invocable` | `yes_or_no` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `security_type` | `character_data` | Wenn die Funktion mit den Privilegien des aktuellen Benutzers läuft, dann `INVOKER`; wenn sie mit den Privilegien des Benutzers läuft, der sie definiert hat, dann `DEFINER` |
| `to_sql_specific_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `to_sql_specific_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `to_sql_specific_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `as_locator` | `yes_or_no` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `created` | `time_stamp` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `last_altered` | `time_stamp` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `new_savepoint_level` | `yes_or_no` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `is_udt_dependent` | `yes_or_no` | Derzeit immer `NO`. Die Alternative `YES` bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_from_data_type` | `character_data` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_as_locator` | `yes_or_no` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_char_max_length` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_char_octet_length` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_char_set_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_char_set_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_char_set_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_collation_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_collation_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_collation_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_numeric_precision` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_numeric_precision_radix` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_numeric_scale` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_datetime_precision` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_interval_type` | `character_data` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_interval_precision` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_type_udt_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_type_udt_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_type_udt_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_scope_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_scope_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_scope_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_maximum_cardinality` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `result_cast_dtd_identifier` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |

## 35.46. schemata

Die Sicht `schemata` enthält alle Schemas in der aktuellen Datenbank, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.44. Spalten von `schemata`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `catalog_name` | `sql_identifier` | Name der Datenbank, in der das Schema enthalten ist; immer die aktuelle Datenbank |
| `schema_name` | `sql_identifier` | Name des Schemas |
| `schema_owner` | `sql_identifier` | Name des Eigentümers des Schemas |
| `default_character_set_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `default_character_set_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `default_character_set_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `sql_path` | `character_data` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |

## 35.47. sequences

Die Sicht `sequences` enthält alle Sequenzen, die in der aktuellen Datenbank definiert sind. Angezeigt werden nur Sequenzen, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.45. Spalten von `sequences`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `sequence_catalog` | `sql_identifier` | Name der Datenbank, die die Sequenz enthält; immer die aktuelle Datenbank |
| `sequence_schema` | `sql_identifier` | Name des Schemas, das die Sequenz enthält |
| `sequence_name` | `sql_identifier` | Name der Sequenz |
| `data_type` | `character_data` | Datentyp der Sequenz |
| `numeric_precision` | `cardinal_number` | Diese Spalte enthält die deklarierte oder implizite Genauigkeit des Sequenzdatentyps (siehe oben). Die Genauigkeit gibt die Anzahl signifikanter Ziffern an. Sie kann dezimal (Basis 10) oder binär (Basis 2) ausgedrückt werden, wie in der Spalte `numeric_precision_radix` angegeben |
| `numeric_precision_radix` | `cardinal_number` | Diese Spalte gibt an, in welcher Basis die Werte in den Spalten `numeric_precision` und `numeric_scale` ausgedrückt werden. Der Wert ist entweder 2 oder 10 |
| `numeric_scale` | `cardinal_number` | Diese Spalte enthält die deklarierte oder implizite Skalierung des Sequenzdatentyps (siehe oben). Die Skalierung gibt die Anzahl signifikanter Ziffern rechts vom Dezimalpunkt an. Sie kann dezimal (Basis 10) oder binär (Basis 2) ausgedrückt werden, wie in der Spalte `numeric_precision_radix` angegeben |
| `start_value` | `character_data` | Startwert der Sequenz |
| `minimum_value` | `character_data` | Minimalwert der Sequenz |
| `maximum_value` | `character_data` | Maximalwert der Sequenz |
| `increment` | `character_data` | Schrittweite der Sequenz |
| `cycle_option` | `yes_or_no` | `YES`, wenn die Sequenz umläuft, sonst `NO` |

Beachten Sie, dass die Start-, Minimal-, Maximal- und Schrittweitenwerte gemäß SQL-Standard als Zeichenketten zurückgegeben werden.

## 35.48. sql_features

Die Tabelle `sql_features` enthält Informationen darüber, welche im SQL-Standard definierten formalen Funktionen von PostgreSQL unterstützt werden. Dies sind dieselben Informationen, die in Anhang D dargestellt werden. Dort finden Sie auch zusätzliche Hintergrundinformationen.

**Tabelle 35.46. Spalten von `sql_features`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `feature_id` | `character_data` | Bezeichnerzeichenkette der Funktion |
| `feature_name` | `character_data` | Beschreibender Name der Funktion |
| `sub_feature_id` | `character_data` | Bezeichnerzeichenkette der Unterfunktion oder eine Zeichenkette der Länge null, wenn es keine Unterfunktion ist |
| `sub_feature_name` | `character_data` | Beschreibender Name der Unterfunktion oder eine Zeichenkette der Länge null, wenn es keine Unterfunktion ist |
| `is_supported` | `yes_or_no` | `YES`, wenn die Funktion von der aktuellen PostgreSQL-Version vollständig unterstützt wird, sonst `NO` |
| `is_verified_by` | `character_data` | Immer null, da die PostgreSQL-Entwicklergruppe keine formalen Tests der Funktionskonformität durchführt |
| `comments` | `character_data` | Möglicherweise ein Kommentar zum Unterstützungsstatus der Funktion |

## 35.49. sql_implementation_info

Die Tabelle `sql_implementation_info` enthält Informationen über verschiedene Aspekte, die vom SQL-Standard als implementierungsdefiniert offen gelassen werden. Diese Informationen sind in erster Linie für die Verwendung im Zusammenhang mit der ODBC-Schnittstelle gedacht; Benutzer anderer Schnittstellen werden sie vermutlich nur wenig nützlich finden. Aus diesem Grund werden die einzelnen Implementierungsinformationspunkte hier nicht beschrieben; Sie finden sie in der Beschreibung der ODBC-Schnittstelle.

**Tabelle 35.47. Spalten von `sql_implementation_info`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `implementation_info_id` | `character_data` | Bezeichnerzeichenkette des Implementierungsinformationspunkts |
| `implementation_info_name` | `character_data` | Beschreibender Name des Implementierungsinformationspunkts |
| `integer_value` | `cardinal_number` | Wert des Implementierungsinformationspunkts oder null, wenn der Wert in der Spalte `character_value` enthalten ist |
| `character_value` | `character_data` | Wert des Implementierungsinformationspunkts oder null, wenn der Wert in der Spalte `integer_value` enthalten ist |
| `comments` | `character_data` | Möglicherweise ein Kommentar zum Implementierungsinformationspunkt |

## 35.50. sql_parts

Die Tabelle `sql_parts` enthält Informationen darüber, welche der verschiedenen Teile des SQL-Standards von PostgreSQL unterstützt werden.

**Tabelle 35.48. Spalten von `sql_parts`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `feature_id` | `character_data` | Eine Bezeichnerzeichenkette, die die Nummer des Teils enthält |
| `feature_name` | `character_data` | Beschreibender Name des Teils |
| `is_supported` | `yes_or_no` | `YES`, wenn der Teil von der aktuellen PostgreSQL-Version vollständig unterstützt wird, sonst `NO` |
| `is_verified_by` | `character_data` | Immer null, da die PostgreSQL-Entwicklergruppe keine formalen Tests der Funktionskonformität durchführt |
| `comments` | `character_data` | Möglicherweise ein Kommentar zum Unterstützungsstatus des Teils |

## 35.51. sql_sizing

Die Tabelle `sql_sizing` enthält Informationen über verschiedene Größenbeschränkungen und Maximalwerte in PostgreSQL. Diese Informationen sind in erster Linie für die Verwendung im Zusammenhang mit der ODBC-Schnittstelle gedacht; Benutzer anderer Schnittstellen werden sie vermutlich nur wenig nützlich finden. Aus diesem Grund werden die einzelnen Größenangaben hier nicht beschrieben; Sie finden sie in der Beschreibung der ODBC-Schnittstelle.

**Tabelle 35.49. Spalten von `sql_sizing`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `sizing_id` | `cardinal_number` | Bezeichner des Größenpunkts |
| `sizing_name` | `character_data` | Beschreibender Name des Größenpunkts |
| `supported_value` | `cardinal_number` | Wert des Größenpunkts oder 0, wenn die Größe unbegrenzt ist oder nicht bestimmt werden kann, oder null, wenn die Funktionen, für die der Größenpunkt anwendbar ist, nicht unterstützt werden |
| `comments` | `character_data` | Möglicherweise ein Kommentar zum Größenpunkt |

## 35.52. table_constraints

Die Sicht `table_constraints` enthält alle Constraints, die zu Tabellen gehören, deren Eigentümer der aktuelle Benutzer ist oder auf die er ein anderes Privileg als `SELECT` besitzt.

**Tabelle 35.50. Spalten von `table_constraints`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `constraint_catalog` | `sql_identifier` | Name der Datenbank, die das Constraint enthält; immer die aktuelle Datenbank |
| `constraint_schema` | `sql_identifier` | Name des Schemas, das das Constraint enthält |
| `constraint_name` | `sql_identifier` | Name des Constraints |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält |
| `table_name` | `sql_identifier` | Name der Tabelle |
| `constraint_type` | `character_data` | Typ des Constraints: `CHECK` (einschließlich Not-Null-Constraints), `FOREIGN KEY`, `PRIMARY KEY` oder `UNIQUE` |
| `is_deferrable` | `yes_or_no` | `YES`, wenn das Constraint aufschiebbar ist, sonst `NO` |
| `initially_deferred` | `yes_or_no` | `YES`, wenn das Constraint aufschiebbar und anfänglich aufgeschoben ist, sonst `NO` |
| `enforced` | `yes_or_no` | `YES`, wenn das Constraint erzwungen wird, sonst `NO` |
| `nulls_distinct` | `yes_or_no` | Wenn das Constraint ein Unique-Constraint ist, dann `YES`, wenn das Constraint Nullwerte als verschieden behandelt, oder `NO`, wenn es Nullwerte als nicht verschieden behandelt; andernfalls null für andere Constraint-Typen |

## 35.53. table_privileges

Die Sicht `table_privileges` identifiziert alle auf Tabellen oder Sichten gewährten Privilegien, die an eine derzeit aktivierte Rolle oder von einer derzeit aktivierten Rolle gewährt wurden. Für jede Kombination aus Tabelle, Gewährer und Empfänger gibt es eine Zeile.

**Tabelle 35.51. Spalten von `table_privileges`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `grantor` | `sql_identifier` | Name der Rolle, die das Privileg gewährt hat |
| `grantee` | `sql_identifier` | Name der Rolle, der das Privileg gewährt wurde |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält |
| `table_name` | `sql_identifier` | Name der Tabelle |
| `privilege_type` | `character_data` | Typ des Privilegs: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES` oder `TRIGGER` |
| `is_grantable` | `yes_or_no` | `YES`, wenn das Privileg weitergewährt werden kann, sonst `NO` |
| `with_hierarchy` | `yes_or_no` | Im SQL-Standard ist `WITH HIERARCHY OPTION` ein eigenes Unterprivileg, das bestimmte Operationen auf Tabellenvererbungshierarchien erlaubt. In PostgreSQL ist dies im Privileg `SELECT` enthalten, daher zeigt diese Spalte `YES`, wenn das Privileg `SELECT` ist, sonst `NO` |

## 35.54. tables

Die Sicht `tables` enthält alle Tabellen und Sichten, die in der aktuellen Datenbank definiert sind. Angezeigt werden nur Tabellen und Sichten, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.52. Spalten von `tables`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält |
| `table_name` | `sql_identifier` | Name der Tabelle |
| `table_type` | `character_data` | Typ der Tabelle: `BASE TABLE` für eine persistente Basistabelle (der normale Tabellentyp), `VIEW` für eine Sicht, `FOREIGN` für eine Fremdtabelle oder `LOCAL TEMPORARY` für eine temporäre Tabelle |
| `self_referencing_column_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `reference_generation` | `character_data` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `user_defined_type_catalog` | `sql_identifier` | Wenn die Tabelle eine typisierte Tabelle ist, der Name der Datenbank, die den zugrunde liegenden Datentyp enthält; immer die aktuelle Datenbank. Andernfalls null |
| `user_defined_type_schema` | `sql_identifier` | Wenn die Tabelle eine typisierte Tabelle ist, der Name des Schemas, das den zugrunde liegenden Datentyp enthält. Andernfalls null |
| `user_defined_type_name` | `sql_identifier` | Wenn die Tabelle eine typisierte Tabelle ist, der Name des zugrunde liegenden Datentyps. Andernfalls null |
| `is_insertable_into` | `yes_or_no` | `YES`, wenn in die Tabelle eingefügt werden kann, sonst `NO`; in Basistabellen kann immer eingefügt werden, in Sichten nicht unbedingt |
| `is_typed` | `yes_or_no` | `YES`, wenn die Tabelle eine typisierte Tabelle ist, sonst `NO` |
| `commit_action` | `character_data` | Noch nicht implementiert |

## 35.55. transforms

Die Sicht `transforms` enthält Informationen über die in der aktuellen Datenbank definierten Transforms. Genauer gesagt enthält sie eine Zeile für jede Funktion, die in einem Transform enthalten ist, also die »from SQL«- oder »to SQL«-Funktion.

**Tabelle 35.53. Spalten von `transforms`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `udt_catalog` | `sql_identifier` | Name der Datenbank, die den Typ enthält, für den der Transform gilt; immer die aktuelle Datenbank |
| `udt_schema` | `sql_identifier` | Name des Schemas, das den Typ enthält, für den der Transform gilt |
| `udt_name` | `sql_identifier` | Name des Typs, für den der Transform gilt |
| `specific_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `specific_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `specific_name` | `sql_identifier` | Der »spezifische Name« der Funktion. Siehe [Abschnitt 35.45](35_Das_Information_Schema.md#3545-routines) für weitere Informationen. |
| `group_name` | `sql_identifier` | Der SQL-Standard erlaubt, Transforms in »Gruppen« zu definieren und zur Laufzeit eine Gruppe auszuwählen. PostgreSQL unterstützt dies nicht. Stattdessen sind Transforms sprachspezifisch. Als Kompromiss enthält dieses Feld die Sprache, für die der Transform gilt |
| `transform_type` | `character_data` | `FROM SQL` oder `TO SQL` |

## 35.56. triggered_update_columns

Für Trigger in der aktuellen Datenbank, die eine Spaltenliste angeben (zum Beispiel `UPDATE OF column1, column2`), identifiziert die Sicht `triggered_update_columns` diese Spalten. Trigger, die keine Spaltenliste angeben, sind in dieser Sicht nicht enthalten. Angezeigt werden nur Spalten, deren Eigentümer der aktuelle Benutzer ist oder auf die er ein anderes Privileg als `SELECT` besitzt.

**Tabelle 35.54. Spalten von `triggered_update_columns`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `trigger_catalog` | `sql_identifier` | Name der Datenbank, die den Trigger enthält; immer die aktuelle Datenbank |
| `trigger_schema` | `sql_identifier` | Name des Schemas, das den Trigger enthält |
| `trigger_name` | `sql_identifier` | Name des Triggers |
| `event_object_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält, auf der der Trigger definiert ist; immer die aktuelle Datenbank |
| `event_object_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält, auf der der Trigger definiert ist |
| `event_object_table` | `sql_identifier` | Name der Tabelle, auf der der Trigger definiert ist |
| `event_object_column` | `sql_identifier` | Name der Spalte, auf der der Trigger definiert ist |

## 35.57. triggers

Die Sicht `triggers` enthält alle Trigger, die in der aktuellen Datenbank auf Tabellen und Sichten definiert sind, deren Eigentümer der aktuelle Benutzer ist oder auf die er ein anderes Privileg als `SELECT` besitzt.

**Tabelle 35.55. Spalten von `triggers`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `trigger_catalog` | `sql_identifier` | Name der Datenbank, die den Trigger enthält; immer die aktuelle Datenbank |
| `trigger_schema` | `sql_identifier` | Name des Schemas, das den Trigger enthält |
| `trigger_name` | `sql_identifier` | Name des Triggers |
| `event_manipulation` | `character_data` | Ereignis, das den Trigger auslöst: `INSERT`, `UPDATE` oder `DELETE` |
| `event_object_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält, auf der der Trigger definiert ist; immer die aktuelle Datenbank |
| `event_object_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält, auf der der Trigger definiert ist |
| `event_object_table` | `sql_identifier` | Name der Tabelle, auf der der Trigger definiert ist |
| `action_order` | `cardinal_number` | Ausführungsreihenfolge unter Triggern auf derselben Tabelle mit demselben `event_manipulation`, `action_timing` und `action_orientation`. In PostgreSQL werden Trigger in Namensreihenfolge ausgelöst, daher spiegelt diese Spalte dies wider |
| `action_condition` | `character_data` | `WHEN`-Bedingung des Triggers, null wenn keine vorhanden ist; ebenfalls null, wenn die Tabelle nicht einer derzeit aktivierten Rolle gehört |
| `action_statement` | `character_data` | Anweisung, die vom Trigger ausgeführt wird; derzeit immer `EXECUTE FUNCTION function(...)` |
| `action_orientation` | `character_data` | Gibt an, ob der Trigger einmal pro verarbeiteter Zeile oder einmal pro Anweisung ausgelöst wird: `ROW` oder `STATEMENT` |
| `action_timing` | `character_data` | Zeitpunkt, zu dem der Trigger ausgelöst wird: `BEFORE`, `AFTER` oder `INSTEAD OF` |
| `action_reference_old_table` | `sql_identifier` | Name der »old«-Transition-Tabelle oder null, wenn keine vorhanden ist |
| `action_reference_new_table` | `sql_identifier` | Name der »new«-Transition-Tabelle oder null, wenn keine vorhanden ist |
| `action_reference_old_row` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `action_reference_new_row` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `created` | `time_stamp` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |

Trigger in PostgreSQL haben zwei Inkompatibilitäten mit dem SQL-Standard, die die Darstellung im Information Schema betreffen. Erstens sind Triggernamen in PostgreSQL lokal zu jeder Tabelle, statt unabhängige Schemaobjekte zu sein. Daher können in einem Schema doppelte Triggernamen definiert sein, solange sie zu verschiedenen Tabellen gehören. `trigger_catalog` und `trigger_schema` sind tatsächlich die Werte, die zur Tabelle gehören, auf der der Trigger definiert ist. Zweitens können Trigger in PostgreSQL so definiert werden, dass sie bei mehreren Ereignissen ausgelöst werden, zum Beispiel `ON INSERT OR UPDATE`, während der SQL-Standard nur eines erlaubt. Wenn ein Trigger für mehrere Ereignisse definiert ist, wird er im Information Schema als mehrere Zeilen dargestellt, eine für jeden Ereignistyp. Als Folge dieser beiden Punkte ist der Primärschlüssel der Sicht `triggers` tatsächlich (`trigger_catalog`, `trigger_schema`, `event_object_table`, `trigger_name`, `event_manipulation`) statt (`trigger_catalog`, `trigger_schema`, `trigger_name`), wie es der SQL-Standard festlegt. Wenn Sie Ihre Trigger jedoch standardkonform definieren, also mit im Schema eindeutigen Triggernamen und nur einem Ereignistyp pro Trigger, betrifft Sie dies nicht.

> **Hinweis:** Vor PostgreSQL 9.1 hießen die Spalten `action_timing`, `action_reference_old_table`, `action_reference_new_table`, `action_reference_old_row` und `action_reference_new_row` dieser Sicht `condition_timing`, `condition_reference_old_table`, `condition_reference_new_table`, `condition_reference_old_row` beziehungsweise `condition_reference_new_row`. So waren sie im SQL:1999-Standard benannt. Die neue Benennung entspricht SQL:2003 und später.

## 35.58. udt_privileges

Die Sicht `udt_privileges` identifiziert `USAGE`-Privilegien, die auf benutzerdefinierten Typen an eine derzeit aktivierte Rolle oder von einer derzeit aktivierten Rolle gewährt wurden. Für jede Kombination aus Typ, Gewährer und Empfänger gibt es eine Zeile. Diese Sicht zeigt nur zusammengesetzte Typen (siehe [Abschnitt 35.60](35_Das_Information_Schema.md#3560-userdefinedtypes) für den Grund); für Domain-Privilegien siehe [Abschnitt 35.59](35_Das_Information_Schema.md#3559-usageprivileges).

**Tabelle 35.56. Spalten von `udt_privileges`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `grantor` | `sql_identifier` | Name der Rolle, die das Privileg gewährt hat |
| `grantee` | `sql_identifier` | Name der Rolle, der das Privileg gewährt wurde |
| `udt_catalog` | `sql_identifier` | Name der Datenbank, die den Typ enthält; immer die aktuelle Datenbank |
| `udt_schema` | `sql_identifier` | Name des Schemas, das den Typ enthält |
| `udt_name` | `sql_identifier` | Name des Typs |
| `privilege_type` | `character_data` | Immer `TYPE USAGE` |
| `is_grantable` | `yes_or_no` | `YES`, wenn das Privileg weitergewährt werden kann, sonst `NO` |

## 35.59. usage_privileges

Die Sicht `usage_privileges` identifiziert `USAGE`-Privilegien, die auf verschiedenen Objektarten an eine derzeit aktivierte Rolle oder von einer derzeit aktivierten Rolle gewährt wurden. In PostgreSQL betrifft dies derzeit Sortierfolgen, Domains, Foreign-Data Wrapper, Foreign Servers und Sequenzen. Für jede Kombination aus Objekt, Gewährer und Empfänger gibt es eine Zeile.

Da Sortierfolgen in PostgreSQL keine echten Privilegien haben, zeigt diese Sicht implizite, nicht weitergewährbare `USAGE`-Privilegien an, die vom Eigentümer an `PUBLIC` gewährt wurden, und zwar für alle Sortierfolgen. Die anderen Objekttypen zeigen dagegen echte Privilegien.

In PostgreSQL unterstützen Sequenzen zusätzlich zum `USAGE`-Privileg auch `SELECT`- und `UPDATE`-Privilegien. Diese sind nicht standardkonform und daher im Information Schema nicht sichtbar.

**Tabelle 35.57. Spalten von `usage_privileges`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `grantor` | `sql_identifier` | Name der Rolle, die das Privileg gewährt hat |
| `grantee` | `sql_identifier` | Name der Rolle, der das Privileg gewährt wurde |
| `object_catalog` | `sql_identifier` | Name der Datenbank, die das Objekt enthält; immer die aktuelle Datenbank |
| `object_schema` | `sql_identifier` | Name des Schemas, das das Objekt enthält, falls zutreffend, sonst eine leere Zeichenkette |
| `object_name` | `sql_identifier` | Name des Objekts |
| `object_type` | `character_data` | `COLLATION`, `DOMAIN`, `FOREIGN DATA WRAPPER`, `FOREIGN SERVER` oder `SEQUENCE` |
| `privilege_type` | `character_data` | Immer `USAGE` |
| `is_grantable` | `yes_or_no` | `YES`, wenn das Privileg weitergewährt werden kann, sonst `NO` |

## 35.60. user_defined_types

Die Sicht `user_defined_types` enthält derzeit alle zusammengesetzten Typen, die in der aktuellen Datenbank definiert sind. Angezeigt werden nur Typen, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

SQL kennt zwei Arten benutzerdefinierter Typen: strukturierte Typen (in PostgreSQL auch zusammengesetzte Typen genannt) und Distinct-Typen (in PostgreSQL nicht implementiert). Verwenden Sie zukunftssicher die Spalte `user_defined_type_category`, um zwischen diesen zu unterscheiden. Andere benutzerdefinierte Typen wie Basistypen und Enums, die PostgreSQL-Erweiterungen sind, werden hier nicht angezeigt. Für Domains siehe stattdessen [Abschnitt 35.23](35_Das_Information_Schema.md#3523-domains).

**Tabelle 35.58. Spalten von `user_defined_types`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `user_defined_type_catalog` | `sql_identifier` | Name der Datenbank, die den Typ enthält; immer die aktuelle Datenbank |
| `user_defined_type_schema` | `sql_identifier` | Name des Schemas, das den Typ enthält |
| `user_defined_type_name` | `sql_identifier` | Name des Typs |
| `user_defined_type_category` | `character_data` | Derzeit immer `STRUCTURED` |
| `is_instantiable` | `yes_or_no` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `is_final` | `yes_or_no` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `ordering_form` | `character_data` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `ordering_category` | `character_data` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `ordering_routine_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `ordering_routine_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `ordering_routine_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `reference_type` | `character_data` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `data_type` | `character_data` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_maximum_length` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_octet_length` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `character_set_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `collation_catalog` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `collation_schema` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `collation_name` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `numeric_precision` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `numeric_precision_radix` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `numeric_scale` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `datetime_precision` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `interval_type` | `character_data` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `interval_precision` | `cardinal_number` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `source_dtd_identifier` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |
| `ref_dtd_identifier` | `sql_identifier` | Bezieht sich auf eine Funktion, die in PostgreSQL nicht verfügbar ist |

## 35.61. user_mapping_options

Die Sicht `user_mapping_options` enthält alle Optionen, die für User Mappings in der aktuellen Datenbank definiert sind. Angezeigt werden nur solche User Mappings, bei denen der aktuelle Benutzer Zugriff auf den zugehörigen Foreign Server hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.59. Spalten von `user_mapping_options`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `authorization_identifier` | `sql_identifier` | Name des Benutzers, der gemappt wird, oder `PUBLIC`, wenn das Mapping öffentlich ist |
| `foreign_server_catalog` | `sql_identifier` | Name der Datenbank, in der der von diesem Mapping verwendete Foreign Server definiert ist; immer die aktuelle Datenbank |
| `foreign_server_name` | `sql_identifier` | Name des Foreign Servers, der von diesem Mapping verwendet wird |
| `option_name` | `sql_identifier` | Name einer Option |
| `option_value` | `character_data` | Wert der Option. Diese Spalte wird als null angezeigt, außer wenn der aktuelle Benutzer der gemappte Benutzer ist, das Mapping für `PUBLIC` gilt und der aktuelle Benutzer der Servereigentümer ist, oder der aktuelle Benutzer ein Superuser ist. Ziel ist, Passwortinformationen zu schützen, die als User-Mapping-Option gespeichert sind |

## 35.62. user_mappings

Die Sicht `user_mappings` enthält alle User Mappings, die in der aktuellen Datenbank definiert sind. Angezeigt werden nur solche User Mappings, bei denen der aktuelle Benutzer Zugriff auf den zugehörigen Foreign Server hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.60. Spalten von `user_mappings`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `authorization_identifier` | `sql_identifier` | Name des Benutzers, der gemappt wird, oder `PUBLIC`, wenn das Mapping öffentlich ist |
| `foreign_server_catalog` | `sql_identifier` | Name der Datenbank, in der der von diesem Mapping verwendete Foreign Server definiert ist; immer die aktuelle Datenbank |
| `foreign_server_name` | `sql_identifier` | Name des Foreign Servers, der von diesem Mapping verwendet wird |

## 35.63. view_column_usage

Die Sicht `view_column_usage` identifiziert alle Spalten, die im Abfrageausdruck einer Sicht verwendet werden, also in der `SELECT`-Anweisung, die die Sicht definiert. Eine Spalte ist nur enthalten, wenn die Tabelle, die die Spalte enthält, einer derzeit aktivierten Rolle gehört.

> **Hinweis:** Spalten von Systemtabellen sind nicht enthalten. Dies sollte irgendwann behoben werden.

**Tabelle 35.61. Spalten von `view_column_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `view_catalog` | `sql_identifier` | Name der Datenbank, die die Sicht enthält; immer die aktuelle Datenbank |
| `view_schema` | `sql_identifier` | Name des Schemas, das die Sicht enthält |
| `view_name` | `sql_identifier` | Name der Sicht |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Tabelle enthält, welche die von der Sicht verwendete Spalte enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Tabelle enthält, welche die von der Sicht verwendete Spalte enthält |
| `table_name` | `sql_identifier` | Name der Tabelle, die die von der Sicht verwendete Spalte enthält |
| `column_name` | `sql_identifier` | Name der Spalte, die von der Sicht verwendet wird |

## 35.64. view_routine_usage

Die Sicht `view_routine_usage` identifiziert alle Routinen (Funktionen und Prozeduren), die im Abfrageausdruck einer Sicht verwendet werden, also in der `SELECT`-Anweisung, die die Sicht definiert. Eine Routine ist nur enthalten, wenn diese Routine einer derzeit aktivierten Rolle gehört.

**Tabelle 35.62. Spalten von `view_routine_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Sicht enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Sicht enthält |
| `table_name` | `sql_identifier` | Name der Sicht |
| `specific_catalog` | `sql_identifier` | Name der Datenbank, die die Funktion enthält; immer die aktuelle Datenbank |
| `specific_schema` | `sql_identifier` | Name des Schemas, das die Funktion enthält |
| `specific_name` | `sql_identifier` | Der »spezifische Name« der Funktion. Siehe [Abschnitt 35.45](35_Das_Information_Schema.md#3545-routines) für weitere Informationen. |

## 35.65. view_table_usage

Die Sicht `view_table_usage` identifiziert alle Tabellen, die im Abfrageausdruck einer Sicht verwendet werden, also in der `SELECT`-Anweisung, die die Sicht definiert. Eine Tabelle ist nur enthalten, wenn diese Tabelle einer derzeit aktivierten Rolle gehört.

> **Hinweis:** Systemtabellen sind nicht enthalten. Dies sollte irgendwann behoben werden.

**Tabelle 35.63. Spalten von `view_table_usage`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `view_catalog` | `sql_identifier` | Name der Datenbank, die die Sicht enthält; immer die aktuelle Datenbank |
| `view_schema` | `sql_identifier` | Name des Schemas, das die Sicht enthält |
| `view_name` | `sql_identifier` | Name der Sicht |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die von der Sicht verwendete Tabelle enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die von der Sicht verwendete Tabelle enthält |
| `table_name` | `sql_identifier` | Name der Tabelle, die von der Sicht verwendet wird |

## 35.66. views

Die Sicht `views` enthält alle Sichten, die in der aktuellen Datenbank definiert sind. Angezeigt werden nur Sichten, auf die der aktuelle Benutzer Zugriff hat, entweder weil er Eigentümer ist oder weil er ein Privileg besitzt.

**Tabelle 35.64. Spalten von `views`**

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `table_catalog` | `sql_identifier` | Name der Datenbank, die die Sicht enthält; immer die aktuelle Datenbank |
| `table_schema` | `sql_identifier` | Name des Schemas, das die Sicht enthält |
| `table_name` | `sql_identifier` | Name der Sicht |
| `view_definition` | `character_data` | Abfrageausdruck, der die Sicht definiert; null, wenn die Sicht nicht einer derzeit aktivierten Rolle gehört |
| `check_option` | `character_data` | `CASCADED` oder `LOCAL`, wenn für die Sicht eine `CHECK OPTION` definiert ist, sonst `NONE` |
| `is_updatable` | `yes_or_no` | `YES`, wenn die Sicht aktualisierbar ist (also `UPDATE` und `DELETE` erlaubt), sonst `NO` |
| `is_insertable_into` | `yes_or_no` | `YES`, wenn in die Sicht eingefügt werden kann (also `INSERT` erlaubt), sonst `NO` |
| `is_trigger_updatable` | `yes_or_no` | `YES`, wenn für die Sicht ein `INSTEAD OF UPDATE`-Trigger definiert ist, sonst `NO` |
| `is_trigger_deletable` | `yes_or_no` | `YES`, wenn für die Sicht ein `INSTEAD OF DELETE`-Trigger definiert ist, sonst `NO` |
| `is_trigger_insertable_into` | `yes_or_no` | `YES`, wenn für die Sicht ein `INSTEAD OF INSERT`-Trigger definiert ist, sonst `NO` |
