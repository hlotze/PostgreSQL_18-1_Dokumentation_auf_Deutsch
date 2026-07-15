# 65. Eingebaute Index Access Methods

Dieses Kapitel beschreibt die eingebauten Index Access Methods von PostgreSQL. Es ergänzt die allgemeine Schnittstellendefinition aus [Kapitel 63](63_Schnittstellendefinition_für_Index_Access_Methods.md) um die konkreten Eigenschaften der mitgelieferten Zugriffsmethoden.

## 65.1. B-Tree-Indizes

### 65.1.1. Einführung

PostgreSQL enthält eine Implementierung der Standarddatenstruktur `btree`, also eines mehrwegigen balancierten Baums. Jeder Datentyp, der in eine wohldefinierte lineare Ordnung sortiert werden kann, kann durch einen B-Tree-Index indiziert werden. Die einzige Einschränkung besteht darin, dass ein Indexeintrag ungefähr ein Drittel einer Seite nicht überschreiten darf, nach TOAST-Kompression, falls diese anwendbar ist.

Da jede B-Tree-Operatorklasse ihrem Datentyp eine Sortierreihenfolge auferlegt, werden B-Tree-Operatorklassen, genauer gesagt Operatorfamilien, in PostgreSQL allgemein als Repräsentation der Sortiersemantik verwendet. Deshalb besitzen sie einige Eigenschaften, die über das hinausgehen, was nur zur Unterstützung von B-Tree-Indizes nötig wäre, und auch Systemteile, die recht weit von der B-Tree-AM entfernt sind, nutzen diese Informationen.

### 65.1.2. Verhalten von B-Tree-Operatorklassen

Wie in Tabelle 36.3 gezeigt, muss eine B-Tree-Operatorklasse fünf Vergleichsoperatoren bereitstellen: `<`, `<=`, `=`, `>=` und `>`. Man könnte erwarten, dass auch `<>` Teil der Operatorklasse sein sollte. Das ist jedoch nicht der Fall, weil eine `<>`-`WHERE`-Klausel für eine Indexsuche fast nie nützlich wäre. Für einige Zwecke behandelt der Planner `<>` als mit einer B-Tree-Operatorklasse verbunden; er findet diesen Operator jedoch über den Negator-Link des Operators `=`, nicht über `pg_amop`.

Wenn mehrere Datentypen nahezu identische Sortiersemantik besitzen, können ihre Operatorklassen in einer Operatorfamilie gruppiert werden. Das ist vorteilhaft, weil es dem Planner erlaubt, Schlüsse über typübergreifende Vergleiche zu ziehen. Jede Operatorklasse innerhalb der Familie sollte die eintypigen Operatoren und zugehörigen Support-Funktionen für ihren Eingabedatentyp enthalten, während typübergreifende Vergleichsoperatoren und Support-Funktionen »lose« in der Familie stehen. Es ist empfehlenswert, eine vollständige Menge typübergreifender Operatoren in die Familie aufzunehmen, damit der Planner jede Vergleichsbedingung darstellen kann, die er aus Transitivität ableitet.

Eine B-Tree-Operatorfamilie muss einige grundlegende Annahmen erfüllen:

- Ein Operator `=` muss eine Äquivalenzrelation sein. Für alle Nicht-Null-Werte `A`, `B` und `C` des Datentyps gilt:
  - `A = A` ist wahr (reflexives Gesetz).
  - Wenn `A = B`, dann `B = A` (symmetrisches Gesetz).
  - Wenn `A = B` und `B = C`, dann `A = C` (transitives Gesetz).

- Ein Operator `<` muss eine strenge Ordnungsrelation sein. Für alle Nicht-Null-Werte `A`, `B` und `C` gilt:
  - `A < A` ist falsch (irreflexives Gesetz).
  - Wenn `A < B` und `B < C`, dann `A < C` (transitives Gesetz).

- Die Ordnung ist außerdem total. Für alle Nicht-Null-Werte `A` und `B` gilt:
  - Genau eine der Aussagen `A < B`, `A = B` und `B < A` ist wahr (Trichotomiegesetz).

Die anderen drei Operatoren werden auf offensichtliche Weise aus `=` und `<` definiert und müssen konsistent mit ihnen arbeiten.

Für eine Operatorfamilie, die mehrere Datentypen unterstützt, müssen die obigen Gesetze auch dann gelten, wenn `A`, `B` und `C` aus beliebigen Datentypen der Familie stammen. Die transitiven Gesetze sind dabei am schwierigsten sicherzustellen, weil sie in typübergreifenden Situationen Aussagen darüber treffen, dass das Verhalten von zwei oder drei verschiedenen Operatoren konsistent ist. Es würde zum Beispiel nicht funktionieren, `float8` und `numeric` in dieselbe Operatorfamilie aufzunehmen, zumindest nicht mit der aktuellen Semantik, bei der `numeric`-Werte für Vergleiche mit `float8` nach `float8` konvertiert werden. Wegen der begrenzten Genauigkeit von `float8` gibt es unterschiedliche `numeric`-Werte, die mit demselben `float8`-Wert gleich vergleichen würden; dadurch würde das transitive Gesetz verletzt.

Eine weitere Anforderung an eine Familie mit mehreren Datentypen lautet, dass implizite oder binäre Casts zwischen enthaltenen Datentypen die zugehörige Sortierreihenfolge nicht ändern dürfen.

Warum ein B-Tree-Index diese Gesetze innerhalb eines einzelnen Datentyps benötigt, ist recht offensichtlich: Ohne sie gibt es keine Ordnung, in der Schlüssel angeordnet werden könnten. Auch Indexsuchen mit einem Vergleichsschlüssel eines anderen Datentyps erfordern, dass Vergleiche über zwei Datentypen hinweg vernünftig funktionieren. Die Erweiterung auf drei oder mehr Datentypen innerhalb einer Familie ist für den B-Tree-Indexmechanismus selbst nicht zwingend erforderlich, aber der Planner verlässt sich zu Optimierungszwecken darauf.

### 65.1.3. B-Tree-Support-Funktionen

Wie in Tabelle 36.9 gezeigt, definiert B-Tree eine erforderliche und fünf optionale Support-Funktionen. Die benutzerdefinierten Methoden sind:

- `order`
  - Für jede Kombination von Datentypen, für die eine B-Tree-Operatorfamilie Vergleichsoperatoren bereitstellt, muss sie eine Vergleichs-Support-Funktion bereitstellen. Diese wird in `pg_amproc` mit Support-Funktionsnummer 1 und mit `amproclefttype`/`amprocrighttype` gleich den linken und rechten Datentypen des Vergleichs registriert. Die Vergleichsfunktion muss zwei Nicht-Null-Werte `A` und `B` annehmen und einen `int32`-Wert zurückgeben, der kleiner als 0, gleich 0 oder größer als 0 ist, wenn `A < B`, `A = B` beziehungsweise `A > B` gilt. Ein Null-Ergebnis ist nicht erlaubt: Alle Werte des Datentyps müssen vergleichbar sein. Beispiele befinden sich in `src/backend/access/nbtree/nbtcompare.c`.
  - Wenn die verglichenen Werte einen kollatierbaren Datentyp besitzen, wird die passende Kollations-OID mit dem Standardmechanismus `PG_GET_COLLATION()` an die Vergleichs-Support-Funktion übergeben.

- `sortsupport`
  - Optional kann eine B-Tree-Operatorfamilie Sort-Support-Funktionen bereitstellen, die unter Support-Funktionsnummer 2 registriert werden. Diese Funktionen erlauben, Vergleiche für Sortierzwecke effizienter zu implementieren als durch naiven Aufruf der Vergleichs-Support-Funktion. Die beteiligten APIs sind in `src/include/utils/sortsupport.h` definiert.

- `in_range`
  - Optional kann eine B-Tree-Operatorfamilie `in_range`-Support-Funktionen bereitstellen, die unter Support-Funktionsnummer 3 registriert werden. Sie werden nicht während B-Tree-Indexoperationen verwendet. Stattdessen erweitern sie die Semantik der Operatorfamilie, sodass sie Window-Klauseln mit den Frame-Grenzen `RANGE offset PRECEDING` und `RANGE offset FOLLOWING` unterstützen kann; siehe Abschnitt 4.2.8.
  - Eine `in_range`-Funktion hat die Signatur:

    ```sql
    in_range(val type1, base type1, offset type2, sub bool, less bool)
    returns bool
    ```

  - `val` und `base` müssen denselben Typ haben, der von der Operatorfamilie geordnet wird. `offset` kann einen anderen Typ haben, auch einen sonst nicht von der Familie unterstützten. Die eingebaute Familie `time_ops` stellt zum Beispiel eine `in_range`-Funktion bereit, deren `offset` den Typ `interval` hat.
  - Die Funktion soll abhängig von `sub` und `less` `base` und `offset` addieren oder subtrahieren und anschließend `val` vergleichen:
    - Wenn `!sub` und `!less`, gib `val >= (base + offset)` zurück.
    - Wenn `!sub` und `less`, gib `val <= (base + offset)` zurück.
    - Wenn `sub` und `!less`, gib `val >= (base - offset)` zurück.
    - Wenn `sub` und `less`, gib `val <= (base - offset)` zurück.
  - Vorher muss die Funktion das Vorzeichen von `offset` prüfen. Ist es kleiner als null, muss sie Fehlercode `ERRCODE_INVALID_PRECEDING_OR_FOLLOWING_SIZE` (`22013`) mit einem Text wie »invalid preceding or following size in window function« auslösen. Diese Anforderung wird an `in_range` delegiert, damit der Kerncode nicht wissen muss, was »kleiner als null« für einen bestimmten Datentyp bedeutet.
  - Wenn praktikabel, sollten `in_range`-Funktionen keinen Fehler werfen, wenn `base + offset` oder `base - offset` überlaufen würde. Das korrekte Vergleichsergebnis kann häufig trotzdem bestimmt werden. Wenn der Datentyp Konzepte wie »infinity« oder `NaN` enthält, ist zusätzliche Sorgfalt nötig, damit die Ergebnisse von `in_range` mit der normalen Sortierreihenfolge der Operatorfamilie übereinstimmen.
  - Die Ergebnisse müssen monoton zur Sortierreihenfolge passen. Für feste Werte von `offset` und `sub` gilt insbesondere: Ist `in_range` mit `less = true` für `val1` und `base` wahr, muss es für jedes `val2 <= val1` mit derselben `base` wahr sein; ist es falsch, muss es für jedes `val2 >= val1` falsch sein. Entsprechende Bedingungen gelten bei Variation von `base`, und mit umgekehrten Bedingungen für `less = false`.

- `equalimage`
  - Optional kann eine B-Tree-Operatorfamilie `equalimage`-Support-Funktionen bereitstellen, registriert unter Support-Funktionsnummer 4. »Equality implies image equality« erlaubt dem Kerncode festzustellen, wann die B-Tree-Deduplizierungsoptimierung sicher angewendet werden kann. Derzeit werden `equalimage`-Funktionen nur beim Erzeugen oder Neuerzeugen eines Index aufgerufen.
  - Die Signatur lautet:

    ```sql
    equalimage(opcintype oid) returns bool
    ```

  - Der Rückgabewert ist statische Information über eine Operatorklasse und Kollation. `true` bedeutet, dass die Ordnungsfunktion der Operatorklasse nur dann 0 zurückgibt, wenn ihre Argumente auch ohne Verlust semantischer Information austauschbar sind. Keine registrierte Funktion oder `false` bedeutet, dass diese Bedingung nicht angenommen werden darf.
  - Im Kern wird Deduplizierung nur dann als sicher betrachtet, wenn jede indizierte Spalte eine Operatorklasse verwendet, die eine `equalimage`-Funktion registriert, und jeder Aufruf tatsächlich `true` zurückgibt. Bildgleichheit ist fast, aber nicht ganz, bitweise Gleichheit; bei `varlena`-Datentypen kann TOAST-Kompression gleiche Werte auf der Platte unterschiedlich darstellen.

- `options`
  - Optional kann eine B-Tree-Operatorfamilie eine `options`-Support-Funktion bereitstellen, registriert unter Support-Funktionsnummer 5. Sie definiert benutzersichtbare Parameter, die das Verhalten der Operatorklasse steuern.
  - Die Signatur lautet:

    ```sql
    options(relopts local_relopts *) returns void
    ```

  - Die Funktion erhält einen Zeiger auf eine `local_relopts`-Struktur, die mit den operatorklassenspezifischen Optionen gefüllt werden muss. Andere Support-Funktionen können diese Optionen mit den Makros `PG_HAS_OPCLASS_OPTIONS()` und `PG_GET_OPCLASS_OPTIONS()` abrufen.

- `skipsupport`
  - Optional kann eine B-Tree-Operatorfamilie eine Skip-Support-Funktion bereitstellen, registriert unter Support-Funktionsnummer 6. Diese Funktionen geben dem B-Tree-Code eine Möglichkeit, alle möglichen Werte, die vom zugrunde liegenden Eingabetyp einer Operatorklasse dargestellt werden können, in Schlüsselraumordnung zu durchlaufen. Dies wird von der Skip-Scan-Optimierung verwendet. Die APIs sind in `src/include/utils/skipsupport.h` definiert.
  - Operatorklassen ohne Skip-Support-Funktion können weiterhin Skip Scan verwenden; der Kerncode verwendet dann eine Fallback-Strategie. Für kontinuierliche Typen ist eine Skip-Support-Funktion meistens weder sinnvoll noch machbar.

### 65.1.4. Implementierung

Dieser Abschnitt behandelt Implementierungsdetails von B-Tree-Indizes, die für fortgeschrittene Benutzer nützlich sein können. Eine deutlich detailliertere, internals-orientierte Beschreibung steht in `src/backend/access/nbtree/README`.

#### 65.1.4.1. B-Tree-Struktur

PostgreSQL-B-Tree-Indizes sind mehrstufige Baumstrukturen, bei denen jede Ebene des Baums als doppelt verkettete Liste von Seiten verwendet werden kann. Eine einzelne Metaseite wird an einer festen Position am Anfang der ersten Segmentdatei des Index gespeichert. Alle anderen Seiten sind entweder Leaf-Seiten oder interne Seiten. Leaf-Seiten sind die Seiten auf der untersten Ebene des Baums. Alle anderen Ebenen bestehen aus internen Seiten. Jede Leaf-Seite enthält Tupel, die auf Tabellenzeilen zeigen. Jede interne Seite enthält Tupel, die auf die nächsttiefere Ebene im Baum zeigen. Typischerweise sind mehr als 99 Prozent aller Seiten Leaf-Seiten. Sowohl interne Seiten als auch Leaf-Seiten verwenden das in [Abschnitt 66.6](66_Physische_Speicherung_der_Datenbank.md#666-layout-von-datenbankseiten) beschriebene Standardseitenformat.

Neue Leaf-Seiten werden zu einem B-Tree-Index hinzugefügt, wenn auf einer vorhandenen Leaf-Seite kein Platz für ein eingehendes Tupel ist. Eine Seitenteilung schafft Platz, indem ein Teil der Einträge auf eine neue Seite verschoben wird. Seitenteilungen müssen außerdem einen neuen Downlink auf die neue Seite in die Elternseite einfügen, was wiederum deren Teilung auslösen kann. Seitenteilungen kaskadieren rekursiv nach oben. Wenn schließlich die Root-Seite keinen neuen Downlink aufnehmen kann, erfolgt eine Root-Seitenteilung. Dadurch wird der Baumstruktur eine neue Ebene hinzugefügt, indem eine neue Root-Seite oberhalb der ursprünglichen Root-Seite erzeugt wird.

#### 65.1.4.2. Bottom-up Index Deletion

B-Tree-Indizes wissen nicht direkt, dass unter MVCC mehrere vorhandene Versionen derselben logischen Tabellenzeile existieren können. Für einen Index ist jedes Tupel ein eigenständiges Objekt, das einen eigenen Indexeintrag benötigt. Tupel aus »Version Churn« können sich ansammeln und Abfragelatenz sowie Durchsatz beeinträchtigen. Das tritt typischerweise bei update-lastigen Workloads auf, bei denen die meisten Updates die HOT-Optimierung nicht verwenden können. Schon die Änderung einer einzigen von einem Index abgedeckten Spalte während eines `UPDATE` erfordert einen neuen Satz von Indextupeln, je eines für jeden Index der Tabelle. Dazu gehören insbesondere auch Indizes, die durch das `UPDATE` nicht »logisch geändert« wurden.

B-Tree-Indizes löschen Version-Churn-Indextupel inkrementell durch Bottom-up-Index-Deletion-Läufe. Ein solcher Lauf wird als Reaktion auf eine erwartete Seitenteilung ausgelöst, die durch Version Churn entstehen würde. Dies geschieht nur bei Indizes, die durch `UPDATE`-Anweisungen nicht logisch geändert werden, bei denen sich sonst veraltete Versionen konzentriert auf bestimmten Seiten ansammeln würden. Häufig lässt sich eine Seitenteilung vermeiden, auch wenn Heuristiken manchmal keine Garbage-Indextupel finden; in diesem Fall löst eine Seitenteilung oder Deduplizierung das Platzproblem.

> Nicht alle Löschoperationen innerhalb von B-Tree-Indizes sind Bottom-up-Löschungen. Es gibt auch einfache Indextupel-Löschung: eine zurückgestellte Wartungsoperation, die Indextupel löscht, von denen bekannt ist, dass sie sicher gelöscht werden können, weil das `LP_DEAD`-Bit ihres Item-Identifiers bereits gesetzt ist. Wie Bottom-up-Löschung findet einfache Löschung an dem Punkt statt, an dem eine Seitenteilung erwartet wird, um diese Teilung zu vermeiden.
>
> Einfache Löschung ist opportunistisch, weil sie nur stattfinden kann, wenn aktuelle Indexscans nebenbei die `LP_DEAD`-Bits betroffener Items gesetzt haben. Vor PostgreSQL 14 war einfache Löschung die einzige Kategorie von B-Tree-Löschung. Der Hauptunterschied besteht darin, dass nur einfache Löschung opportunistisch durch vorbeikommende Indexscans getrieben wird, während nur Bottom-up-Löschung gezielt Version Churn aus `UPDATE`s angeht, die indizierte Spalten nicht logisch ändern.

Bottom-up Index Deletion übernimmt bei bestimmten Workloads den größten Teil der Garbage-Bereinigung für einzelne Indizes. Das ist für B-Tree-Indizes zu erwarten, die starkem Version Churn durch `UPDATE`s ausgesetzt sind, welche die abgedeckten Spalten selten oder nie logisch ändern. Die durchschnittliche und schlimmste Anzahl von Versionen pro logischer Zeile kann allein durch gezielte inkrementelle Löschläufe niedrig gehalten werden. Trotzdem wird irgendwann ein vollständiger Bereinigungslauf durch `VACUUM`, typischerweise in einem Autovacuum-Worker, als Teil der gemeinsamen Bereinigung der Tabelle und all ihrer Indizes erforderlich.

Anders als `VACUUM` gibt Bottom-up Index Deletion keine starken Garantien darüber, wie alt das älteste Garbage-Indextupel sein darf. Kein Index darf »floating garbage« behalten, das vor einem konservativen Cutoff-Zeitpunkt tot wurde, der gemeinsam für Tabelle und alle ihre Indizes gilt. Diese grundlegende Tabelleninvariante macht es sicher, Tabellen-TIDs wiederzuverwenden.

#### 65.1.4.3. Deduplizierung

Ein Duplikat ist ein Leaf-Seiten-Tupel, bei dem alle indizierten Schlüsselspalten Werte besitzen, die mit entsprechenden Spaltenwerten mindestens eines anderen Leaf-Seiten-Tupels im selben Index übereinstimmen. Solche Duplikate sind in der Praxis häufig. B-Tree-Indizes können für Duplikate eine spezielle, platzsparende Darstellung verwenden, wenn die optionale Technik der Deduplizierung aktiviert ist.

Deduplizierung fasst Gruppen von Duplikattupeln periodisch zusammen und erzeugt ein einzelnes Posting-List-Tupel für jede Gruppe. Die Schlüsselwerte erscheinen in dieser Darstellung nur einmal. Danach folgt ein sortiertes Array von TIDs, die auf Zeilen in der Tabelle zeigen. Das reduziert die Speichergröße von Indizes erheblich, wenn jeder Wert oder jede unterschiedliche Kombination von Spaltenwerten im Durchschnitt mehrfach erscheint. Abfragelatenz und Index-Vacuuming-Overhead können deutlich sinken, und der Gesamtdurchsatz kann steigen.

> B-Tree-Deduplizierung ist auch bei »Duplikaten« wirksam, die einen `NULL`-Wert enthalten, obwohl `NULL`-Werte nach dem `=`-Mitglied einer B-Tree-Operatorklasse niemals gleich sind. Für den Teil der Implementierung, der die B-Tree-Struktur auf Platte versteht, ist `NULL` lediglich ein weiterer Wert aus der Domäne indizierter Werte.

Der Deduplizierungsprozess erfolgt lazy, wenn ein neues Item eingefügt wird, das nicht auf eine vorhandene Leaf-Seite passt, und nur dann, wenn Indextupel-Löschung nicht genug Platz freimachen konnte. Anders als GIN-Posting-List-Tupel müssen B-Tree-Posting-List-Tupel nicht jedes Mal wachsen, wenn ein neues Duplikat eingefügt wird; sie sind nur eine alternative physische Darstellung der ursprünglichen logischen Inhalte der Leaf-Seite. Dieses Design priorisiert konstante Performance bei gemischten Lese-/Schreib-Workloads. Deduplizierung ist standardmäßig aktiviert.

`CREATE INDEX` und `REINDEX` wenden Deduplizierung beim Erzeugen von Posting-List-Tupeln ebenfalls an, verwenden dabei aber eine leicht andere Strategie: Gruppen gewöhnlicher Duplikattupel aus der sortierten Tabelleneingabe werden vor dem Hinzufügen zur aktuellen ausstehenden Leaf-Seite in Posting-List-Tupel zusammengeführt.

Schreiblastige Workloads mit wenigen oder keinen Duplikatwerten zahlen eine kleine feste Performance-Strafe, sofern Deduplizierung nicht ausdrücklich deaktiviert wird. Der Storage-Parameter `deduplicate_items` kann Deduplizierung für einzelne Indizes deaktivieren. Bei reinen Lese-Workloads gibt es keine Strafe, weil das Lesen von Posting-List-Tupeln mindestens so effizient ist wie das Lesen der Standarddarstellung.

Eindeutige Indizes können Deduplizierung manchmal ebenfalls verwenden. Dadurch können Leaf-Seiten zusätzliche Version-Churn-Duplikate vorübergehend »absorbieren«. Besonders bei lang laufenden Transaktionen, deren Snapshot Garbage Collection blockiert, ergänzt Deduplizierung in eindeutigen Indizes die Bottom-up Index Deletion.

> Für eindeutige Indizes wird eine spezielle Heuristik angewendet, um zu bestimmen, ob ein Deduplizierungslauf stattfinden soll. Sie kann oft direkt zur Seitenteilung springen und so eine Performance-Strafe durch nutzlose Deduplizierungsläufe vermeiden. Wenn Sie sich Sorgen um den Overhead der Deduplizierung machen, erwägen Sie, `deduplicate_items = off` selektiv zu setzen. Deduplizierung in eindeutigen Indizes aktiviert zu lassen, hat nur geringe Nachteile.

Deduplizierung kann aufgrund von Implementierungseinschränkungen nicht in allen Fällen verwendet werden. Beim Ausführen von `CREATE INDEX` oder `REINDEX` wird bestimmt, ob Deduplizierung sicher ist.

Deduplizierung gilt in folgenden Fällen als unsicher:

- `text`, `varchar` und `char` können bei nichtdeterministischen Kollationen keine Deduplizierung verwenden, weil Unterschiede in Groß-/Kleinschreibung und Akzenten erhalten bleiben müssen.
- `numeric` kann keine Deduplizierung verwenden, weil die Anzeige-Skala gleicher Werte erhalten bleiben muss.
- `jsonb` kann keine Deduplizierung verwenden, weil die `jsonb`-B-Tree-Operatorklasse intern `numeric` verwendet.
- `float4` und `float8` können keine Deduplizierung verwenden, weil diese Typen unterschiedliche Darstellungen für `-0` und `0` haben, obwohl diese als gleich gelten.
- Container-Typen wie Composite Types, Arrays oder Range Types können derzeit keine Deduplizierung verwenden.
- `INCLUDE`-Indizes können nie Deduplizierung verwenden.

## 65.2. GiST-Indizes

### 65.2.1. Einführung

GiST steht für Generalized Search Tree. Es ist eine balancierte, baumstrukturierte Zugriffsmethode, die als Basisschablone dient, in der beliebige Indizierungsschemata implementiert werden können. B-Trees, R-Trees und viele andere Schemata lassen sich in GiST implementieren.

Ein Vorteil von GiST besteht darin, dass Fachleute für einen Datentyp benutzerdefinierte Datentypen mit passenden Zugriffsmethoden entwickeln können, ohne Datenbankspezialisten sein zu müssen.

Einige Informationen hier stammen aus dem GiST Indexing Project der University of California at Berkeley sowie aus Marcel Kornackers Arbeit *Access Methods for Next-Generation Database Systems*. Die GiST-Implementierung in PostgreSQL wird hauptsächlich von Teodor Sigaev und Oleg Bartunov gepflegt.

```text
http://gist.cs.berkeley.edu/
http://www.sai.msu.su/~megera/postgres/gist/papers/concurrency/access-methods-for-next-generation.pdf.gz
http://www.sai.msu.su/~megera/postgres/gist/
```

### 65.2.2. Eingebaute Operatorklassen

Die PostgreSQL-Kerndistribution enthält GiST-Operatorklassen für geometrische Typen, Netzwerktypen, Range- und Multirange-Typen, `tsquery` und `tsvector`. Einige optionale Module aus [Anhang F](F_Weitere_mitgelieferte_Module_und_Erweiterungen.md) stellen zusätzliche GiST-Operatorklassen bereit.

| Operatorklasse | Indexierbare Operatoren | Ordnungsoperatoren |
| --- | --- | --- |
| `box_ops` | `<<`, `&<`, `&&`, `&>`, `>>`, `~=`, `@>`, `<@`, `&<|`, `<<|`, `|>>`, `|&>` für `box` | `<-> (box, point)` |
| `circle_ops` | `<<`, `&<`, `&>`, `>>`, `<@`, `@>`, `~=`, `&&`, `|>>`, `<<|`, `&<|`, `|&>` für `circle` | `<-> (circle, point)` |
| `inet_ops` | `<<`, `<<=`, `>>`, `>>=`, `=`, `<>`, `<`, `<=`, `>`, `>=`, `&&` für `inet` | |
| `multirange_ops` | `=`, `&&`, `@>`, `<@`, `<<`, `>>`, `&<`, `&>`, `-|-` für Multiranges und Ranges | |
| `point_ops` | `|>>`, `<<`, `>>`, `<<|`, `~=`, `<@` für `point` | `<-> (point, point)` |
| `poly_ops` | `<<`, `&<`, `&>`, `>>`, `<@`, `@>`, `~=`, `&&`, `<<|`, `&<|`, `|&>`, `|>>` für `polygon` | `<-> (polygon, point)` |
| `range_ops` | `=`, `&&`, `@>`, `<@`, `<<`, `>>`, `&<`, `&>`, `-|-` für Ranges und Multiranges | |
| `tsquery_ops` | `<@`, `@>` für `tsquery` | |
| `tsvector_ops` | `@@ (tsvector, tsquery)` | |

Aus historischen Gründen ist `inet_ops` nicht die Standardklasse für die Typen `inet` und `cidr`. Um sie zu verwenden, muss der Klassenname in `CREATE INDEX` angegeben werden:

```sql
CREATE INDEX ON my_table USING GIST (my_inet_column inet_ops);
```

### 65.2.3. Erweiterbarkeit

Traditionell bedeutete die Implementierung einer neuen Index Access Method viel schwierige Arbeit, unter anderem Verständnis von Lock Manager und Write-Ahead Log. Die GiST-Schnittstelle besitzt ein hohes Abstraktionsniveau: Der Implementierer muss nur die Semantik des zu indizierenden Datentyps implementieren. Die GiST-Schicht kümmert sich selbst um Nebenläufigkeit, Logging und die Suche in der Baumstruktur.

Diese Erweiterbarkeit darf nicht mit der Erweiterbarkeit anderer Standard-Suchbäume hinsichtlich der Datentypen verwechselt werden. PostgreSQL unterstützt etwa erweiterbare B-Trees und Hash-Indizes. Das bedeutet, dass ein B-Tree oder Hash-Index für beliebige Datentypen gebaut werden kann. B-Trees unterstützen aber nur Bereichsprädikate wie `<`, `=` und `>`, Hash-Indizes nur Gleichheitsabfragen. Mit einem GiST-basierten Index können dagegen domänenspezifische Fragen formuliert werden, etwa »finde alle Bilder von Pferden« oder »finde alle überbelichteten Bilder«, sofern die Operatorklasse diese Semantik implementiert.

Um eine GiST-Zugriffsmethode nutzbar zu machen, müssen mehrere benutzerdefinierte Methoden implementiert werden, die das Verhalten von Schlüsseln im Baum definieren. GiST verbindet Erweiterbarkeit mit Allgemeinheit, Wiederverwendung von Code und einer klaren Schnittstelle.

Eine GiST-Operatorklasse muss fünf Methoden bereitstellen; weitere Methoden sind optional:

- `consistent`
  - Bestimmt für einen Indexeintrag `p` und einen Abfragewert `q`, ob der Indexeintrag mit der Abfrage konsistent ist, also ob das Prädikat `indexed_column indexable_operator q` für irgendeine durch den Indexeintrag repräsentierte Zeile wahr sein könnte. Bei Leaf-Einträgen entspricht das dem Test der indexierbaren Bedingung; bei internen Knoten entscheidet es, ob der zugehörige Teilbaum durchsucht werden muss. Bei Ergebnis `true` muss außerdem ein `recheck`-Flag zurückgegeben werden. Ist `recheck = false`, wurde exakt geprüft; ist `recheck = true`, ist die Zeile nur ein Kandidat.

  ```sql
  CREATE OR REPLACE FUNCTION my_consistent(internal, data_type,
      smallint, oid, internal)
  RETURNS bool
  AS 'MODULE_PATHNAME'
  LANGUAGE C STRICT;
  ```

- `union`
  - Konsolidiert Informationen im Baum. Aus einer Menge von Einträgen erzeugt diese Funktion einen neuen Indexeintrag, der alle gegebenen Einträge repräsentiert. Das Ergebnis muss ein Wert des Storage-Typs des Index sein und auf neu mit `palloc()` allozierten Speicher zeigen.

  ```sql
  CREATE OR REPLACE FUNCTION my_union(internal, internal)
  RETURNS storage_type
  AS 'MODULE_PATHNAME'
  LANGUAGE C STRICT;
  ```

- `compress`
  - Konvertiert ein Datenelement in ein Format, das zur physischen Speicherung auf einer Indexseite geeignet ist. Wird `compress` ausgelassen, werden Datenelemente unverändert im Index gespeichert.

  ```sql
  CREATE OR REPLACE FUNCTION my_compress(internal)
  RETURNS internal
  AS 'MODULE_PATHNAME'
  LANGUAGE C STRICT;
  ```

- `decompress`
  - Konvertiert die gespeicherte Darstellung eines Datenelements in ein Format, das von den anderen GiST-Methoden der Operatorklasse verarbeitet werden kann. Wird `decompress` ausgelassen, wird angenommen, dass die anderen Methoden direkt auf dem gespeicherten Datenformat arbeiten können.

- `penalty`
  - Gibt einen Wert zurück, der die »Kosten« des Einfügens eines neuen Eintrags in einen bestimmten Baumzweig angibt. Einträge werden entlang des Pfads mit der geringsten Penalty eingefügt. Rückgabewerte sollten nicht negativ sein; negative Werte werden wie null behandelt.

- `picksplit`
  - Entscheidet bei einer Seitenteilung, welche Einträge auf der alten Seite bleiben und welche auf die neue Seite verschoben werden. Wie `penalty` ist `picksplit` entscheidend für gute Indexperformance.

- `same`
  - Gibt an, ob zwei Indexeinträge identisch sind. Aus historischen Gründen speichert die Funktion das boolesche Ergebnis in einem Ausgabeparameter, statt es direkt zurückzugeben.

- `distance`
  - Bestimmt für einen Indexeintrag und einen Abfragewert die »Distanz« des Indexeintrags zum Abfragewert. Diese Funktion ist erforderlich, wenn die Operatorklasse Ordnungsoperatoren enthält. Sie ermöglicht geordnete Scans, insbesondere Nearest-Neighbor-Suchen.

- `fetch`
  - Konvertiert die komprimierte Indexdarstellung eines Datenelements für Index-Only-Scans zurück in den ursprünglichen Datentyp. Die zurückgegebenen Daten müssen eine exakte, nicht-lossy Kopie des ursprünglich indizierten Werts sein. Wenn `compress` für Leaf-Einträge lossy ist, darf die Operatorklasse keine `fetch`-Funktion definieren.

- `options`
  - Definiert benutzersichtbare Parameter, die das Verhalten der Operatorklasse steuern. Die Optionen werden über `local_relopts` definiert und können von anderen Support-Funktionen mit `PG_HAS_OPCLASS_OPTIONS()` und `PG_GET_OPCLASS_OPTIONS()` gelesen werden.

- `sortsupport`
  - Gibt eine Vergleichsfunktion zurück, die Daten so sortiert, dass Lokalität erhalten bleibt. Diese Methode wird von `CREATE INDEX` und `REINDEX` verwendet. Fehlt sie, wird der Index durch Einfügen jedes Tupels in den Baum mit `penalty` und `picksplit` gebaut, was deutlich langsamer ist.

- `translate_cmptype`
  - Übersetzt einen `CompareType`-Wert aus `src/include/nodes/primnodes.h` in eine Strategienummer der Operatorklasse. Dies wird für temporale Index-Constraints verwendet.

Alle GiST-Support-Methoden werden normalerweise in kurzlebigen Speicher-Kontexten aufgerufen. Wenn eine Support-Methode Daten über wiederholte Aufrufe hinweg zwischenspeichern möchte, sollte sie langlebige Daten in `fcinfo->flinfo->fn_mcxt` allozieren und einen Zeiger in `fcinfo->flinfo->fn_extra` speichern.

### 65.2.4. Implementierung

#### 65.2.4.1. GiST-Index-Build-Methoden

Der einfachste Weg, einen GiST-Index zu bauen, besteht darin, alle Einträge einzeln einzufügen. Das ist bei großen Indizes tendenziell langsam, weil viele zufällige I/O-Zugriffe nötig sind, wenn die Indextupel über den Index verteilt sind und der Index nicht in den Cache passt. PostgreSQL unterstützt zwei alternative Methoden für den initialen Build eines GiST-Index: sortierten und gepufferten Modus.

Die sortierte Methode ist nur verfügbar, wenn jede verwendete Operatorklasse eine `sortsupport`-Funktion bereitstellt. Ist das der Fall, ist sie normalerweise die beste Methode und wird standardmäßig verwendet.

Die gepufferte Methode fügt Tupel nicht sofort direkt in den Index ein. Sie kann die Menge zufälliger I/O bei nicht geordneten Datensätzen stark reduzieren. Bei gut geordneten Datensätzen ist der Nutzen kleiner oder nicht vorhanden, weil jeweils nur wenige Seiten neue Tupel erhalten und diese Seiten im Cache liegen.

Falls Sortierung nicht möglich ist, wechselt ein GiST-Index-Build standardmäßig zur gepufferten Methode, sobald die Indexgröße `effective_cache_size` erreicht. Buffering kann mit dem Parameter `buffering` des Befehls `CREATE INDEX` manuell erzwungen oder verhindert werden.

### 65.2.5. Beispiele

Die PostgreSQL-Quelldistribution enthält mehrere Beispiele für mit GiST implementierte Indexmethoden. Das Kernsystem stellt derzeit Textsuche-Unterstützung für `tsvector` und `tsquery` sowie R-Tree-ähnliche Funktionalität für einige eingebaute geometrische Datentypen bereit; siehe `src/backend/access/gist/gistproc.c`.

Zusätzliche GiST-Operatorklassen befinden sich unter anderem in diesen `contrib`-Modulen:

- `btree_gist`: B-Tree-ähnliche Funktionalität für mehrere Datentypen
- `cube`: Indizierung mehrdimensionaler Cubes
- `hstore`: Speicherung von Schlüssel/Wert-Paaren
- `intarray`: RD-Tree für eindimensionale Arrays von `int4`-Werten
- `ltree`: Indizierung baumartiger Strukturen
- `pg_trgm`: Textähnlichkeit mit Trigramm-Matching
- `seg`: Indizierung von »float ranges«

## 65.3. SP-GiST-Indizes

### 65.3.1. Einführung

SP-GiST steht für Space-Partitioned GiST. SP-GiST unterstützt partitionierte Suchbäume und erleichtert die Entwicklung einer breiten Palette nichtbalancierter Datenstrukturen wie Quadtrees, k-d-Trees und Radix Trees (Tries). Gemeinsam ist diesen Strukturen, dass sie den Suchraum wiederholt in Partitionen teilen, die nicht gleich groß sein müssen. Suchen, die gut zur Partitionierungsregel passen, können sehr schnell sein.

Diese Datenstrukturen wurden ursprünglich für die Verwendung im Hauptspeicher entwickelt. Dort bestehen sie meist aus dynamisch allozierten Knoten, die über Zeiger verbunden sind. Für direkte Speicherung auf Platte eignet sich das nicht, weil solche Zeigerketten sehr lang sein können und zu viele Plattenzugriffe erfordern würden. Plattenbasierte Datenstrukturen sollten dagegen einen hohen Fanout besitzen, um I/O zu minimieren. SP-GiST löst die Aufgabe, Suchbaumknoten so auf Plattenseiten abzubilden, dass eine Suche nur wenige Plattenseiten lesen muss, selbst wenn sie viele Knoten durchläuft.

Wie GiST soll SP-GiST die Entwicklung eigener Datentypen mit passenden Zugriffsmethoden durch Domänenexperten ermöglichen.

```text
https://www.cs.purdue.edu/spgist/
http://www.sai.msu.su/~megera/wiki/spgist_dev
```

### 65.3.2. Eingebaute Operatorklassen

Die PostgreSQL-Kerndistribution enthält SP-GiST-Operatorklassen für `box`, `inet`, `point`, `polygon`, `range` und `text`.

| Operatorklasse | Indexierbare Operatoren | Ordnungsoperatoren |
| --- | --- | --- |
| `box_ops` | räumliche Box-Operatoren wie `<<`, `&<`, `&>`, `>>`, `<@`, `@>`, `~=`, `&&` und vertikale Varianten | `<-> (box, point)` |
| `inet_ops` | `<<`, `<<=`, `>>`, `>>=`, `=`, `<>`, `<`, `<=`, `>`, `>=`, `&&` | |
| `kd_point_ops` | Punktoperatoren wie `|>>`, `<<`, `>>`, `<<|`, `~=`, `<@` | `<-> (point, point)` |
| `poly_ops` | Polygonoperatoren wie `<<`, `&<`, `&>`, `>>`, `<@`, `@>`, `~=`, `&&` und vertikale Varianten | `<-> (polygon, point)` |
| `quad_point_ops` | Punktoperatoren wie `|>>`, `<<`, `>>`, `<<|`, `~=`, `<@` | `<-> (point, point)` |
| `range_ops` | `=`, `&&`, `@>`, `<@`, `<<`, `>>`, `&<`, `&>`, `-|-` für Ranges | |
| `text_ops` | `=`, `<`, `<=`, `>`, `>=`, `~<~`, `~<=~`, `~>=~`, `~>~`, `^@` | |

Von den beiden Operatorklassen für `point` ist `quad_point_ops` die Standardklasse. `kd_point_ops` unterstützt dieselben Operatoren, verwendet aber eine andere Indexdatenstruktur, die in manchen Anwendungen bessere Performance bieten kann. `quad_point_ops`, `kd_point_ops` und `poly_ops` unterstützen den Ordnungsoperator `<->`, der k-Nearest-Neighbor-Suche über indizierte Punkt- oder Polygondatensätze ermöglicht.

### 65.3.3. Erweiterbarkeit

SP-GiST bietet eine Schnittstelle mit hohem Abstraktionsniveau: Der Entwickler einer Zugriffsmethode muss nur Methoden implementieren, die spezifisch für einen Datentyp sind. Der SP-GiST-Kern ist für effiziente Abbildung auf Platte, Suche in der Baumstruktur, Nebenläufigkeit und Logging verantwortlich.

Leaf-Tupel eines SP-GiST-Baums enthalten normalerweise Werte desselben Datentyps wie die indizierte Spalte. Sie können aber auch lossy Darstellungen der indizierten Spalte enthalten. Leaf-Tupel auf Root-Ebene repräsentieren den ursprünglich indizierten Datenwert direkt; Leaf-Tupel auf tieferen Ebenen können dagegen nur einen Teilwert enthalten, etwa ein Suffix. In diesem Fall müssen die Support-Funktionen der Operatorklasse den ursprünglichen Wert mithilfe von Informationen rekonstruieren können, die beim Abstieg durch interne Tupel gesammelt wurden.

Wenn ein SP-GiST-Index mit `INCLUDE`-Spalten erzeugt wird, werden die Werte dieser Spalten ebenfalls in Leaf-Tupeln gespeichert. Die `INCLUDE`-Spalten sind für die SP-GiST-Operatorklasse ohne Bedeutung.

Interne Tupel sind komplexer, weil sie Verzweigungspunkte im Suchbaum darstellen. Jedes interne Tupel enthält eine Menge von einem oder mehreren Knoten, die Gruppen ähnlicher Leaf-Werte repräsentieren. Ein Knoten enthält einen Downlink, der entweder zu einem weiteren internen Tupel tieferer Ebene oder zu einer kurzen Liste von Leaf-Tupeln auf derselben Indexseite führt. Ein Knoten hat normalerweise ein Label, das ihn beschreibt; in einem Radix Tree könnte das Label etwa das nächste Zeichen des Stringwerts sein. Alternativ kann eine Operatorklasse Knotenlabels auslassen, wenn sie mit einer festen Knotengruppe für alle internen Tupel arbeitet; siehe [Abschnitt 65.3.4.2](65_Eingebaute_Index_Access_Methods.md#65342-spgist-ohne-knotenlabels).

Optional kann ein internes Tupel einen Präfixwert besitzen, der alle seine Mitglieder beschreibt. In einem Radix Tree wäre das etwa das gemeinsame Präfix der repräsentierten Strings. Der Präfixwert muss nicht tatsächlich ein Präfix sein; er kann beliebige von der Operatorklasse benötigte Daten enthalten.

> Der SP-GiST-Kerncode kümmert sich um Null-Einträge. Obwohl SP-GiST-Indizes Einträge für Nullwerte in indizierten Spalten speichern, ist dies vor dem Code der Indexoperatorklasse verborgen: Null-Indexeinträge oder Suchbedingungen werden nie an die Methoden der Operatorklasse übergeben. Es wird angenommen, dass SP-GiST-Operatoren strikt sind und daher für Nullwerte nicht erfolgreich sein können.

Eine SP-GiST-Operatorklasse muss fünf benutzerdefinierte Methoden bereitstellen, zwei weitere sind optional:

- `config`
  - Gibt statische Informationen über die Indeximplementierung zurück, darunter die Datentyp-OIDs für Präfixe, Knotenlabels und Leaf-Werte.
  - Die Funktion füllt eine `spgConfigOut`-Struktur. Für Operatorklassen ohne Präfixe kann `prefixType` auf `VOIDOID` gesetzt werden; für Klassen ohne Knotenlabels entsprechend `labelType`. `canReturnData` sollte `true` sein, wenn die Operatorklasse den ursprünglich gelieferten Indexwert rekonstruieren kann. `longValuesOK` darf nur `true` sein, wenn variable lange Werte durch wiederholtes Suffixing segmentiert werden können.

- `choose`
  - Wählt beim Einfügen eines neuen Werts in ein internes Tupel die passende Aktion. Die Funktion kann in einen vorhandenen Knoten absteigen (`spgMatchNode`), einen neuen Knoten hinzufügen (`spgAddNode`) oder das interne Tupel splitten (`spgSplitTuple`).
  - `datum` ist der ursprüngliche einzufügende Wert; `leafDatum` ist der aktuell auf Leaf-Ebene zu speichernde Wert. `level` zählt die aktuelle Ebene ab null.

- `picksplit`
  - Gruppiert mehrere Leaf-Tupel in Knoten, wenn eine Leaf-Tupelliste zu groß wird. Die Funktion setzt Präfixinformationen, Knotenlabels, die Zuordnung von Tupeln zu Knoten und die Leaf-Datums, die in neuen Leaf-Tupeln gespeichert werden.
  - Wenn mehr als ein Leaf-Tupel übergeben wird, sollte `picksplit` sie in mehr als einen Knoten klassifizieren. Gelingt dies nicht, erzeugt der SP-GiST-Kern ein `allTheSame`-Tupel; siehe [Abschnitt 65.3.4.3](65_Eingebaute_Index_Access_Methods.md#65343-allthesameinterne-tupel).

- `inner_consistent`
  - Gibt die Knoten beziehungsweise Zweige zurück, denen während der Suche im Baum gefolgt werden soll. Die Eingabe enthält Scan Keys, optionale Ordnungsoperatoren, rekonstruierte Werte, Traversal-Werte, Ebeneninformationen, Präfixe, Knotenlabels und das Flag `allTheSame`.
  - Die Funktion muss die zu besuchenden Knoten, mögliche Level-Inkremente, rekonstruierte Werte, Traversal-Werte und Distanzen für geordnete Suche bereitstellen.

- `leaf_consistent`
  - Gibt `true` zurück, wenn ein Leaf-Tupel die Abfrage erfüllt. Bei `returnData = true` muss `leafValue` auf den ursprünglich indizierten Wert gesetzt werden. `recheck` zeigt an, dass der Operator gegen das tatsächliche Heap-Tupel erneut geprüft werden muss. Bei geordneter Suche kann `distances` gesetzt werden; `recheckDistances` bedeutet, dass der Executor die exakten Distanzen nach dem Holen des Heap-Tupels berechnen muss.

- `compress`
  - Optional. Konvertiert ein Datenelement in ein Format, das zur physischen Speicherung in einem Leaf-Tupel geeignet ist. Es akzeptiert einen Wert vom Typ `spgConfigIn.attType` und gibt einen Wert vom Typ `spgConfigOut.leafType` zurück. Der Ausgabewert darf keinen out-of-line-TOAST-Zeiger enthalten.

- `options`
  - Optional. Definiert benutzersichtbare Parameter, die das Verhalten der Operatorklasse steuern. Wie bei GiST erfolgt dies über `local_relopts`.

Alle SP-GiST-Support-Methoden werden normalerweise in einem kurzlebigen Speicher-Kontext aufgerufen. Nach der Verarbeitung jedes Tupels wird `CurrentMemoryContext` zurückgesetzt. Wenn die indizierte Spalte einen kollatierbaren Datentyp besitzt, wird die Indexkollation mit `PG_GET_COLLATION()` an alle Support-Methoden übergeben.

### 65.3.4. Implementierung

Dieser Abschnitt behandelt Implementierungsdetails und Tricks, die für Implementierer von SP-GiST-Operatorklassen nützlich sind.

#### 65.3.4.1. SP-GiST-Grenzen

Einzelne Leaf-Tupel und interne Tupel müssen auf eine einzelne Indexseite passen, standardmäßig 8 kB. Beim Indizieren von Werten variabler Länge können lange Werte daher nur mit Methoden wie Radix Trees unterstützt werden, bei denen jede Baumebene ein Präfix enthält, das kurz genug für eine Seite ist, und die letzte Leaf-Ebene ein ebenfalls seitenpassendes Suffix enthält. Eine Operatorklasse sollte `longValuesOK` nur dann auf `true` setzen, wenn sie dies sicherstellen kann.

Ebenso ist die Operatorklasse dafür verantwortlich, dass interne Tupel nicht zu groß für eine Indexseite werden. Das begrenzt sowohl die Anzahl der Kindknoten in einem internen Tupel als auch die maximale Größe eines Präfixwerts.

Wenn ein Knoten eines internen Tupels auf eine Menge von Leaf-Tupeln zeigt, müssen diese Tupel alle auf derselben Indexseite liegen. Wird diese Menge zu groß, wird ein Split durchgeführt und ein zwischengeschaltetes internes Tupel eingefügt. Damit dies das Problem löst, muss das neue interne Tupel die Leaf-Werte in mehr als eine Knotengruppe aufteilen.

#### 65.3.4.2. SP-GiST ohne Knotenlabels

Einige Baumalgorithmen verwenden für jedes interne Tupel eine feste Knotengruppe. In einem Quadtree gibt es zum Beispiel immer genau vier Knoten für die vier Quadranten um den Mittelpunkt des internen Tupels. In solchen Fällen arbeitet der Code typischerweise mit Knotennummern, und explizite Knotenlabels sind nicht nötig. Um Knotenlabels zu unterdrücken und Speicherplatz zu sparen, kann `picksplit` `NULL` für das Array `nodeLabels` zurückgeben; entsprechend kann `choose` bei einer `spgSplitTuple`-Aktion `NULL` für `prefixNodeLabels` zurückgeben.

Bei einem internen Tupel ohne Labels ist es ein Fehler, wenn `choose` `spgAddNode` zurückgibt, da die Knotengruppe in solchen Fällen fest sein soll.

#### 65.3.4.3. `allTheSame`-interne Tupel

Der SP-GiST-Kern kann das Ergebnis der `picksplit`-Funktion überschreiben, wenn `picksplit` die übergebenen Leaf-Werte nicht in mindestens zwei Knotenkategorien aufteilt. In diesem Fall wird das neue interne Tupel mit mehreren Knoten erzeugt, die jeweils dasselbe Label haben, falls Labels vorhanden sind, und die Leaf-Werte werden zufällig auf diese äquivalenten Knoten verteilt. Das Flag `allTheSame` wird gesetzt, um `choose` und `inner_consistent` zu warnen, dass das Tupel nicht die erwartete Knotengruppe besitzt.

Bei einem `allTheSame`-Tupel bedeutet ein `choose`-Ergebnis `spgMatchNode`, dass der neue Wert einem beliebigen äquivalenten Knoten zugeordnet werden kann. Der Kerncode ignoriert den gelieferten Wert `nodeN` und steigt zufällig in einen der Knoten ab, um den Baum balanciert zu halten. `spgAddNode` ist hier ein Fehler; wenn der einzufügende Wert nicht zu den vorhandenen Knoten passt, muss `spgSplitTuple` verwendet werden.

### 65.3.5. Beispiele

Die PostgreSQL-Quelldistribution enthält mehrere Beispiele für SP-GiST-Indexoperatorklassen. Code dazu befindet sich in `src/backend/access/spgist/` und `src/backend/utils/adt/`.

## 65.4. GIN-Indizes

### 65.4.1. Einführung

GIN steht für Generalized Inverted Index. GIN ist für Fälle gedacht, in denen die zu indizierenden Items zusammengesetzte Werte sind und Abfragen nach Elementwerten suchen müssen, die innerhalb dieser zusammengesetzten Items vorkommen. Items könnten zum Beispiel Dokumente sein, und Abfragen könnten nach Dokumenten suchen, die bestimmte Wörter enthalten.

Das Wort Item bezeichnet einen zusammengesetzten Wert, der indiziert werden soll. Das Wort Schlüssel bezeichnet einen Elementwert. GIN speichert und sucht immer Schlüssel, nicht Item-Werte selbst.

Ein GIN-Index speichert eine Menge von Paaren `(key, posting list)`, wobei eine Posting List eine Menge von Zeilen-IDs ist, in denen der Schlüssel vorkommt. Dieselbe Zeilen-ID kann in mehreren Posting Lists erscheinen, weil ein Item mehr als einen Schlüssel enthalten kann. Jeder Schlüsselwert wird nur einmal gespeichert, weshalb ein GIN-Index sehr kompakt ist, wenn derselbe Schlüssel häufig vorkommt.

GIN ist generalisiert, weil der GIN-Zugriffsmethodencode die konkreten Operationen, die er beschleunigt, nicht kennen muss. Stattdessen verwendet er benutzerdefinierte Strategien für bestimmte Datentypen. Die Strategie definiert, wie Schlüssel aus indizierten Items und Abfragebedingungen extrahiert werden und wie bestimmt wird, ob eine Zeile, die einige Schlüsselwerte einer Abfrage enthält, die Abfrage tatsächlich erfüllt.

```text
http://www.sai.msu.su/~megera/wiki/Gin
```

### 65.4.2. Eingebaute Operatorklassen

Die PostgreSQL-Kerndistribution enthält die folgenden GIN-Operatorklassen:

| Operatorklasse | Indexierbare Operatoren |
| --- | --- |
| `array_ops` | `&&`, `@>`, `<@`, `=` für Arrays |
| `jsonb_ops` | `@>`, `@?`, `@@`, `?`, `?|`, `?&` für `jsonb` |
| `jsonb_path_ops` | `@>`, `@?`, `@@` für `jsonb`/`jsonpath` |
| `tsvector_ops` | `@@ (tsvector, tsquery)` |

Von den beiden Operatorklassen für `jsonb` ist `jsonb_ops` die Standardklasse. `jsonb_path_ops` unterstützt weniger Operatoren, bietet für diese Operatoren aber bessere Performance. Einzelheiten finden Sie in Abschnitt 8.14.4.

### 65.4.3. Erweiterbarkeit

Die GIN-Schnittstelle besitzt ein hohes Abstraktionsniveau. Der Implementierer der Zugriffsmethode muss nur die Semantik des Datentyps implementieren, auf den zugegriffen wird. Die GIN-Schicht kümmert sich selbst um Nebenläufigkeit, Logging und Suche in der Baumstruktur.

Eine GIN-Operatorklasse muss einige benutzerdefinierte Methoden implementieren, die das Verhalten von Schlüsseln im Baum und die Beziehungen zwischen Schlüsseln, indizierten Items und indexierbaren Abfragen definieren. GIN verbindet Erweiterbarkeit mit Allgemeinheit, Code-Wiederverwendung und einer klaren Schnittstelle.

Eine GIN-Operatorklasse muss zwei Methoden bereitstellen:

- `extractValue(Datum itemValue, int32 *nkeys, bool **nullFlags)`
  - Gibt für ein zu indizierendes Item ein mit `palloc` alloziertes Array von Schlüsseln zurück. Die Anzahl zurückgegebener Schlüssel muss in `*nkeys` gespeichert werden. Wenn Schlüssel null sein können, muss zusätzlich ein Array von `*nkeys` booleschen Feldern alloziert, seine Adresse in `*nullFlags` gespeichert und die Flags passend gesetzt werden. Wenn alle Schlüssel nicht-null sind, kann `*nullFlags` `NULL` bleiben. Der Rückgabewert kann `NULL` sein, wenn das Item keine Schlüssel enthält.

- `extractQuery(Datum query, int32 *nkeys, StrategyNumber n, bool **pmatch, Pointer **extra_data, bool **nullFlags, int32 *searchMode)`
  - Gibt für einen Abfragewert ein mit `palloc` alloziertes Array von Schlüsseln zurück. `query` ist der Wert auf der rechten Seite eines indexierbaren Operators, dessen linke Seite die indizierte Spalte ist. `n` ist die Strategienummer des Operators innerhalb der Operatorklasse. Die Anzahl der zurückgegebenen Schlüssel wird in `*nkeys` gespeichert.
  - `searchMode` erlaubt `extractQuery`, Details der Suche festzulegen. `GIN_SEARCH_MODE_DEFAULT` betrachtet nur Items als Kandidaten, die mindestens einen zurückgegebenen Schlüssel enthalten. `GIN_SEARCH_MODE_INCLUDE_EMPTY` betrachtet zusätzlich Items ohne Schlüssel als Kandidaten. `GIN_SEARCH_MODE_ALL` betrachtet alle nicht-null Items im Index als Kandidaten und ist deutlich langsamer.
  - `pmatch` wird verwendet, wenn Partial Match unterstützt wird. `extra_data` erlaubt zusätzliche Daten an `consistent` und `comparePartial` zu übergeben.

Eine Operatorklasse muss außerdem prüfen können, ob ein indiziertes Item zur Abfrage passt. Dafür gibt es zwei Varianten: die boolesche Funktion `consistent` und die ternäre Funktion `triConsistent`. `triConsistent` deckt die Funktionalität beider ab, sodass sie allein ausreicht. Wenn die boolesche Variante deutlich günstiger zu berechnen ist, kann es vorteilhaft sein, beide bereitzustellen.

- `consistent`
  - Gibt `true` zurück, wenn ein indiziertes Item den Abfrageoperator mit Strategienummer `n` erfüllt oder möglicherweise erfüllt. Die Funktion kennt den ursprünglichen Item-Wert nicht direkt, sondern nur, welche aus der Abfrage extrahierten Schlüssel in einem gegebenen indizierten Item vorkommen. Das Array `check` enthält für jeden Abfrageschlüssel, ob er im Item vorhanden ist. Bei Erfolg setzt die Funktion `*recheck`, um anzugeben, ob das Heap-Tupel erneut gegen den Abfrageoperator geprüft werden muss.

- `triConsistent`
  - Arbeitet ähnlich wie `consistent`, verwendet aber pro Schlüssel die Werte `GIN_TRUE`, `GIN_FALSE` und `GIN_MAYBE`. Die Funktion soll nur dann `GIN_TRUE` oder `GIN_FALSE` zurückgeben, wenn dies unabhängig von unbekannten `GIN_MAYBE`-Schlüsseln sicher ist; andernfalls muss sie `GIN_MAYBE` zurückgeben.

Zusätzlich muss GIN Schlüsselwerte sortieren können. Eine Operatorklasse kann dazu eine Vergleichsmethode definieren:

- `compare(Datum a, Datum b)`
  - Vergleicht zwei Schlüssel und gibt einen Integer kleiner null, gleich null oder größer null zurück. Null-Schlüssel werden nie an diese Funktion übergeben. Wenn keine Vergleichsfunktion angegeben ist, sucht GIN die Standard-B-Tree-Operatorklasse für den Schlüsseltyp und verwendet deren Vergleichsfunktion.

Optional kann eine GIN-Operatorklasse weitere Methoden bereitstellen:

- `comparePartial(Datum partial_key, Datum key, StrategyNumber n, Pointer extra_data)`
  - Vergleicht einen Partial-Match-Abfrageschlüssel mit einem Indexschlüssel. Ein negatives Ergebnis bedeutet, dass der Indexschlüssel nicht passt, der Scan aber fortgesetzt werden soll; null bedeutet Übereinstimmung; ein positives Ergebnis bedeutet, dass der Scan beendet werden kann.

- `options(local_relopts *relopts)`
  - Definiert benutzersichtbare Parameter für das Verhalten der Operatorklasse. Die Optionen können über `PG_HAS_OPCLASS_OPTIONS()` und `PG_GET_OPCLASS_OPTIONS()` abgerufen werden.

Um Partial-Match-Abfragen zu unterstützen, muss eine Operatorklasse `comparePartial` bereitstellen, und `extractQuery` muss bei einer Partial-Match-Abfrage den Parameter `pmatch` setzen.

### 65.4.4. Implementierung

Intern enthält ein GIN-Index einen B-Tree-Index über Schlüsseln. Jeder Schlüssel ist ein Element eines oder mehrerer indizierter Items, etwa ein Arrayelement. Jedes Tupel auf einer Leaf-Seite enthält entweder einen Zeiger auf einen B-Tree von Heap-Zeigern, einen sogenannten Posting Tree, oder eine einfache Liste von Heap-Zeigern, eine Posting List, wenn diese klein genug ist, um zusammen mit dem Schlüsselwert in ein einzelnes Indextupel zu passen.

Seit PostgreSQL 9.1 können Null-Schlüsselwerte in den Index aufgenommen werden. Außerdem werden Platzhalter-Nulls für indizierte Items aufgenommen, die null sind oder laut `extractValue` keine Schlüssel enthalten. Dadurch können Suchen leere Items finden, wenn ihre Semantik das verlangt.

Mehrspalten-GIN-Indizes werden implementiert, indem ein einzelner B-Tree über zusammengesetzten Werten `(column number, key value)` gebaut wird. Die Schlüsselwerte verschiedener Spalten können unterschiedliche Typen haben.

#### 65.4.4.1. GIN-Fast-Update-Technik

Das Aktualisieren eines GIN-Index ist tendenziell langsam, weil das Einfügen oder Aktualisieren einer Heap-Zeile viele Einfügungen in den Index verursachen kann, eine für jeden aus dem indizierten Item extrahierten Schlüssel. GIN kann einen großen Teil dieser Arbeit aufschieben, indem neue Tupel in eine temporäre, unsortierte Liste ausstehender Einträge eingefügt werden. Wenn die Tabelle gevacuumt oder autoanalysiert wird, wenn `gin_clean_pending_list` aufgerufen wird oder wenn die Pending List größer als `gin_pending_list_limit` wird, werden die Einträge mit denselben Bulk-Insert-Techniken wie beim initialen Indexbau in die Hauptdatenstruktur von GIN verschoben.

Der Hauptnachteil besteht darin, dass Suchen zusätzlich zum regulären Index auch die Pending List scannen müssen. Eine große Pending List verlangsamt Suchen deutlich. Außerdem kann ein Update, das die Liste »zu groß« werden lässt, einen sofortigen Bereinigungslauf auslösen und dadurch deutlich langsamer sein als andere Updates. Korrekte Verwendung von Autovacuum kann beide Probleme minimieren.

Wenn konsistente Antwortzeit wichtiger ist als Update-Geschwindigkeit, kann die Verwendung ausstehender Einträge durch Abschalten des Storage-Parameters `fastupdate` für einen GIN-Index deaktiviert werden.

#### 65.4.4.2. Partial-Match-Algorithmus

GIN kann Partial-Match-Abfragen unterstützen, bei denen die Abfrage keine exakte Übereinstimmung für einen oder mehrere Schlüssel bestimmt, aber die möglichen Treffer in einem hinreichend engen Bereich von Schlüsselwerten liegen. `extractQuery` gibt dann statt eines exakt zu vergleichenden Schlüsselwerts einen Schlüsselwert zurück, der die untere Grenze des zu durchsuchenden Bereichs darstellt, und setzt das Flag `pmatch` auf `true`. Der Schlüsselbereich wird anschließend mit `comparePartial` gescannt.

### 65.4.5. GIN-Tipps und Tricks

- Erzeugen statt Einfügen
  - Das Einfügen in einen GIN-Index kann langsam sein, weil pro Item wahrscheinlich viele Schlüssel eingefügt werden. Bei Bulk Inserts in eine Tabelle ist es daher ratsam, den GIN-Index zu löschen und ihn nach Abschluss der Bulk Inserts neu zu erzeugen. Wenn `fastupdate` aktiviert ist, ist die Strafe geringer, aber bei sehr großen Updates kann Löschen und Neuerzeugen weiterhin besser sein.

- `maintenance_work_mem`
  - Die Build-Zeit eines GIN-Index reagiert sehr empfindlich auf die Einstellung `maintenance_work_mem`; bei der Indexerzeugung sollte nicht am Arbeitsspeicher gespart werden.

- `gin_pending_list_limit`
  - Bei einer Serie von Einfügungen in einen vorhandenen GIN-Index mit aktiviertem `fastupdate` bereinigt das System die Pending-Entry-Liste, sobald sie größer als `gin_pending_list_limit` wird. Um Schwankungen der beobachteten Antwortzeit zu vermeiden, sollte die Bereinigung im Hintergrund, also über Autovacuum, erfolgen.
  - `gin_pending_list_limit` kann für einzelne GIN-Indizes über Storage-Parameter überschrieben werden.

- `gin_fuzzy_search_limit`
  - GIN besitzt eine konfigurierbare weiche Obergrenze für die Anzahl zurückgegebener Zeilen: den Konfigurationsparameter `gin_fuzzy_search_limit`. Standardmäßig ist er 0, also unbegrenzt. Bei einem Nicht-Null-Limit wird eine zufällig gewählte Teilmenge des gesamten Ergebnisses zurückgegeben. Erfahrungswerte im Bereich einiger Tausend, etwa 5000 bis 20000, funktionieren gut.

### 65.4.6. Einschränkungen

GIN nimmt an, dass indexierbare Operatoren strikt sind. Das bedeutet, dass `extractValue` bei einem Null-Item-Wert gar nicht aufgerufen wird; stattdessen wird automatisch ein Platzhalter-Indexeintrag erzeugt. `extractQuery` wird bei einem Null-Abfragewert ebenfalls nicht aufgerufen; die Abfrage gilt stattdessen als unerfüllbar. Null-Schlüsselwerte innerhalb eines nicht-null zusammengesetzten Items oder Abfragewerts werden jedoch unterstützt.

### 65.4.7. Beispiele

Die PostgreSQL-Kerndistribution enthält die oben gezeigten GIN-Operatorklassen. Zusätzlich enthalten diese `contrib`-Module GIN-Operatorklassen:

- `btree_gin`: B-Tree-ähnliche Funktionalität für mehrere Datentypen
- `hstore`: Modul zum Speichern von Schlüssel/Wert-Paaren
- `intarray`: erweiterte Unterstützung für `int[]`
- `pg_trgm`: Textähnlichkeit mit Trigramm-Matching

## 65.5. BRIN-Indizes

### 65.5.1. Einführung

BRIN steht für Block Range Index. BRIN ist für sehr große Tabellen gedacht, in denen bestimmte Spalten eine natürliche Korrelation mit ihrer physischen Position innerhalb der Tabelle besitzen.

BRIN arbeitet mit Blockbereichen oder Seitenbereichen. Ein Blockbereich ist eine Gruppe physisch benachbarter Seiten in der Tabelle; für jeden Blockbereich speichert der Index Zusammenfassungsinformationen. Eine Tabelle mit Verkaufsaufträgen könnte etwa eine Datumsspalte besitzen, bei der frühere Aufträge meist auch früher in der Tabelle erscheinen. Eine Tabelle mit Postleitzahlen könnte die Codes einer Stadt natürlicherweise gruppiert enthalten.

BRIN-Indizes können Abfragen über gewöhnliche Bitmap-Indexscans bedienen und geben alle Tupel auf allen Seiten innerhalb jedes Bereichs zurück, dessen gespeicherte Zusammenfassung mit den Abfragebedingungen konsistent ist. Der Query Executor prüft diese Tupel erneut und verwirft diejenigen, die die Bedingungen nicht erfüllen. BRIN-Indizes sind also lossy. Da ein BRIN-Index sehr klein ist, verursacht sein Scan nur wenig Zusatzaufwand gegenüber einem sequenziellen Scan, kann aber große Tabellenteile überspringen, die bekanntermaßen keine passenden Tupel enthalten.

Welche Daten ein BRIN-Index speichert und welche Abfragen er bedienen kann, hängt von der für jede Indexspalte gewählten Operatorklasse ab. Datentypen mit linearer Sortierordnung können zum Beispiel Operatorklassen verwenden, die Minimal- und Maximalwert innerhalb jedes Blockbereichs speichern. Geometrische Typen könnten die Bounding Box aller Objekte im Blockbereich speichern.

Die Größe des Blockbereichs wird bei der Indexerzeugung durch den Storage-Parameter `pages_per_range` bestimmt. Die Anzahl der Indexeinträge entspricht der Anzahl der Relationsseiten geteilt durch `pages_per_range`. Ein kleinerer Wert erzeugt einen größeren Index, speichert aber präzisere Zusammenfassungsdaten und kann mehr Datenblöcke während eines Indexscans überspringen.

#### 65.5.1.1. Indexwartung

Bei der Erzeugung werden alle vorhandenen Heap-Seiten gescannt und für jeden Bereich, einschließlich des möglicherweise unvollständigen letzten Bereichs, ein Summary-Indextupel erzeugt. Wenn neue Seiten mit Daten gefüllt werden, aktualisieren bereits zusammengefasste Seitenbereiche ihre Zusammenfassungsinformationen mit Daten aus den neuen Tupeln. Wenn eine neue Seite erzeugt wird, die nicht in den zuletzt zusammengefassten Bereich fällt, erhält der zugehörige Bereich nicht automatisch ein Summary-Tupel. Er bleibt unzusammengefasst, bis später ein Zusammenfassungslauf angestoßen wird.

Die initiale Zusammenfassung eines Seitenbereichs kann auf mehrere Arten ausgelöst werden: durch manuelles `VACUUM` oder Autovacuum, durch den Storage-Parameter `autosummarize` des Index, wenn dieser aktiviert ist, oder durch die folgenden Funktionen:

- `brin_summarize_new_values(regclass)`: fasst alle unzusammengefassten Bereiche zusammen.
- `brin_summarize_range(regclass, bigint)`: fasst nur den Bereich zusammen, der die angegebene Seite enthält, falls er noch unzusammengefasst ist.

Während diese Funktionen laufen, wird `search_path` vorübergehend auf `pg_catalog, pg_temp` geändert.

Wenn Autosummarization aktiviert ist, wird eine Anforderung an Autovacuum gesendet, sobald eine Einfügung für das erste Item der ersten Seite des nächsten Blockbereichs erkannt wird. Ist die Anforderungswarteschlange voll, wird die Anforderung nicht aufgezeichnet, und eine Meldung wird ins Serverlog geschrieben:

```text
LOG: request for BRIN range summarization for index "brin_wi_idx"
 page 128 was not recorded
```

Umgekehrt kann ein Bereich mit `brin_desummarize_range(regclass, bigint)` wieder de-zusammengefasst werden. Das ist nützlich, wenn das Indextupel wegen geänderter vorhandener Werte keine gute Repräsentation mehr ist. Einzelheiten finden Sie in Abschnitt 9.28.8.

### 65.5.2. Eingebaute Operatorklassen

Die PostgreSQL-Kerndistribution enthält BRIN-Operatorklassen der Typen `minmax`, `minmax-multi`, `inclusion` und `bloom`.

- `minmax`-Operatorklassen speichern Minimal- und Maximalwerte der indizierten Spalte innerhalb des Bereichs.
- `inclusion`-Operatorklassen speichern einen Wert, der die Werte der indizierten Spalte innerhalb des Bereichs einschließt.
- `bloom`-Operatorklassen erzeugen einen Bloom-Filter für alle Werte der indizierten Spalte im Bereich.
- `minmax-multi`-Operatorklassen speichern mehrere Minimal- und Maximalwerte, die die im Bereich auftretenden Werte repräsentieren.

Eingebaute BRIN-Operatorklassen existieren unter anderem für `bit`, `box`, `bpchar`, `bytea`, `"char"`, `date`, `float4`, `float8`, `inet`, `int2`, `int4`, `int8`, `interval`, `macaddr`, `macaddr8`, `name`, `numeric`, `oid`, `pg_lsn`, Range-Typen, `text`, `tid`, `timestamp`, `timestamptz`, `time`, `timetz`, `uuid` und `varbit`, jeweils mit den passenden `bloom`-, `minmax`-, `minmax_multi`- oder `inclusion`-Varianten.

#### 65.5.2.1. Operatorklassenparameter

Einige eingebaute Operatorklassen erlauben Parameter, die das Verhalten der Operatorklasse beeinflussen. Jede Operatorklasse hat ihre eigene Menge zulässiger Parameter. Nur die `bloom`- und `minmax-multi`-Operatorklassen erlauben Parameter.

`bloom`-Operatorklassen akzeptieren:

- `n_distinct_per_range`
  - Definiert die geschätzte Anzahl unterschiedlicher Nicht-Null-Werte im Blockbereich. BRIN-Bloom-Indizes verwenden diesen Wert zur Dimensionierung des Bloom-Filters. Positive Werte bedeuten, dass jeder Blockbereich genau diese Anzahl unterschiedlicher Nicht-Null-Werte enthält. Negative Werte, größer oder gleich `-1`, bedeuten, dass die Anzahl unterschiedlicher Nicht-Null-Werte linear mit der maximal möglichen Anzahl von Tupeln im Blockbereich wächst. Der Standardwert ist `-0.1`, und die Mindestzahl unterschiedlicher Nicht-Null-Werte ist 16.

- `false_positive_rate`
  - Definiert die gewünschte False-Positive-Rate zur Dimensionierung des Bloom-Filters. Werte müssen zwischen `0.0001` und `0.25` liegen. Der Standardwert ist `0.01`, also 1 Prozent.

`minmax-multi`-Operatorklassen akzeptieren:

- `values_per_range`
  - Definiert die maximale Anzahl von Werten, die von BRIN-Minmax-Indizes zur Zusammenfassung eines Blockbereichs gespeichert werden. Jeder Wert kann entweder einen Punkt oder eine Grenze eines Intervalls repräsentieren. Werte müssen zwischen 8 und 256 liegen; der Standardwert ist 32.

### 65.5.3. Erweiterbarkeit

Die BRIN-Schnittstelle besitzt ein hohes Abstraktionsniveau. Der Entwickler muss nur die Semantik der Zusammenfassungswerte implementieren, die im Index gespeichert werden, und beschreiben, wie sie mit Scan Keys interagieren. BRIN verbindet Erweiterbarkeit mit Allgemeinheit, Wiederverwendung von Code und einer klaren Schnittstelle.

Eine BRIN-Operatorklasse muss vier Methoden bereitstellen:

- `BrinOpcInfo *opcInfo(Oid type_oid)`
  - Gibt interne Informationen über die Summary-Daten der indizierten Spalten zurück. Der Rückgabewert muss auf eine mit `palloc` allozierte `BrinOpcInfo`-Struktur zeigen. `BrinOpcInfo.oi_opaque` kann von Operatorklassenroutinen verwendet werden, um während eines Indexscans Informationen zwischen Support-Funktionen weiterzugeben.

- `bool consistent(BrinDesc *bdesc, BrinValues *column, ScanKey *keys, int nkeys)`
  - Gibt zurück, ob alle `ScanKey`-Einträge mit den gegebenen indizierten Werten für einen Bereich konsistent sind. Die zu verwendende Attributnummer wird als Teil des Scan Keys übergeben. Mehrere Scan Keys für dasselbe Attribut können gleichzeitig übergeben werden; ihre Anzahl bestimmt `nkeys`.

- `bool addValue(BrinDesc *bdesc, BrinValues *column, Datum newval, bool isnull)`
  - Modifiziert bei einem Indextupel und einem indizierten Wert das angegebene Attribut des Tupels so, dass es zusätzlich den neuen Wert repräsentiert. Wenn eine Änderung vorgenommen wurde, wird `true` zurückgegeben.

- `bool unionTuples(BrinDesc *bdesc, BrinValues *a, BrinValues *b)`
  - Konsolidiert zwei Indextupel. Die Funktion modifiziert das angegebene Attribut des ersten Tupels so, dass es beide Tupel repräsentiert. Das zweite Tupel wird nicht geändert.

Optional kann eine BRIN-Operatorklasse bereitstellen:

- `void options(local_relopts *relopts)`
  - Definiert benutzersichtbare Parameter, die das Verhalten der Operatorklasse steuern. Die Optionen werden über `local_relopts` beschrieben und können mit `PG_HAS_OPCLASS_OPTIONS()` und `PG_GET_OPCLASS_OPTIONS()` gelesen werden.

Die Kerndistribution enthält Support für vier Arten von Operatorklassen: `minmax`, `minmax-multi`, `inclusion` und `bloom`. Für eingebaute Datentypen werden passende Operatorklassendefinitionen mitgeliefert. Zusätzliche Operatorklassen können für andere Datentypen mit äquivalenten Definitionen angelegt werden, ohne Quellcode zu schreiben, sofern passende Katalogeinträge deklariert werden.

Für einen Datentyp mit totaler Ordnung können die `minmax`-Support-Funktionen mit den entsprechenden Operatoren verwendet werden:

| Operatorklassenmitglied | Objekt |
| --- | --- |
| Support Function 1 | interne Funktion `brin_minmax_opcinfo()` |
| Support Function 2 | interne Funktion `brin_minmax_add_value()` |
| Support Function 3 | interne Funktion `brin_minmax_consistent()` |
| Support Function 4 | interne Funktion `brin_minmax_union()` |
| Operator Strategy 1 | Operator kleiner als |
| Operator Strategy 2 | Operator kleiner oder gleich |
| Operator Strategy 3 | Operator gleich |
| Operator Strategy 4 | Operator größer oder gleich |
| Operator Strategy 5 | Operator größer als |

Für komplexe Datentypen, deren Werte in einem anderen Typ enthalten sein können, können die `inclusion`-Support-Funktionen verwendet werden. Sie benötigen zusätzlich eine Funktion zum Zusammenführen zweier Elemente und können optionale Funktionen für Zusammenführbarkeit, Enthaltensein und Leerheit bereitstellen. Die Strategien decken räumliche Relationen wie links von, rechts von, überlappt, enthält, ist enthalten in, oberhalb/unterhalb sowie Gleichheit ab.

Für Datentypen mit Gleichheitsoperator und Hash-Unterstützung können die `bloom`-Support-Prozeduren verwendet werden. Die zentrale benutzerdefinierte Funktion ist Support Procedure 11, die einen Hash eines Elements berechnet; die einzige Operatorstrategie ist Gleichheit.

`minmax-multi` erweitert `minmax`, indem ein Blockbereich nicht in ein einziges zusammenhängendes Intervall zusammengefasst wird, sondern in mehrere kleinere Intervalle. Dadurch werden Ausreißerwerte besser behandelt. Support Procedure 11 berechnet die Distanz zwischen zwei Werten, also die Länge eines Bereichs.

Sowohl `minmax`- als auch `inclusion`-Operatorklassen unterstützen typübergreifende Operatoren, allerdings werden die Abhängigkeiten dabei komplexer. Beispiele sind `float4_minmax_ops` für `minmax` und `box_inclusion_ops` für `inclusion`.

## 65.6. Hash-Indizes

### 65.6.1. Überblick

PostgreSQL enthält eine Implementierung persistenter Hash-Indizes auf Platte, die vollständig crash-recoverable sind. Jeder Datentyp kann durch einen Hash-Index indiziert werden, einschließlich Datentypen ohne wohldefinierte lineare Ordnung. Hash-Indizes speichern nur den Hashwert der indizierten Daten; daher gibt es keine Einschränkungen für die Größe der indizierten Datenspalte.

Hash-Indizes unterstützen nur einspaltige Indizes und keine Eindeutigkeitsprüfung. Sie unterstützen nur den Operator `=`, sodass `WHERE`-Klauseln mit Bereichsoperationen Hash-Indizes nicht nutzen können.

Jedes Hash-Indextupel speichert nur den 4-Byte-Hashwert, nicht den tatsächlichen Spaltenwert. Dadurch können Hash-Indizes beim Indizieren längerer Datenelemente wie UUIDs oder URLs deutlich kleiner sein als B-Trees. Das Fehlen des Spaltenwerts macht allerdings alle Hash-Indexscans lossy. Hash-Indizes können an Bitmap-Indexscans und Rückwärtsscans teilnehmen.

Hash-Indizes sind besonders für `SELECT`- und update-lastige Workloads optimiert, die Gleichheitsscans auf größeren Tabellen verwenden. In einem B-Tree-Index müssen Suchen durch den Baum bis zur Leaf-Seite absteigen. Bei Tabellen mit Millionen Zeilen kann dieser Abstieg die Zugriffszeit erhöhen. Das Äquivalent einer Leaf-Seite in einem Hash-Index heißt Bucket-Seite. Ein Hash-Index kann direkt auf Bucket-Seiten zugreifen und so die Indexzugriffszeit bei größeren Tabellen potenziell verringern. Diese Reduktion logischer I/O wird besonders deutlich, wenn Index oder Daten größer als `shared_buffers` beziehungsweise RAM sind.

Hash-Indizes sind darauf ausgelegt, mit ungleichmäßigen Hashwertverteilungen umzugehen. Direkter Zugriff auf Bucket-Seiten funktioniert gut, wenn Hashwerte gleichmäßig verteilt sind. Wenn Einfügungen dazu führen, dass eine Bucket-Seite voll wird, werden zusätzliche Overflow-Seiten an diese Bucket-Seite gekettet. Bei Abfragen müssen alle Overflow-Seiten eines Buckets gescannt werden. Ein unausgeglichener Hash-Index kann daher bei manchen Daten schlechter sein als ein B-Tree, gemessen an der Anzahl benötigter Blockzugriffe.

Daher eignen sich Hash-Indizes am besten für eindeutige, nahezu eindeutige Daten oder Daten mit niedriger Anzahl von Zeilen pro Hash-Bucket. Eine Möglichkeit, Probleme zu vermeiden, besteht darin, stark nicht-eindeutige Werte durch eine partielle Indexbedingung aus dem Index auszuschließen, was aber nicht immer geeignet ist.

Wie B-Trees führen Hash-Indizes einfache Indextupel-Löschung aus. Dabei handelt es sich um eine zurückgestellte Wartungsoperation, die Indextupel löscht, deren `LP_DEAD`-Bit gesetzt ist. Wenn eine Einfügung keinen Platz auf einer Seite findet, versucht PostgreSQL, das Erzeugen einer neuen Overflow-Seite durch Entfernen toter Indextupel zu vermeiden. Dies kann nicht erfolgen, wenn die Seite zu diesem Zeitpunkt gepinnt ist. Die Löschung toter Indexzeiger erfolgt außerdem während `VACUUM`.

Wenn möglich, versucht `VACUUM`, Indextupel auf möglichst wenige Overflow-Seiten zusammenzuschieben und so die Overflow-Kette zu minimieren. Wird eine Overflow-Seite leer, kann sie für andere Buckets wiederverwendet werden; sie wird jedoch nie an das Betriebssystem zurückgegeben. Derzeit gibt es keine Möglichkeit, einen Hash-Index zu verkleinern, außer ihn mit `REINDEX` neu aufzubauen. Auch die Anzahl der Buckets kann nicht reduziert werden.

Hash-Indizes können die Anzahl der Bucket-Seiten erweitern, wenn die Anzahl indizierter Zeilen wächst. Die Abbildung von Hashschlüsseln auf Bucket-Nummern ist so gewählt, dass der Index inkrementell erweitert werden kann. Wenn ein neuer Bucket hinzugefügt wird, muss genau ein vorhandener Bucket gesplittet werden; einige seiner Tupel werden gemäß der aktualisierten Abbildung in den neuen Bucket übertragen. Diese Erweiterung geschieht im Vordergrund und kann die Ausführungszeit von Benutzereinfügungen erhöhen. Deshalb sind Hash-Indizes möglicherweise nicht für Tabellen mit schnell wachsender Zeilenzahl geeignet.

### 65.6.2. Implementierung

In einem Hash-Index gibt es vier Arten von Seiten: die Metaseite (Seite null) mit statisch allozierten Steuerinformationen, primäre Bucket-Seiten, Overflow-Seiten und Bitmap-Seiten, die freigegebene und wiederverwendbare Overflow-Seiten verfolgen. Für Adressierungszwecke gelten Bitmap-Seiten als Teilmenge der Overflow-Seiten.

Sowohl das Scannen des Index als auch das Einfügen von Tupeln erfordern, den Bucket zu finden, in dem ein gegebenes Tupel liegen sollte. Dafür werden Bucket-Anzahl, `highmask` und `lowmask` von der Metaseite benötigt. Aus Performance-Gründen wäre es jedoch unerwünscht, die Metaseite für jede Operation sperren und pinnen zu müssen. Stattdessen hält jedes Backend eine gecachte Kopie der Metaseite im Relcache-Eintrag. Diese liefert die korrekte Bucket-Abbildung, solange der Ziel-Bucket seit der letzten Cache-Aktualisierung nicht gesplittet wurde.

Primäre Bucket-Seiten und Overflow-Seiten werden unabhängig alloziert, weil ein Index relativ zu seiner Bucket-Anzahl mehr oder weniger Overflow-Seiten benötigen kann. Der Hash-Code verwendet interessante Adressierungsregeln, um eine variable Anzahl von Overflow-Seiten zu unterstützen, ohne primäre Bucket-Seiten nach ihrer Erzeugung verschieben zu müssen.

Jede in der indizierten Tabelle vorhandene Zeile wird im Hash-Index durch ein einzelnes Indextupel repräsentiert. Hash-Indextupel werden in Bucket-Seiten und, falls vorhanden, Overflow-Seiten gespeichert. Suchen werden beschleunigt, indem die Indexeinträge auf jeder Indexseite nach Hashcode sortiert gehalten werden, sodass innerhalb einer Indexseite binäre Suche möglich ist. Es gibt jedoch keine Annahme über die relative Ordnung von Hashcodes über verschiedene Indexseiten eines Buckets hinweg.

Die Bucket-Splitting-Algorithmen zur Erweiterung eines Hash-Index sind zu komplex, um hier im Detail beschrieben zu werden. Sie werden ausführlicher in `src/backend/access/hash/README` behandelt. Der Split-Algorithmus ist crash-sicher und kann neu gestartet werden, wenn er nicht erfolgreich abgeschlossen wurde.
