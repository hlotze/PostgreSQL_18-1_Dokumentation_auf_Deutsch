# 29. Logische Replikation

Logische Replikation ist eine Methode, Datenobjekte und ihre Änderungen anhand ihrer Replica Identity zu replizieren, üblicherweise anhand eines Primärschlüssels. Der Begriff logisch steht hier im Gegensatz zur physischen Replikation, die exakte Blockadressen und Byte-für-Byte-Replikation verwendet. PostgreSQL unterstützt beide Mechanismen gleichzeitig; siehe [Kapitel 26](26_Hochverfügbarkeit_Lastverteilung_und_Replikation.md). Logische Replikation erlaubt eine feingranulare Kontrolle sowohl über die Datenreplikation als auch über die Sicherheit.

Logische Replikation verwendet ein Publish-and-Subscribe-Modell, bei dem ein oder mehrere Subscriber eine oder mehrere Publications auf einem Publisher-Knoten abonnieren. Subscriber ziehen Daten aus den Publications, die sie abonniert haben, und können Daten anschließend erneut veröffentlichen, um kaskadierende Replikation oder komplexere Konfigurationen zu ermöglichen.

Wenn die logische Replikation einer Tabelle typischerweise startet, erstellt PostgreSQL einen Snapshot der Tabellendaten in der Publisher-Datenbank und kopiert ihn zum Subscriber. Danach werden Änderungen auf dem Publisher seit der ersten Kopie fortlaufend an den Subscriber gesendet. Der Subscriber wendet die Daten in derselben Reihenfolge wie der Publisher an, sodass transaktionale Konsistenz für Publications innerhalb einer einzelnen Subscription garantiert ist. Diese Methode der Datenreplikation wird manchmal als transaktionale Replikation bezeichnet.

Typische Anwendungsfälle für logische Replikation sind:

- Inkrementelle Änderungen in einer einzelnen Datenbank oder in einer Teilmenge einer Datenbank an Subscriber senden, sobald sie auftreten.
- Trigger für einzelne Änderungen auslösen, wenn sie beim Subscriber eintreffen.
- Mehrere Datenbanken in einer einzigen zusammenführen, zum Beispiel für Analysezwecke.
- Zwischen verschiedenen Hauptversionen von PostgreSQL replizieren.
- Zwischen PostgreSQL-Instanzen auf verschiedenen Plattformen replizieren, zum Beispiel von Linux nach Windows.
- Unterschiedlichen Benutzergruppen Zugriff auf replizierte Daten geben.
- Eine Teilmenge der Datenbank zwischen mehreren Datenbanken teilen.

Die Subscriber-Datenbank verhält sich wie jede andere PostgreSQL-Instanz und kann als Publisher für andere Datenbanken verwendet werden, indem eigene Publications definiert werden. Wenn der Subscriber von der Anwendung als schreibgeschützt behandelt wird, entstehen aus einer einzelnen Subscription keine Konflikte. Wenn dagegen entweder durch eine Anwendung oder durch andere Subscriber Schreibvorgänge auf denselben Tabellensatz erfolgen, können Konflikte entstehen.

## 29.1. Publication

Eine Publication kann auf jedem Primärserver der physischen Replikation definiert werden. Der Knoten, auf dem eine Publication definiert ist, wird als Publisher bezeichnet. Eine Publication ist eine Menge von Änderungen, die aus einer Tabelle oder einer Gruppe von Tabellen erzeugt wird, und kann auch als Change Set oder Replication Set beschrieben werden. Jede Publication existiert nur in einer Datenbank.

Publications unterscheiden sich von Schemata und beeinflussen nicht, wie auf die Tabelle zugegriffen wird. Jede Tabelle kann bei Bedarf zu mehreren Publications hinzugefügt werden. Publications können derzeit nur Tabellen und alle Tabellen in einem Schema enthalten. Objekte müssen explizit hinzugefügt werden, außer wenn eine Publication für `ALL TABLES` erstellt wird.

Publications können die von ihnen erzeugten Änderungen auf jede Kombination von `INSERT`, `UPDATE`, `DELETE` und `TRUNCATE` beschränken, ähnlich wie Trigger durch bestimmte Ereignistypen ausgelöst werden. Standardmäßig werden alle Operationstypen repliziert. Diese Publication-Spezifikationen gelten nur für DML-Operationen; sie beeinflussen die anfängliche Kopie zur Datensynchronisierung nicht. Zeilenfilter haben keine Wirkung für `TRUNCATE`; siehe [Abschnitt 29.4](29_Logische_Replikation.md#294-zeilenfilter).

Jede Publication kann mehrere Subscriber haben.

Eine Publication wird mit dem Befehl `CREATE PUBLICATION` erstellt und kann später mit den entsprechenden Befehlen geändert oder entfernt werden.

Einzelne Tabellen können mit `ALTER PUBLICATION` dynamisch hinzugefügt und entfernt werden. Sowohl `ADD TABLE` als auch `DROP TABLE` sind transaktional, sodass die Tabelle beim richtigen Snapshot mit der Replikation beginnt oder aufhört, sobald die Transaktion committet wurde.

### 29.1.1. Replica Identity

Für eine veröffentlichte Tabelle muss eine Replica Identity konfiguriert sein, damit `UPDATE`- und `DELETE`-Operationen repliziert werden können und die passenden Zeilen, die auf der Subscriber-Seite aktualisiert oder gelöscht werden sollen, identifiziert werden können.

Standardmäßig ist dies der Primärschlüssel, sofern einer vorhanden ist. Ein anderer eindeutiger Index kann unter bestimmten zusätzlichen Anforderungen ebenfalls als Replica Identity festgelegt werden. Wenn die Tabelle keinen geeigneten Schlüssel hat, kann sie auf Replica Identity `FULL` gesetzt werden; dann wird die gesamte Zeile zum Schlüssel. Wenn Replica Identity `FULL` angegeben ist, können auf der Subscriber-Seite Indizes zum Suchen der Zeilen verwendet werden. Kandidatenindizes müssen `btree` oder `hash` sein, dürfen nicht partiell sein, und das linke Indexfeld muss eine Spalte sein, kein Ausdruck, die auf die Spalte der veröffentlichten Tabelle verweist. Diese Einschränkungen für nicht eindeutige Indexeigenschaften folgen einigen der Einschränkungen, die für Primärschlüssel erzwungen werden. Wenn keine solchen geeigneten Indizes vorhanden sind, kann die Suche auf der Subscriber-Seite sehr ineffizient sein; daher sollte Replica Identity `FULL` nur als Fallback verwendet werden, wenn keine andere Lösung möglich ist.

Wenn auf der Publisher-Seite eine andere Replica Identity als `FULL` gesetzt ist, muss auf der Subscriber-Seite ebenfalls eine Replica Identity gesetzt sein, die dieselben oder weniger Spalten umfasst.

Tabellen mit einer Replica Identity, die als `NOTHING`, als `DEFAULT` ohne Primärschlüssel oder als `USING INDEX` mit einem gelöschten Index definiert ist, können keine `UPDATE`- oder `DELETE`-Operationen unterstützen, wenn sie in einer Publication enthalten sind, die diese Aktionen repliziert. Der Versuch solcher Operationen führt zu einem Fehler auf dem Publisher.

`INSERT`-Operationen können unabhängig von einer Replica Identity fortgesetzt werden.

Details zum Setzen der Replica Identity finden Sie unter `ALTER TABLE ... REPLICA IDENTITY`.

## 29.2. Subscription

Eine Subscription ist die Downstream-Seite der logischen Replikation. Der Knoten, auf dem eine Subscription definiert ist, wird als Subscriber bezeichnet. Eine Subscription definiert die Verbindung zu einer anderen Datenbank und die Menge der Publications, eine oder mehrere, die sie abonnieren möchte.

Die Subscriber-Datenbank verhält sich wie jede andere PostgreSQL-Instanz und kann als Publisher für andere Datenbanken verwendet werden, indem eigene Publications definiert werden.

Ein Subscriber-Knoten kann bei Bedarf mehrere Subscriptions haben. Es ist möglich, mehrere Subscriptions zwischen einem einzelnen Publisher-Subscriber-Paar zu definieren; in diesem Fall muss sorgfältig darauf geachtet werden, dass sich die abonnierten Publication-Objekte nicht überschneiden.

Jede Subscription empfängt Änderungen über einen Replikationsslot (siehe Abschnitt 26.2.6). Zusätzliche Replikationsslots können für die anfängliche Datensynchronisierung bereits vorhandener Tabellendaten erforderlich sein; diese werden am Ende der Datensynchronisierung gelöscht.

Eine Subscription der logischen Replikation kann als Standby für synchrone Replikation dienen (siehe Abschnitt 26.2.8). Der Standby-Name ist standardmäßig der Name der Subscription. Ein alternativer Name kann als `application_name` in den Verbindungsinformationen der Subscription angegeben werden.

Subscriptions werden von `pg_dump` ausgegeben, wenn der aktuelle Benutzer ein Superuser ist. Andernfalls wird eine Warnung geschrieben und Subscriptions werden übersprungen, weil Nicht-Superuser nicht alle Subscription-Informationen aus dem Katalog `pg_subscription` lesen können.

Die Subscription wird mit `CREATE SUBSCRIPTION` hinzugefügt und kann jederzeit mit dem Befehl `ALTER SUBSCRIPTION` angehalten oder fortgesetzt sowie mit `DROP SUBSCRIPTION` entfernt werden.

Wenn eine Subscription gelöscht und neu erstellt wird, gehen die Synchronisierungsinformationen verloren. Das bedeutet, dass die Daten anschließend erneut synchronisiert werden müssen.

Die Schemadefinitionen werden nicht repliziert, und die veröffentlichten Tabellen müssen auf dem Subscriber existieren. Nur reguläre Tabellen können Ziel der Replikation sein. Sie können zum Beispiel nicht in eine View replizieren.

Die Tabellen werden zwischen Publisher und Subscriber anhand des vollqualifizierten Tabellennamens abgeglichen. Replikation in anders benannte Tabellen auf dem Subscriber wird nicht unterstützt.

Spalten einer Tabelle werden ebenfalls anhand des Namens abgeglichen. Die Reihenfolge der Spalten in der Subscriber-Tabelle muss nicht mit der des Publishers übereinstimmen. Die Datentypen der Spalten müssen nicht übereinstimmen, solange die Textdarstellung der Daten in den Zieltyp konvertiert werden kann. Sie können zum Beispiel von einer Spalte des Typs `integer` in eine Spalte des Typs `bigint` replizieren. Die Zieltabelle kann auch zusätzliche Spalten haben, die von der veröffentlichten Tabelle nicht bereitgestellt werden. Solche Spalten werden mit dem Standardwert gefüllt, der in der Definition der Zieltabelle angegeben ist. Logische Replikation im Binärformat ist jedoch restriktiver. Details finden Sie bei der Option `binary` von `CREATE SUBSCRIPTION`.

### 29.2.1. Replikationsslot-Verwaltung

Wie bereits erwähnt, empfängt jede aktive Subscription Änderungen von einem Replikationsslot auf der entfernten, veröffentlichenden Seite.

Zusätzliche Slots für die Tabellensynchronisierung sind normalerweise vorübergehend; sie werden intern erstellt, um die anfängliche Tabellensynchronisierung auszuführen, und automatisch gelöscht, sobald sie nicht mehr benötigt werden. Diese Slots für die Tabellensynchronisierung haben generierte Namen: `pg_%u_sync_%u_%llu` (Parameter: Subscription-OID, Tabellen-Relid, Systemkennung `sysid`).

Normalerweise wird der entfernte Replikationsslot automatisch erstellt, wenn die Subscription mit `CREATE SUBSCRIPTION` erstellt wird, und automatisch gelöscht, wenn die Subscription mit `DROP SUBSCRIPTION` gelöscht wird. In einigen Situationen kann es jedoch nützlich oder notwendig sein, die Subscription und den zugrunde liegenden Replikationsslot getrennt zu bearbeiten. Einige Szenarien:

- Beim Erstellen einer Subscription existiert der Replikationsslot bereits. In diesem Fall kann die Subscription mit der Option `create_slot = false` erstellt werden, um sie mit dem vorhandenen Slot zu verknüpfen.
- Beim Erstellen einer Subscription ist der entfernte Host nicht erreichbar oder in einem unklaren Zustand. In diesem Fall kann die Subscription mit der Option `connect = false` erstellt werden. Der entfernte Host wird dann überhaupt nicht kontaktiert. Dies verwendet `pg_dump`. Der entfernte Replikationsslot muss dann manuell erstellt werden, bevor die Subscription aktiviert werden kann.
- Beim Löschen einer Subscription soll der Replikationsslot behalten werden. Das kann nützlich sein, wenn die Subscriber-Datenbank auf einen anderen Host verschoben und dort aktiviert wird. Trennen Sie in diesem Fall den Slot mit `ALTER SUBSCRIPTION` von der Subscription, bevor Sie versuchen, die Subscription zu löschen.
- Beim Löschen einer Subscription ist der entfernte Host nicht erreichbar. Trennen Sie in diesem Fall den Slot mit `ALTER SUBSCRIPTION` von der Subscription, bevor Sie versuchen, die Subscription zu löschen. Wenn die entfernte Datenbankinstanz nicht mehr existiert, ist danach keine weitere Aktion erforderlich. Wenn die entfernte Datenbankinstanz jedoch nur nicht erreichbar ist, sollten der Replikationsslot und alle gegebenenfalls noch vorhandenen Slots für die Tabellensynchronisierung anschließend manuell gelöscht werden; andernfalls würden sie weiterhin WAL reservieren und könnten schließlich dazu führen, dass die Platte vollläuft. Solche Fälle sollten sorgfältig untersucht werden.

### 29.2.2. Beispiele: Logische Replikation einrichten

Erstellen Sie einige Testtabellen auf dem Publisher.

```sql
/* pub # */ CREATE TABLE t1(a int, b text, PRIMARY KEY(a));
/* pub # */ CREATE TABLE t2(c int, d text, PRIMARY KEY(c));
/* pub # */ CREATE TABLE t3(e int, f text, PRIMARY KEY(e));
```

Erstellen Sie dieselben Tabellen auf dem Subscriber.

```sql
/* sub # */ CREATE TABLE t1(a int, b text, PRIMARY KEY(a));
/* sub # */ CREATE TABLE t2(c int, d text, PRIMARY KEY(c));
/* sub # */ CREATE TABLE t3(e int, f text, PRIMARY KEY(e));
```

Fügen Sie Daten auf der Publisher-Seite in die Tabellen ein.

```sql
/* pub # */ INSERT INTO t1 VALUES (1, 'one'), (2, 'two'), (3, 'three');
/* pub # */ INSERT INTO t2 VALUES (1, 'A'), (2, 'B'), (3, 'C');
/* pub # */ INSERT INTO t3 VALUES (1, 'i'), (2, 'ii'), (3, 'iii');
```

Erstellen Sie Publications für die Tabellen. Die Publications `pub2` und `pub3a` verbieten einige Publish-Operationen. Die Publication `pub3b` hat einen Zeilenfilter (siehe [Abschnitt 29.4](29_Logische_Replikation.md#294-zeilenfilter)).

```sql
/* pub # */ CREATE PUBLICATION pub1 FOR TABLE t1;
/* pub # */ CREATE PUBLICATION pub2 FOR TABLE t2 WITH (publish = 'truncate');
/* pub # */ CREATE PUBLICATION pub3a FOR TABLE t3 WITH (publish = 'truncate');
/* pub # */ CREATE PUBLICATION pub3b FOR TABLE t3 WHERE (e > 5);
```

Erstellen Sie Subscriptions für die Publications. Die Subscription `sub3` abonniert sowohl `pub3a` als auch `pub3b`. Alle Subscriptions kopieren standardmäßig die Anfangsdaten.

```sql
/* sub # */ CREATE SUBSCRIPTION sub1
/* sub - */ CONNECTION 'host=localhost dbname=test_pub application_name=sub1'
/* sub - */ PUBLICATION pub1;
/* sub # */ CREATE SUBSCRIPTION sub2
/* sub - */ CONNECTION 'host=localhost dbname=test_pub application_name=sub2'
/* sub - */ PUBLICATION pub2;
/* sub # */ CREATE SUBSCRIPTION sub3
/* sub - */ CONNECTION 'host=localhost dbname=test_pub application_name=sub3'
/* sub - */ PUBLICATION pub3a, pub3b;
```

Beachten Sie, dass die anfänglichen Tabellendaten unabhängig von der Publish-Operation der Publication kopiert werden.

```sql
/* sub # */ SELECT * FROM t1;
```

```text
 a |   b
---+-------
 1 | one
 2 | two
 3 | three
(3 rows)
```

```sql
/* sub # */ SELECT * FROM t2;
```

```text
 c | d
---+---
 1 | A
 2 | B
 3 | C
(3 rows)
```

Da die anfängliche Datenkopie die Publish-Operation ignoriert und weil die Publication `pub3a` keinen Zeilenfilter hat, enthält die kopierte Tabelle `t3` außerdem alle Zeilen, auch wenn sie nicht zum Zeilenfilter der Publication `pub3b` passen.

```sql
/* sub # */ SELECT * FROM t3;
```

```text
 e | f
---+-----
 1 | i
 2 | ii
 3 | iii
(3 rows)
```

Fügen Sie auf der Publisher-Seite weitere Daten in die Tabellen ein.

```sql
/* pub # */ INSERT INTO t1 VALUES (4, 'four'), (5, 'five'), (6, 'six');
/* pub # */ INSERT INTO t2 VALUES (4, 'D'), (5, 'E'), (6, 'F');
/* pub # */ INSERT INTO t3 VALUES (4, 'iv'), (5, 'v'), (6, 'vi');
```

Die Daten auf der Publisher-Seite sehen nun so aus:

```sql
/* pub # */ SELECT * FROM t1;
```

```text
 a |   b
---+-------
 1 | one
 2 | two
 3 | three
 4 | four
 5 | five
 6 | six
(6 rows)
```

```sql
/* pub # */ SELECT * FROM t2;
```

```text
 c | d
---+---
 1 | A
 2 | B
 3 | C
 4 | D
 5 | E
 6 | F
(6 rows)
```

```sql
/* pub # */ SELECT * FROM t3;
```

```text
 e | f
---+-----
 1 | i
 2 | ii
 3 | iii
 4 | iv
 5 | v
 6 | vi
(6 rows)
```

Beachten Sie, dass während der normalen Replikation die passenden Publish-Operationen verwendet werden. Das bedeutet, dass die Publications `pub2` und `pub3a` das `INSERT` nicht replizieren. Außerdem repliziert die Publication `pub3b` nur Daten, die zum Zeilenfilter von `pub3b` passen. Die Daten auf der Subscriber-Seite sehen nun so aus:

```sql
/* sub # */ SELECT * FROM t1;
```

```text
 a |   b
---+-------
 1 | one
 2 | two
 3 | three
 4 | four
 5 | five
 6 | six
(6 rows)
```

```sql
/* sub # */ SELECT * FROM t2;
```

```text
 c | d
---+---
 1 | A
 2 | B
 3 | C
(3 rows)
```

```sql
/* sub # */ SELECT * FROM t3;
```

```text
 e | f
---+-----
 1 | i
 2 | ii
 3 | iii
 6 | vi
(4 rows)
```

### 29.2.3. Beispiele: Verzögerte Erstellung von Replikationsslots

Es gibt einige Fälle (zum Beispiel Abschnitt 29.2.1), in denen der Benutzer den entfernten Replikationsslot manuell erstellen muss, bevor die Subscription aktiviert werden kann, falls der entfernte Replikationsslot nicht automatisch erstellt wurde. Die Schritte zum Erstellen des Slots und zum Aktivieren der Subscription werden in den folgenden Beispielen gezeigt. Diese Beispiele geben das Standard-Ausgabe-Plugin für logische Decodierung an (`pgoutput`), das die eingebaute logische Replikation verwendet.

Erstellen Sie zuerst eine Publication, die in den Beispielen verwendet wird.

```sql
/* pub # */ CREATE PUBLICATION pub1 FOR ALL TABLES;
```

Beispiel 1: Die Subscription gibt `connect = false` an.

- Erstellen Sie die Subscription.

```sql
/* sub # */ CREATE SUBSCRIPTION sub1
/* sub - */ CONNECTION 'host=localhost dbname=test_pub'
/* sub - */ PUBLICATION pub1
/* sub - */ WITH (connect=false);
```

```text
WARNING: subscription was created, but is not connected
HINT: To initiate replication, you must manually create the
 replication slot, enable the subscription, and refresh the
 subscription.
```

- Erstellen Sie auf dem Publisher manuell einen Slot. Da der Name während `CREATE SUBSCRIPTION` nicht angegeben wurde, entspricht der Name des zu erstellenden Slots dem Namen der Subscription, zum Beispiel `sub1`.

```sql
/* pub # */ SELECT * FROM
 pg_create_logical_replication_slot('sub1', 'pgoutput');
```

```text
 slot_name |    lsn
-----------+-----------
 sub1      | 0/19404D0
(1 row)
```

- Schließen Sie auf dem Subscriber die Aktivierung der Subscription ab. Danach beginnen die Tabellen von `pub1` mit der Replikation.

```sql
/* sub # */ ALTER SUBSCRIPTION sub1 ENABLE;
/* sub # */ ALTER SUBSCRIPTION sub1 REFRESH PUBLICATION;
```

Beispiel 2: Die Subscription gibt `connect = false` an, aber zusätzlich die Option `slot_name`.

- Erstellen Sie die Subscription.

```sql
/* sub # */ CREATE SUBSCRIPTION sub1
/* sub - */ CONNECTION 'host=localhost dbname=test_pub'
/* sub - */ PUBLICATION pub1
/* sub - */ WITH (connect=false, slot_name='myslot');
```

```text
WARNING: subscription was created, but is not connected
HINT: To initiate replication, you must manually create the
 replication slot, enable the subscription, and refresh the
 subscription.
```

- Erstellen Sie auf dem Publisher manuell einen Slot mit demselben Namen, der während `CREATE SUBSCRIPTION` angegeben wurde, zum Beispiel `myslot`.

```sql
/* pub # */ SELECT * FROM
 pg_create_logical_replication_slot('myslot', 'pgoutput');
```

```text
 slot_name |    lsn
-----------+-----------
 myslot    | 0/19059A0
(1 row)
```

- Auf dem Subscriber entsprechen die verbleibenden Aktivierungsschritte der Subscription denen von zuvor.

```sql
/* sub # */ ALTER SUBSCRIPTION sub1 ENABLE;
/* sub # */ ALTER SUBSCRIPTION sub1 REFRESH PUBLICATION;
```

Beispiel 3: Die Subscription gibt `slot_name = NONE` an.

- Erstellen Sie die Subscription. Wenn `slot_name = NONE` gesetzt ist, sind außerdem `enabled = false` und `create_slot = false` erforderlich.

```sql
/* sub # */ CREATE SUBSCRIPTION sub1
/* sub - */ CONNECTION 'host=localhost dbname=test_pub'
/* sub - */ PUBLICATION pub1
/* sub - */ WITH (slot_name=NONE, enabled=false, create_slot=false);
```

- Erstellen Sie auf dem Publisher manuell einen Slot mit einem beliebigen Namen, zum Beispiel `myslot`.

```sql
/* pub # */ SELECT * FROM
 pg_create_logical_replication_slot('myslot', 'pgoutput');
```

```text
 slot_name |    lsn
-----------+-----------
 myslot    | 0/1905930
(1 row)
```

- Verknüpfen Sie auf dem Subscriber die Subscription mit dem gerade erstellten Slot-Namen.

```sql
/* sub # */ ALTER SUBSCRIPTION sub1 SET (slot_name='myslot');
```

- Die übrigen Aktivierungsschritte der Subscription entsprechen denen von zuvor.

```sql
/* sub # */ ALTER SUBSCRIPTION sub1 ENABLE;
/* sub # */ ALTER SUBSCRIPTION sub1 REFRESH PUBLICATION;
```

## 29.3. Failover bei logischer Replikation

Damit Subscriber-Knoten Daten vom Publisher-Knoten weiter replizieren können, selbst wenn der Publisher-Knoten ausfällt, muss es einen physischen Standby geben, der dem Publisher-Knoten entspricht. Die logischen Slots auf dem Primärserver, die den Subscriptions entsprechen, können mit dem Standby-Server synchronisiert werden, indem beim Erstellen von Subscriptions `failover = true` angegeben wird. Details finden Sie in Abschnitt 47.2.3. Das Aktivieren des Parameters `failover` stellt einen nahtlosen Übergang dieser Subscriptions sicher, nachdem der Standby hochgestuft wurde. Sie können Publications auf dem neuen Primärserver weiter abonnieren.

Da die Logik zur Slot-Synchronisierung asynchron kopiert, muss vor dem Failover bestätigt werden, dass die Replikationsslots mit dem Standby-Server synchronisiert wurden. Für einen erfolgreichen Failover muss der Standby-Server dem Subscriber voraus sein. Dies kann durch Konfiguration von `synchronized_standby_slots` erreicht werden.

Um zu bestätigen, dass der Standby-Server für einen bestimmten Subscriber tatsächlich failoverbereit ist, prüfen Sie mit den folgenden Schritten, ob alle von diesem Subscriber benötigten logischen Replikationsslots mit dem Standby-Server synchronisiert wurden:

1. Verwenden Sie auf dem Subscriber-Knoten das folgende SQL, um zu ermitteln, welche Replikationsslots mit dem Standby synchronisiert werden sollen, der hochgestuft werden soll. Diese Abfrage gibt die relevanten Replikationsslots zurück, die mit failoverfähigen Subscriptions verknüpft sind.

```sql
/* sub # */ SELECT
               array_agg(quote_literal(s.subslotname)) AS slots
           FROM pg_subscription s
           WHERE s.subfailover AND
                 s.subslotname IS NOT NULL;
```

```text
 slots
-------
 {'sub1','sub2','sub3'}
(1 row)
```

2. Verwenden Sie auf dem Subscriber-Knoten das folgende SQL, um zu ermitteln, welche Slots für die Tabellensynchronisierung mit dem Standby synchronisiert werden sollen, der hochgestuft werden soll. Diese Abfrage muss in jeder Datenbank ausgeführt werden, die failoverfähige Subscription(s) enthält. Beachten Sie, dass der Tabellensynchronisationsslot nur dann mit dem Standby-Server synchronisiert werden sollte, wenn die Tabellenkopie abgeschlossen ist (siehe [Abschnitt 52.55](52_Systemkataloge.md#5255-pgsubscriptionrel)). In anderen Szenarien müssen Sie nicht sicherstellen, dass die Tabellensynchronisationsslots synchronisiert sind, weil sie dann entweder gelöscht oder auf dem neuen Primärserver neu erstellt werden.

```sql
/* sub # */ SELECT
               array_agg(quote_literal(slot_name)) AS slots
          FROM
          (
               SELECT CONCAT('pg_', srsubid, '_sync_', srrelid,
                             '_', ctl.system_identifier) AS slot_name
               FROM pg_control_system() ctl,
                    pg_subscription_rel r,
                    pg_subscription s
               WHERE r.srsubstate = 'f' AND s.oid = r.srsubid
                     AND s.subfailover
          );
```

```text
 slots
-------
 {'pg_16394_sync_16385_7394666715149055164'}
(1 row)
```

3. Prüfen Sie, ob die oben identifizierten logischen Replikationsslots auf dem Standby-Server existieren und für den Failover bereit sind.

```sql
/* standby # */ SELECT slot_name,
                       (synced AND NOT temporary AND
                        invalidation_reason IS NULL) AS failover_ready
                FROM pg_replication_slots
                WHERE slot_name IN
                    ('sub1','sub2','sub3',
                     'pg_16394_sync_16385_7394666715149055164');
```

```text
 slot_name                                  | failover_ready
--------------------------------------------+----------------
 sub1                                       | t
 sub2                                       | t
 sub3                                       | t
 pg_16394_sync_16385_7394666715149055164    | t
(4 rows)
```

Wenn alle Slots auf dem Standby-Server vorhanden sind und das Ergebnis (`failover_ready`) der obigen SQL-Abfrage `true` ist, können bestehende Subscriptions weiterhin Publications auf dem neuen Primärserver abonnieren.

Die ersten beiden Schritte des obigen Verfahrens sind für einen PostgreSQL-Subscriber gedacht. Es wird empfohlen, diese Schritte auf jedem Subscriber-Knoten auszuführen, der nach dem Failover vom vorgesehenen Standby bedient wird, um die vollständige Liste der Replikationsslots zu erhalten. Diese Liste kann anschließend in Schritt 3 geprüft werden, um die Failover-Bereitschaft sicherzustellen. Nicht-PostgreSQL-Subscriber können dagegen eigene Methoden verwenden, um die Replikationsslots zu identifizieren, die von ihren jeweiligen Subscriptions verwendet werden.

In einigen Fällen, etwa bei einem geplanten Failover, muss bestätigt werden, dass alle Subscriber, PostgreSQL oder nicht, nach dem Failover zu einem bestimmten Standby-Server die Replikation fortsetzen können. Verwenden Sie in solchen Fällen statt der ersten beiden Schritte oben das folgende SQL, um zu ermitteln, welche Replikationsslots auf dem Primärserver mit dem Standby synchronisiert werden müssen, der hochgestuft werden soll. Diese Abfrage gibt die relevanten Replikationsslots zurück, die mit allen failoverfähigen Subscriptions verknüpft sind.

```sql
/* primary # */ SELECT array_agg(quote_literal(r.slot_name)) AS slots
                FROM pg_replication_slots r
                WHERE r.failover AND NOT r.temporary;
```

```text
 slots
-------
 {'sub1','sub2','sub3', 'pg_16394_sync_16385_7394666715149055164'}
(1 row)
```

## 29.4. Zeilenfilter

Standardmäßig werden alle Daten aus allen veröffentlichten Tabellen zu den passenden Subscribern repliziert. Die replizierten Daten können durch einen Zeilenfilter reduziert werden. Ein Benutzer kann Zeilenfilter aus Verhaltens-, Sicherheits- oder Performance-Gründen einsetzen. Wenn eine veröffentlichte Tabelle einen Zeilenfilter setzt, wird eine Zeile nur repliziert, wenn ihre Daten den Zeilenfilterausdruck erfüllen. Dadurch kann ein Tabellensatz teilweise repliziert werden. Der Zeilenfilter wird pro Tabelle definiert. Verwenden Sie eine `WHERE`-Klausel nach dem Tabellennamen für jede veröffentlichte Tabelle, aus der Daten herausgefiltert werden sollen. Die `WHERE`-Klausel muss in Klammern eingeschlossen sein. Details finden Sie unter `CREATE PUBLICATION`.

### 29.4.1. Regeln für Zeilenfilter

Zeilenfilter werden angewendet, bevor die Änderungen veröffentlicht werden. Wenn der Zeilenfilter `false` oder `NULL` ergibt, wird die Zeile nicht repliziert. Der Ausdruck der `WHERE`-Klausel wird mit derselben Rolle ausgewertet, die für die Replikationsverbindung verwendet wird, also mit der Rolle, die in der `CONNECTION`-Klausel von `CREATE SUBSCRIPTION` angegeben ist. Zeilenfilter haben keine Wirkung für den Befehl `TRUNCATE`.

### 29.4.2. Ausdrucksbeschränkungen

Die `WHERE`-Klausel erlaubt nur einfache Ausdrücke. Sie darf keine benutzerdefinierten Funktionen, Operatoren, Typen und Collations, keine Verweise auf Systemspalten und keine nicht unveränderlichen eingebauten Funktionen enthalten.

Wenn eine Publication `UPDATE`- oder `DELETE`-Operationen veröffentlicht, darf die `WHERE`-Klausel des Zeilenfilters nur Spalten enthalten, die von der Replica Identity abgedeckt sind (siehe `REPLICA IDENTITY`). Wenn eine Publication nur `INSERT`-Operationen veröffentlicht, kann die `WHERE`-Klausel des Zeilenfilters jede Spalte verwenden.

### 29.4.3. UPDATE-Transformationen

Immer wenn ein `UPDATE` verarbeitet wird, wird der Zeilenfilterausdruck sowohl für die alte als auch für die neue Zeile ausgewertet, also anhand der Daten vor und nach der Aktualisierung. Wenn beide Auswertungen wahr sind, wird die `UPDATE`-Änderung repliziert. Wenn beide Auswertungen falsch sind, wird die Änderung nicht repliziert. Wenn nur eine der alten/neuen Zeilen zum Zeilenfilterausdruck passt, wird das `UPDATE` in ein `INSERT` oder `DELETE` umgewandelt, um Dateninkonsistenzen zu vermeiden. Die Zeile auf dem Subscriber sollte dem entsprechen, was durch den Zeilenfilterausdruck auf dem Publisher definiert ist.

Wenn die alte Zeile den Zeilenfilterausdruck erfüllt, also an den Subscriber gesendet wurde, die neue Zeile aber nicht, dann sollte die alte Zeile aus Sicht der Datenkonsistenz vom Subscriber entfernt werden. Daher wird das `UPDATE` in ein `DELETE` umgewandelt.

Wenn die alte Zeile den Zeilenfilterausdruck nicht erfüllt, also nicht an den Subscriber gesendet wurde, die neue Zeile aber schon, dann sollte die neue Zeile aus Sicht der Datenkonsistenz zum Subscriber hinzugefügt werden. Daher wird das `UPDATE` in ein `INSERT` umgewandelt.

Tabelle 29.1 fasst die angewendeten Transformationen zusammen.

**Tabelle 29.1. Zusammenfassung der UPDATE-Transformationen**

| Alte Zeile | Neue Zeile | Transformation |
| --- | --- | --- |
| keine Übereinstimmung | keine Übereinstimmung | nicht replizieren |
| keine Übereinstimmung | Übereinstimmung | `INSERT` |
| Übereinstimmung | keine Übereinstimmung | `DELETE` |
| Übereinstimmung | Übereinstimmung | `UPDATE` |

### 29.4.4. Partitionierte Tabellen

Wenn die Publication eine partitionierte Tabelle enthält, bestimmt der Publication-Parameter `publish_via_partition_root`, welcher Zeilenfilter verwendet wird. Wenn `publish_via_partition_root` `true` ist, wird der Zeilenfilter der partitionierten Root-Tabelle verwendet. Andernfalls, wenn `publish_via_partition_root` `false` ist, was dem Standard entspricht, wird der Zeilenfilter jeder Partition verwendet.

### 29.4.5. Anfängliche Datensynchronisierung

Wenn die Subscription das Kopieren bereits vorhandener Tabellendaten erfordert und eine Publication `WHERE`-Klauseln enthält, werden nur Daten zum Subscriber kopiert, die die Zeilenfilterausdrücke erfüllen.

Wenn die Subscription mehrere Publications hat, in denen eine Tabelle mit unterschiedlichen `WHERE`-Klauseln veröffentlicht wurde, werden Zeilen kopiert, die einen der Ausdrücke erfüllen. Details finden Sie in Abschnitt 29.4.6.

> **Warnung**
>
> Da die anfängliche Datensynchronisierung den Parameter `publish` beim Kopieren vorhandener Tabellendaten nicht berücksichtigt, können einige Zeilen kopiert werden, die über DML nicht repliziert würden. Siehe Abschnitt 29.9.1 und die Beispiele in Abschnitt 29.2.2.

> **Hinweis**
>
> Wenn der Subscriber eine Version vor 15 verwendet, nutzt das Kopieren bereits vorhandener Daten keine Zeilenfilter, selbst wenn diese in der Publication definiert sind. Das liegt daran, dass ältere Versionen nur die gesamten Tabellendaten kopieren können.

### 29.4.6. Mehrere Zeilenfilter kombinieren

Wenn die Subscription mehrere Publications hat, in denen dieselbe Tabelle mit unterschiedlichen Zeilenfiltern für dieselbe Publish-Operation veröffentlicht wurde, werden diese Ausdrücke mit `OR` verknüpft, sodass Zeilen repliziert werden, die einen der Ausdrücke erfüllen. Das bedeutet, dass alle anderen Zeilenfilter für dieselbe Tabelle redundant werden, wenn:

- eine der Publications keinen Zeilenfilter hat,
- eine der Publications mit `FOR ALL TABLES` erstellt wurde; diese Klausel erlaubt keine Zeilenfilter,
- eine der Publications mit `FOR TABLES IN SCHEMA` erstellt wurde und die Tabelle zu dem referenzierten Schema gehört; diese Klausel erlaubt keine Zeilenfilter.

### 29.4.7. Beispiele

Erstellen Sie einige Tabellen, die in den folgenden Beispielen verwendet werden.

```sql
/* pub # */ CREATE TABLE t1(a int, b int, c text, PRIMARY KEY(a,c));
/* pub # */ CREATE TABLE t2(d int, e int, f int, PRIMARY KEY(d));
/* pub # */ CREATE TABLE t3(g int, h int, i int, PRIMARY KEY(g));
```

Erstellen Sie einige Publications. Publication `p1` hat eine Tabelle (`t1`), und diese Tabelle hat einen Zeilenfilter. Publication `p2` hat zwei Tabellen. Tabelle `t1` hat keinen Zeilenfilter, Tabelle `t2` hat einen Zeilenfilter. Publication `p3` hat zwei Tabellen, und beide haben einen Zeilenfilter.

```sql
/* pub # */ CREATE PUBLICATION p1 FOR TABLE t1 WHERE (a > 5 AND c = 'NSW');
/* pub # */ CREATE PUBLICATION p2 FOR TABLE t1, t2 WHERE (e = 99);
/* pub # */ CREATE PUBLICATION p3 FOR TABLE t2 WHERE (d = 10), t3 WHERE (g = 10);
```

Mit `psql` können die Zeilenfilterausdrücke, sofern definiert, für jede Publication angezeigt werden.

```sql
/* pub # */ \dRp+
```

```text
                                         Publication p1
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Generated columns | Via root
----------+------------+---------+---------+---------+-----------+-------------------+----------
 postgres | f          | t       | t       | t       | t         | none              | f
Tables:
    "public.t1" WHERE ((a > 5) AND (c = 'NSW'::text))

                                         Publication p2
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Generated columns | Via root
----------+------------+---------+---------+---------+-----------+-------------------+----------
 postgres | f          | t       | t       | t       | t         | none              | f
Tables:
    "public.t1"
    "public.t2" WHERE (e = 99)

                                         Publication p3
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Generated columns | Via root
----------+------------+---------+---------+---------+-----------+-------------------+----------
 postgres | f          | t       | t       | t       | t         | none              | f
Tables:
    "public.t2" WHERE (d = 10)
    "public.t3" WHERE (g = 10)
```

Mit `psql` können die Zeilenfilterausdrücke, sofern definiert, für jede Tabelle angezeigt werden. Beachten Sie, dass Tabelle `t1` Mitglied zweier Publications ist, aber nur in `p1` einen Zeilenfilter hat. Beachten Sie außerdem, dass Tabelle `t2` Mitglied zweier Publications ist und in jeder davon einen anderen Zeilenfilter hat.

```sql
/* pub # */ \d t1
```

```text
                  Table "public.t1"
 Column | Type    | Collation | Nullable | Default
--------+---------+-----------+----------+---------
 a      | integer |           | not null |
 b      | integer |           |          |
 c      | text    |           | not null |
Indexes:
    "t1_pkey" PRIMARY KEY, btree (a, c)
Publications:
    "p1" WHERE ((a > 5) AND (c = 'NSW'::text))
    "p2"
```

```sql
/* pub # */ \d t2
```

```text
                  Table "public.t2"
 Column | Type    | Collation | Nullable | Default
--------+---------+-----------+----------+---------
 d      | integer |           | not null |
 e      | integer |           |          |
 f      | integer |           |          |
Indexes:
    "t2_pkey" PRIMARY KEY, btree (d)
Publications:
    "p2" WHERE (e = 99)
    "p3" WHERE (d = 10)
```

```sql
/* pub # */ \d t3
```

```text
                  Table "public.t3"
 Column | Type    | Collation | Nullable | Default
--------+---------+-----------+----------+---------
 g      | integer |           | not null |
 h      | integer |           |          |
 i      | integer |           |          |
Indexes:
    "t3_pkey" PRIMARY KEY, btree (g)
Publications:
    "p3" WHERE (g = 10)
```

Erstellen Sie auf dem Subscriber-Knoten eine Tabelle `t1` mit derselben Definition wie auf dem Publisher und außerdem die Subscription `s1`, die die Publication `p1` abonniert.

```sql
/* sub # */ CREATE TABLE t1(a int, b int, c text, PRIMARY KEY(a,c));
/* sub # */ CREATE SUBSCRIPTION s1
/* sub - */ CONNECTION 'host=localhost dbname=test_pub application_name=s1'
/* sub - */ PUBLICATION p1;
```

Fügen Sie einige Zeilen ein. Nur die Zeilen, die die `WHERE`-Klausel von Tabelle `t1` in Publication `p1` erfüllen, werden repliziert.

```sql
/* pub # */ INSERT INTO t1 VALUES (2, 102, 'NSW');
/* pub # */ INSERT INTO t1 VALUES (3, 103, 'QLD');
/* pub # */ INSERT INTO t1 VALUES (4, 104, 'VIC');
/* pub # */ INSERT INTO t1 VALUES (5, 105, 'ACT');
/* pub # */ INSERT INTO t1 VALUES (6, 106, 'NSW');
/* pub # */ INSERT INTO t1 VALUES (7, 107, 'NT');
/* pub # */ INSERT INTO t1 VALUES (8, 108, 'QLD');
/* pub # */ INSERT INTO t1 VALUES (9, 109, 'NSW');
```

```sql
/* pub # */ SELECT * FROM t1;
```

```text
 a | b   | c
---+-----+-----
 2 | 102 | NSW
 3 | 103 | QLD
 4 | 104 | VIC
 5 | 105 | ACT
 6 | 106 | NSW
 7 | 107 | NT
 8 | 108 | QLD
 9 | 109 | NSW
(8 rows)
```

```sql
/* sub # */ SELECT * FROM t1;
```

```text
 a | b   | c
---+-----+-----
 6 | 106 | NSW
 9 | 109 | NSW
(2 rows)
```

Aktualisieren Sie einige Daten, bei denen alte und neue Zeilenwerte beide die `WHERE`-Klausel von `t1` in Publication `p1` erfüllen. Das `UPDATE` repliziert die Änderung normal.

```sql
/* pub # */ UPDATE t1 SET b = 999 WHERE a = 6;
/* pub # */ SELECT * FROM t1;
```

```text
 a | b   | c
---+-----+-----
 2 | 102 | NSW
 3 | 103 | QLD
 4 | 104 | VIC
 5 | 105 | ACT
 7 | 107 | NT
 8 | 108 | QLD
 9 | 109 | NSW
 6 | 999 | NSW
(8 rows)
```

```sql
/* sub # */ SELECT * FROM t1;
```

```text
 a | b   | c
---+-----+-----
 9 | 109 | NSW
 6 | 999 | NSW
(2 rows)
```

Aktualisieren Sie einige Daten, bei denen die alten Zeilenwerte die `WHERE`-Klausel von `t1` in Publication `p1` nicht erfüllten, die neuen Zeilenwerte sie aber erfüllen. Das `UPDATE` wird in ein `INSERT` umgewandelt und die Änderung repliziert. Beachten Sie die neue Zeile auf dem Subscriber.

```sql
/* pub # */ UPDATE t1 SET a = 555 WHERE a = 2;
/* pub # */ SELECT * FROM t1;
```

```text
  a | b   | c
----+-----+-----
  3 | 103 | QLD
  4 | 104 | VIC
  5 | 105 | ACT
  7 | 107 | NT
  8 | 108 | QLD
  9 | 109 | NSW
  6 | 999 | NSW
555 | 102 | NSW
(8 rows)
```

```sql
/* sub # */ SELECT * FROM t1;
```

```text
  a | b   | c
----+-----+-----
  9 | 109 | NSW
  6 | 999 | NSW
555 | 102 | NSW
(3 rows)
```

Aktualisieren Sie einige Daten, bei denen die alten Zeilenwerte die `WHERE`-Klausel von `t1` in Publication `p1` erfüllten, die neuen Zeilenwerte sie aber nicht erfüllen. Das `UPDATE` wird in ein `DELETE` umgewandelt und die Änderung repliziert. Beachten Sie, dass die Zeile vom Subscriber entfernt wird.

```sql
/* pub # */ UPDATE t1 SET c = 'VIC' WHERE a = 9;
/* pub # */ SELECT * FROM t1;
```

```text
  a | b   | c
----+-----+-----
  3 | 103 | QLD
  4 | 104 | VIC
  5 | 105 | ACT
  7 | 107 | NT
  8 | 108 | QLD
  6 | 999 | NSW
555 | 102 | NSW
  9 | 109 | VIC
(8 rows)
```

```sql
/* sub # */ SELECT * FROM t1;
```

```text
  a | b   | c
----+-----+-----
  6 | 999 | NSW
555 | 102 | NSW
(2 rows)
```

Die folgenden Beispiele zeigen, wie der Publication-Parameter `publish_via_partition_root` bestimmt, ob bei partitionierten Tabellen der Zeilenfilter der Eltern- oder Kindtabelle verwendet wird.

Erstellen Sie auf dem Publisher eine partitionierte Tabelle.

```sql
/* pub # */ CREATE TABLE parent(a int PRIMARY KEY) PARTITION BY RANGE(a);
/* pub # */ CREATE TABLE child PARTITION OF parent DEFAULT;
```

Erstellen Sie dieselben Tabellen auf dem Subscriber.

```sql
/* sub # */ CREATE TABLE parent(a int PRIMARY KEY) PARTITION BY RANGE(a);
/* sub # */ CREATE TABLE child PARTITION OF parent DEFAULT;
```

Erstellen Sie eine Publication `p4` und abonnieren Sie sie anschließend. Der Publication-Parameter `publish_via_partition_root` wird auf `true` gesetzt. Zeilenfilter sind sowohl auf der partitionierten Tabelle (`parent`) als auch auf der Partition (`child`) definiert.

```sql
/* pub # */ CREATE PUBLICATION p4 FOR TABLE parent WHERE (a < 5),
 child WHERE (a >= 5)
/* pub - */ WITH (publish_via_partition_root=true);

/* sub # */ CREATE SUBSCRIPTION s4
/* sub - */ CONNECTION 'host=localhost dbname=test_pub application_name=s4'
/* sub - */ PUBLICATION p4;
```

Fügen Sie einige Werte direkt in die Eltern- und Kindtabellen ein. Sie werden mit dem Zeilenfilter von `parent` repliziert, weil `publish_via_partition_root` `true` ist.

```sql
/* pub # */ INSERT INTO parent VALUES (2), (4), (6);
/* pub # */ INSERT INTO child VALUES (3), (5), (7);
```

```sql
/* pub # */ SELECT * FROM parent ORDER BY a;
```

```text
 a
---
(6 rows)
```

```sql
/* sub # */ SELECT * FROM parent ORDER BY a;
```

```text
 a
---
(3 rows)
```

Wiederholen Sie denselben Test, aber mit einem anderen Wert für `publish_via_partition_root`. Der Publication-Parameter `publish_via_partition_root` wird auf `false` gesetzt. Ein Zeilenfilter wird auf der Partition (`child`) definiert.

```sql
/* pub # */ DROP PUBLICATION p4;
/* pub # */ CREATE PUBLICATION p4 FOR TABLE parent, child WHERE (a >= 5)
/* pub - */ WITH (publish_via_partition_root=false);

/* sub # */ ALTER SUBSCRIPTION s4 REFRESH PUBLICATION;
```

Führen Sie die Einfügungen auf dem Publisher wie zuvor aus. Sie werden mit dem Zeilenfilter von `child` repliziert, weil `publish_via_partition_root` `false` ist.

```sql
/* pub # */ TRUNCATE parent;
/* pub # */ INSERT INTO parent VALUES (2), (4), (6);
/* pub # */ INSERT INTO child VALUES (3), (5), (7);
```

```sql
/* pub # */ SELECT * FROM parent ORDER BY a;
```

```text
 a
---
(6 rows)
```

```sql
/* sub # */ SELECT * FROM child ORDER BY a;
```

```text
 a
---
(3 rows)
```

## 29.5. Spaltenlisten

Jede Publication kann optional angeben, welche Spalten jeder Tabelle zu Subscribern repliziert werden. Die Tabelle auf der Subscriber-Seite muss mindestens alle Spalten haben, die veröffentlicht werden. Wenn keine Spaltenliste angegeben ist, werden alle Spalten auf dem Publisher repliziert. Details zur Syntax finden Sie unter `CREATE PUBLICATION`.

Die Auswahl der Spalten kann aus Verhaltens- oder Performance-Gründen erfolgen. Verlassen Sie sich aus Sicherheitsgründen jedoch nicht auf diese Funktion: Ein böswilliger Subscriber kann Daten aus Spalten erhalten, die nicht ausdrücklich veröffentlicht wurden. Wenn Sicherheit eine Rolle spielt, sollten Schutzmaßnahmen auf der Publisher-Seite angewendet werden.

Wenn keine Spaltenliste angegeben ist, werden später zur Tabelle hinzugefügte Spalten automatisch repliziert. Das bedeutet, dass eine Spaltenliste, die alle Spalten nennt, nicht dasselbe ist wie gar keine Spaltenliste.

Eine Spaltenliste kann nur einfache Spaltenverweise enthalten. Die Reihenfolge der Spalten in der Liste bleibt nicht erhalten.

Generierte Spalten können ebenfalls in einer Spaltenliste angegeben werden. Dadurch können generierte Spalten unabhängig vom Publication-Parameter `publish_generated_columns` veröffentlicht werden. Details finden Sie in Abschnitt 29.6.

Eine Spaltenliste anzugeben, wenn die Publication auch `FOR TABLES IN SCHEMA` veröffentlicht, wird nicht unterstützt.

Bei partitionierten Tabellen bestimmt der Publication-Parameter `publish_via_partition_root`, welche Spaltenliste verwendet wird. Wenn `publish_via_partition_root` `true` ist, wird die Spaltenliste der partitionierten Root-Tabelle verwendet. Andernfalls, wenn `publish_via_partition_root` `false` ist, was dem Standard entspricht, wird die Spaltenliste jeder Partition verwendet.

Wenn eine Publication `UPDATE`- oder `DELETE`-Operationen veröffentlicht, muss jede Spaltenliste die Replica-Identity-Spalten der Tabelle enthalten (siehe `REPLICA IDENTITY`). Wenn eine Publication nur `INSERT`-Operationen veröffentlicht, darf die Spaltenliste Replica-Identity-Spalten auslassen.

Spaltenlisten haben keine Wirkung für den Befehl `TRUNCATE`.

Während der anfänglichen Datensynchronisierung werden nur die veröffentlichten Spalten kopiert. Wenn der Subscriber jedoch eine Version vor 15 verwendet, werden während der anfänglichen Datensynchronisierung alle Spalten der Tabelle kopiert und Spaltenlisten ignoriert. Wenn der Subscriber eine Version vor 18 verwendet, kopiert die anfängliche Tabellensynchronisierung keine generierten Spalten, selbst wenn sie im Publisher definiert sind.

> **Warnung: Kombinieren von Spaltenlisten aus mehreren Publications**
>
> Derzeit werden Subscriptions nicht unterstützt, die mehrere Publications umfassen, in denen dieselbe Tabelle mit unterschiedlichen Spaltenlisten veröffentlicht wurde. `CREATE SUBSCRIPTION` verhindert das Erstellen solcher Subscriptions, aber es ist dennoch möglich, in diese Situation zu geraten, indem nach dem Erstellen einer Subscription Spaltenlisten auf der Publication-Seite hinzugefügt oder geändert werden.
>
> Das bedeutet, dass das Ändern der Spaltenlisten von Tabellen in bereits abonnierten Publications zu Fehlern auf der Subscriber-Seite führen kann.
>
> Wenn eine Subscription von diesem Problem betroffen ist, kann die Replikation nur fortgesetzt werden, indem eine der Spaltenlisten auf der Publication-Seite so angepasst wird, dass alle übereinstimmen; anschließend muss entweder die Subscription neu erstellt oder mit `ALTER SUBSCRIPTION ... DROP PUBLICATION` eine der problematischen Publications entfernt und erneut hinzugefügt werden.

### 29.5.1. Beispiele

Erstellen Sie eine Tabelle `t1`, die im folgenden Beispiel verwendet wird.

```sql
/* pub # */ CREATE TABLE t1(id int, a text, b text, c text, d text,
 e text, PRIMARY KEY(id));
```

Erstellen Sie eine Publication `p1`. Für Tabelle `t1` wird eine Spaltenliste definiert, um die Anzahl der replizierten Spalten zu verringern. Beachten Sie, dass die Reihenfolge der Spaltennamen in der Spaltenliste keine Rolle spielt.

```sql
/* pub # */ CREATE PUBLICATION p1 FOR TABLE t1 (id, b, a, d);
```

Mit `psql` können die Spaltenlisten, sofern definiert, für jede Publication angezeigt werden.

```sql
/* pub # */ \dRp+
```

```text
                                             Publication p1
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Generated columns | Via root
----------+------------+---------+---------+---------+-----------+-------------------+----------
 postgres | f          | t       | t       | t       | t         | none              | f
Tables:
    "public.t1" (id, a, b, d)
```

Mit `psql` können die Spaltenlisten, sofern definiert, für jede Tabelle angezeigt werden.

```sql
/* pub # */ \d t1
```

```text
                  Table "public.t1"
 Column | Type    | Collation | Nullable | Default
--------+---------+-----------+----------+---------
 id     | integer |           | not null |
 a      | text    |           |          |
 b      | text    |           |          |
 c      | text    |           |          |
 d      | text    |           |          |
 e      | text    |           |          |
Indexes:
    "t1_pkey" PRIMARY KEY, btree (id)
Publications:
    "p1" (id, a, b, d)
```

Erstellen Sie auf dem Subscriber-Knoten eine Tabelle `t1`, die jetzt nur eine Teilmenge der Spalten benötigt, die in der Publisher-Tabelle `t1` vorhanden waren, und erstellen Sie außerdem die Subscription `s1`, die die Publication `p1` abonniert.

```sql
/* sub # */ CREATE TABLE t1(id int, b text, a text, d text, PRIMARY KEY(id));
/* sub # */ CREATE SUBSCRIPTION s1
/* sub - */ CONNECTION 'host=localhost dbname=test_pub application_name=s1'
/* sub - */ PUBLICATION p1;
```

Fügen Sie auf dem Publisher-Knoten einige Zeilen in Tabelle `t1` ein.

```sql
/* pub # */ INSERT INTO t1 VALUES(1, 'a-1', 'b-1', 'c-1', 'd-1', 'e-1');
/* pub # */ INSERT INTO t1 VALUES(2, 'a-2', 'b-2', 'c-2', 'd-2', 'e-2');
/* pub # */ INSERT INTO t1 VALUES(3, 'a-3', 'b-3', 'c-3', 'd-3', 'e-3');
/* pub # */ SELECT * FROM t1 ORDER BY id;
```

```text
 id | a   | b   | c   | d   | e
----+-----+-----+-----+-----+-----
  1 | a-1 | b-1 | c-1 | d-1 | e-1
  2 | a-2 | b-2 | c-2 | d-2 | e-2
  3 | a-3 | b-3 | c-3 | d-3 | e-3
(3 rows)
```

Nur Daten aus der Spaltenliste der Publication `p1` werden repliziert.

```sql
/* sub # */ SELECT * FROM t1 ORDER BY id;
```

```text
 id | b   | a   | d
----+-----+-----+-----
  1 | b-1 | a-1 | d-1
  2 | b-2 | a-2 | d-2
  3 | b-3 | a-3 | d-3
(3 rows)
```

## 29.6. Replikation generierter Spalten

Typischerweise wird eine Tabelle auf dem Subscriber genauso definiert wie die Publisher-Tabelle; wenn die Publisher-Tabelle also eine `GENERATED`-Spalte hat, hat die Subscriber-Tabelle eine entsprechende generierte Spalte. In diesem Fall wird immer der Wert der generierten Spalte der Subscriber-Tabelle verwendet.

Beachten Sie im folgenden Beispiel, dass der Wert der generierten Spalte der Subscriber-Tabelle aus der Berechnung der Subscriber-Spalte stammt.

```sql
/* pub # */ CREATE TABLE tab_gen_to_gen (a int, b int GENERATED
 ALWAYS AS (a + 1) STORED);
/* pub # */ INSERT INTO tab_gen_to_gen VALUES (1),(2),(3);
/* pub # */ CREATE PUBLICATION pub1 FOR TABLE tab_gen_to_gen;
/* pub # */ SELECT * FROM tab_gen_to_gen;
```

```text
 a | b
---+---
 1 | 2
 2 | 3
 3 | 4
(3 rows)
```

```sql
/* sub # */ CREATE TABLE tab_gen_to_gen (a int, b int GENERATED
 ALWAYS AS (a * 100) STORED);
/* sub # */ CREATE SUBSCRIPTION sub1 CONNECTION 'dbname=test_pub'
 PUBLICATION pub1;
/* sub # */ SELECT * from tab_gen_to_gen;
```

```text
 a | b
---+----
 1 | 100
 2 | 200
 3 | 300
(3 rows)
```

Tatsächlich veröffentlicht logische Replikation vor Version 18.0 überhaupt keine `GENERATED`-Spalten.

Es kann jedoch manchmal wünschenswert sein, eine generierte Spalte in eine reguläre Spalte zu replizieren.

> **Tipp**
>
> Diese Funktion kann nützlich sein, wenn Daten über ein Ausgabe-Plugin in eine Nicht-PostgreSQL-Datenbank repliziert werden, besonders wenn die Zieldatenbank keine generierten Spalten unterstützt.

Generierte Spalten werden standardmäßig nicht veröffentlicht, aber Benutzer können sich dafür entscheiden, gespeicherte generierte Spalten genauso wie reguläre Spalten zu veröffentlichen.

Dafür gibt es zwei Möglichkeiten:

- Setzen Sie den `PUBLICATION`-Parameter `publish_generated_columns` auf `stored`. Dadurch weist die logische Replikation von PostgreSQL an, aktuelle und zukünftige gespeicherte generierte Spalten der Publication-Tabellen zu veröffentlichen.
- Geben Sie eine Tabellenspaltenliste an, um ausdrücklich zu benennen, welche gespeicherten generierten Spalten veröffentlicht werden.

> **Hinweis**
>
> Bei der Bestimmung, welche Tabellenspalten veröffentlicht werden, hat eine Spaltenliste Vorrang und überschreibt die Wirkung des Parameters `publish_generated_columns`.

Die folgende Tabelle fasst das Verhalten zusammen, wenn generierte Spalten an der logischen Replikation beteiligt sind. Die Ergebnisse werden sowohl für den Fall gezeigt, dass das Veröffentlichen generierter Spalten nicht aktiviert ist, als auch für den Fall, dass es aktiviert ist.

**Tabelle 29.2. Zusammenfassung der Replikationsergebnisse**

| Generierte Spalten veröffentlichen? | Publisher-Tabellenspalte | Subscriber-Tabellenspalte | Ergebnis |
| --- | --- | --- | --- |
| Nein | `GENERATED` | `GENERATED` | Die Publisher-Tabellenspalte wird nicht repliziert. Der Wert der generierten Spalte der Subscriber-Tabelle wird verwendet. |
| Nein | `GENERATED` | regulär | Die Publisher-Tabellenspalte wird nicht repliziert. Der Standardwert der regulären Spalte der Subscriber-Tabelle wird verwendet. |
| Nein | `GENERATED` | fehlt | Die Publisher-Tabellenspalte wird nicht repliziert. Es passiert nichts. |
| Ja | `GENERATED` | `GENERATED` | Fehler. Nicht unterstützt. |
| Ja | `GENERATED` | regulär | Der Wert der Publisher-Tabellenspalte wird in die Subscriber-Tabellenspalte repliziert. |
| Ja | `GENERATED` | fehlt | Fehler. Die Spalte wird als in der Subscriber-Tabelle fehlend gemeldet. |

> **Warnung**
>
> Derzeit werden Subscriptions nicht unterstützt, die mehrere Publications umfassen, in denen dieselbe Tabelle mit unterschiedlichen Spaltenlisten veröffentlicht wurde. Siehe [Abschnitt 29.5](29_Logische_Replikation.md#295-spaltenlisten).
>
> Dieselbe Situation kann auftreten, wenn eine Publication generierte Spalten veröffentlicht, während eine andere Publication in derselben Subscription für dieselbe Tabelle keine generierten Spalten veröffentlicht.

> **Hinweis**
>
> Wenn der Subscriber eine Version vor 18 verwendet, kopiert die anfängliche Tabellensynchronisierung keine generierten Spalten, selbst wenn sie im Publisher definiert sind.

## 29.7. Konflikte

Logische Replikation verhält sich ähnlich wie normale DML-Operationen: Daten werden aktualisiert, selbst wenn sie lokal auf dem Subscriber-Knoten geändert wurden. Wenn eingehende Daten eine Einschränkung verletzen, stoppt die Replikation. Dies wird als Konflikt bezeichnet. Beim Replizieren von `UPDATE`- oder `DELETE`-Operationen gelten fehlende Daten ebenfalls als Konflikt, führen aber nicht zu einem Fehler; solche Operationen werden einfach übersprungen.

In den folgenden Konfliktfällen wird zusätzliche Protokollierung ausgelöst, und Konfliktstatistiken werden gesammelt, die in der Sicht `pg_stat_subscription_stats` angezeigt werden:

- `insert_exists`: Eine Zeile wird eingefügt, die eine eindeutige Einschränkung `NOT DEFERRABLE` verletzt. Beachten Sie, dass `track_commit_timestamp` auf dem Subscriber aktiviert sein sollte, um Details zu Ursprung und Commit-Zeitstempel des konfliktverursachenden Schlüssels zu protokollieren. In diesem Fall wird ein Fehler ausgelöst, bis der Konflikt manuell behoben wurde.

- `update_origin_differs`: Eine Zeile wird aktualisiert, die zuvor von einem anderen Ursprung geändert wurde. Dieser Konflikt kann nur erkannt werden, wenn `track_commit_timestamp` auf dem Subscriber aktiviert ist. Derzeit wird das Update unabhängig vom Ursprung der lokalen Zeile immer angewendet.

- `update_exists`: Der aktualisierte Wert einer Zeile verletzt eine eindeutige Einschränkung `NOT DEFERRABLE`. Beachten Sie, dass `track_commit_timestamp` auf dem Subscriber aktiviert sein sollte, um Details zu Ursprung und Commit-Zeitstempel des konfliktverursachenden Schlüssels zu protokollieren. In diesem Fall wird ein Fehler ausgelöst, bis der Konflikt manuell behoben wurde. Beachten Sie außerdem: Wenn beim Aktualisieren einer partitionierten Tabelle der aktualisierte Zeilenwert eine andere Partitionseinschränkung erfüllt und die Zeile dadurch in eine neue Partition eingefügt wird, kann der Konflikt `insert_exists` auftreten, falls die neue Zeile eine eindeutige Einschränkung `NOT DEFERRABLE` verletzt.

- `update_missing`: Die zu aktualisierende Zeile wurde nicht gefunden. Das Update wird in diesem Szenario einfach übersprungen.

- `delete_origin_differs`: Eine Zeile wird gelöscht, die zuvor von einem anderen Ursprung geändert wurde. Dieser Konflikt kann nur erkannt werden, wenn `track_commit_timestamp` auf dem Subscriber aktiviert ist. Derzeit wird das Delete unabhängig vom Ursprung der lokalen Zeile immer angewendet.

- `delete_missing`: Die zu löschende Zeile wurde nicht gefunden. Das Delete wird in diesem Szenario einfach übersprungen.

- `multiple_unique_conflicts`: Beim Einfügen oder Aktualisieren einer Zeile werden mehrere eindeutige Einschränkungen `NOT DEFERRABLE` verletzt. Um Details zu Ursprung und Commit-Zeitstempel der konfliktverursachenden Schlüssel zu protokollieren, stellen Sie sicher, dass `track_commit_timestamp` auf dem Subscriber aktiviert ist. In diesem Fall wird ein Fehler ausgelöst, bis der Konflikt manuell behoben wurde.

Beachten Sie, dass es weitere Konfliktszenarien gibt, etwa Verletzungen von Ausschlusseinschränkungen. Derzeit werden dafür keine zusätzlichen Details im Log bereitgestellt.

Das Logformat für Konflikte der logischen Replikation lautet:

```text
LOG: conflict detected on relation "schemaname.tablename":
 conflict=conflict_type
DETAIL: detailed_explanation.
{detail_values [; ... ]}.
```

Dabei ist `detail_values` eines von:

```text
Key (column_name [, ...])=(column_value [, ...])
existing local row [(column_name [, ...])=](column_value [, ...])
remote row [(column_name [, ...])=](column_value [, ...])
replica identity {(column_name [, ...])=(column_value [, ...])
 | full [(column_name [, ...])=](column_value [, ...])}
```

Das Log liefert die folgenden Informationen:

`LOG`

- `schemaname.tablename` identifiziert die lokale Relation, die am Konflikt beteiligt ist.
- `conflict_type` ist der Typ des aufgetretenen Konflikts, zum Beispiel `insert_exists` oder `update_exists`.

`DETAIL`

- `detailed_explanation` enthält, sofern verfügbar, Ursprung, Transaktions-ID und Commit-Zeitstempel der Transaktion, die die vorhandene lokale Zeile geändert hat.
- Der Abschnitt `Key` enthält die Schlüsselwerte der lokalen Zeile, die bei den Konflikten `insert_exists`, `update_exists` oder `multiple_unique_conflicts` eine eindeutige Einschränkung verletzt hat.
- Der Abschnitt `existing local row` enthält die lokale Zeile, wenn ihr Ursprung bei `update_origin_differs`- oder `delete_origin_differs`-Konflikten von der entfernten Zeile abweicht oder wenn der Schlüsselwert bei `insert_exists`-, `update_exists`- oder `multiple_unique_conflicts`-Konflikten mit der entfernten Zeile kollidiert.
- Der Abschnitt `remote row` enthält die neue Zeile aus der entfernten Insert- oder Update-Operation, die den Konflikt verursacht hat. Beachten Sie, dass bei einer Update-Operation der Spaltenwert der neuen Zeile `null` ist, wenn der Wert unverändert und toasted ist.
- Der Abschnitt `replica identity` enthält die Replica-Identity-Schlüsselwerte, die verwendet wurden, um nach der vorhandenen lokalen Zeile zu suchen, die aktualisiert oder gelöscht werden soll. Dies kann den vollständigen Zeilenwert enthalten, wenn die lokale Relation mit `REPLICA IDENTITY FULL` markiert ist.
- `column_name` ist der Spaltenname. Für `existing local row`, `remote row` und `replica identity full` werden Spaltennamen nur protokolliert, wenn dem Benutzer das Recht fehlt, auf alle Spalten der Tabelle zuzugreifen. Wenn Spaltennamen vorhanden sind, erscheinen sie in derselben Reihenfolge wie die entsprechenden Spaltenwerte.
- `column_value` ist der Spaltenwert. Große Spaltenwerte werden auf 64 Byte gekürzt.
- Beachten Sie, dass beim Konflikt `multiple_unique_conflicts` mehrere Zeilen für `detailed_explanation` und `detail_values` erzeugt werden, die jeweils die Konfliktinformationen zu unterschiedlichen eindeutigen Einschränkungen beschreiben.

Operationen der logischen Replikation werden mit den Rechten der Rolle ausgeführt, der die Subscription gehört. Fehlende Berechtigungen auf Zieltabellen verursachen Replikationskonflikte, ebenso aktivierte Row-Level-Security auf Zieltabellen, denen der Subscription-Eigentümer unterliegt, unabhängig davon, ob eine Policy das replizierte `INSERT`, `UPDATE`, `DELETE` oder `TRUNCATE` normalerweise ablehnen würde. Diese Einschränkung bei Row-Level-Security könnte in einer zukünftigen PostgreSQL-Version aufgehoben werden.

Ein Konflikt, der einen Fehler erzeugt, stoppt die Replikation; der Benutzer muss ihn manuell beheben. Details zum Konflikt finden sich im Serverlog des Subscribers.

Die Behebung kann entweder dadurch erfolgen, dass Daten oder Berechtigungen auf dem Subscriber so geändert werden, dass sie nicht mit der eingehenden Änderung kollidieren, oder indem die Transaktion übersprungen wird, die mit den vorhandenen Daten kollidiert. Wenn ein Konflikt einen Fehler erzeugt, fährt die Replikation nicht fort, und der Worker für logische Replikation gibt eine Meldung der folgenden Art in das Serverlog des Subscribers aus:

```text
ERROR: conflict detected on relation "public.test":
 conflict=insert_exists
DETAIL: Key already exists in unique index "t_pkey", which
 was modified locally in transaction 740 at 2024-06-26
 10:47:04.727375+08.
Key (c)=(1); existing local row (1, 'local'); remote row (1,
 'remote').
CONTEXT: processing remote data for replication origin "pg_16395"
 during "INSERT" for replication target relation "public.test" in
 transaction 725 finished at 0/14C0378
```

Die LSN der Transaktion, die die einschränkungsverletzende Änderung enthält, und der Name des Replikationsursprungs können dem Serverlog entnommen werden, im obigen Fall LSN `0/14C0378` und Replikationsursprung `pg_16395`. Die Transaktion, die den Konflikt erzeugt hat, kann mit `ALTER SUBSCRIPTION ... SKIP` und der Finish-LSN übersprungen werden, also beispielsweise mit LSN `0/14C0378`. Die Finish-LSN kann eine LSN sein, an der die Transaktion auf dem Publisher committet oder vorbereitet wurde. Alternativ kann die Transaktion auch durch Aufruf der Funktion `pg_replication_origin_advance()` übersprungen werden. Vor Verwendung dieser Funktion muss die Subscription vorübergehend mit `ALTER SUBSCRIPTION ... DISABLE` deaktiviert werden, oder die Subscription kann mit der Option `disable_on_error` verwendet werden. Danach können Sie `pg_replication_origin_advance()` mit `node_name`, also `pg_16395`, und der nächsten LSN nach der Finish-LSN, also `0/14C0379`, verwenden. Die aktuelle Position von Ursprüngen kann in der Systemsicht `pg_replication_origin_status` eingesehen werden.

Beachten Sie, dass das Überspringen der gesamten Transaktion auch Änderungen überspringt, die möglicherweise keine Einschränkung verletzen. Dadurch kann der Subscriber leicht inkonsistent werden. Zusätzliche Details zu konfliktverursachenden Zeilen, etwa ihr Ursprung und Commit-Zeitstempel, sind in der `DETAIL`-Zeile des Logs sichtbar. Diese Informationen sind jedoch nur verfügbar, wenn `track_commit_timestamp` auf dem Subscriber aktiviert ist. Benutzer können diese Informationen verwenden, um zu entscheiden, ob sie die lokale Änderung behalten oder die entfernte Änderung übernehmen wollen. Im obigen Log zeigt die `DETAIL`-Zeile zum Beispiel an, dass die vorhandene Zeile lokal geändert wurde. Benutzer können manuell eine Remote-Change-Wins-Auflösung durchführen.

Wenn der Streaming-Modus `parallel` ist, wird die Finish-LSN fehlgeschlagener Transaktionen möglicherweise nicht protokolliert. In diesem Fall kann es erforderlich sein, den Streaming-Modus auf `on` oder `off` zu ändern und dieselben Konflikte erneut auszulösen, damit die Finish-LSN der fehlgeschlagenen Transaktion in das Serverlog geschrieben wird. Zur Verwendung der Finish-LSN siehe `ALTER SUBSCRIPTION ... SKIP`.

## 29.8. Einschränkungen

Logische Replikation hat derzeit die folgenden Einschränkungen oder fehlenden Funktionen. Diese könnten in zukünftigen Versionen behoben werden.

- Das Datenbankschema und DDL-Befehle werden nicht repliziert. Das anfängliche Schema kann von Hand mit `pg_dump --schema-only` kopiert werden. Spätere Schemaänderungen müssen manuell synchron gehalten werden. Beachten Sie jedoch, dass die Schemata auf beiden Seiten nicht absolut identisch sein müssen. Logische Replikation ist robust, wenn sich Schemadefinitionen in einer laufenden Datenbank ändern: Wenn das Schema auf dem Publisher geändert wird und replizierte Daten beim Subscriber eintreffen, aber nicht zum Tabellenschema passen, meldet die Replikation Fehler, bis das Schema aktualisiert wurde. In vielen Fällen können vorübergehende Fehler vermieden werden, indem additive Schemaänderungen zuerst auf dem Subscriber angewendet werden.
- Sequenzdaten werden nicht repliziert. Daten in `serial`- oder Identity-Spalten, die auf Sequenzen basieren, werden natürlich als Teil der Tabelle repliziert, aber die Sequenz selbst zeigt auf dem Subscriber weiterhin den Startwert. Wenn der Subscriber als schreibgeschützte Datenbank verwendet wird, ist dies normalerweise kein Problem. Wenn jedoch eine Art Switchover oder Failover zur Subscriber-Datenbank vorgesehen ist, müssen die Sequenzen auf die neuesten Werte aktualisiert werden, entweder durch Kopieren der aktuellen Daten vom Publisher, vielleicht mit `pg_dump`, oder durch Ermitteln eines ausreichend hohen Werts aus den Tabellen selbst.
- Die Replikation von `TRUNCATE`-Befehlen wird unterstützt, aber beim Trunkieren von Tabellengruppen, die durch Fremdschlüssel verbunden sind, ist Vorsicht nötig. Beim Replizieren einer Truncate-Aktion trunkieren Subscriber dieselbe Tabellengruppe, die auf dem Publisher trunkiert wurde, entweder ausdrücklich angegeben oder implizit über `CASCADE` gesammelt, abzüglich Tabellen, die nicht Teil der Subscription sind. Das funktioniert korrekt, wenn alle betroffenen Tabellen Teil derselben Subscription sind. Wenn jedoch einige auf dem Subscriber zu trunkierende Tabellen Fremdschlüsselverbindungen zu Tabellen haben, die nicht Teil derselben oder irgendeiner Subscription sind, schlägt die Anwendung der Truncate-Aktion auf dem Subscriber fehl.
- Große Objekte (Large Objects, siehe [Kapitel 33](33_Große_Objekte.md)) werden nicht repliziert. Dafür gibt es keine Umgehung, außer Daten in normalen Tabellen zu speichern.
- Replikation wird nur für Tabellen unterstützt, einschließlich partitionierter Tabellen. Versuche, andere Relationstypen wie Views, materialisierte Views oder Foreign Tables zu replizieren, führen zu einem Fehler.
- Bei der Replikation zwischen partitionierten Tabellen stammt die tatsächliche Replikation standardmäßig von den Leaf-Partitionen auf dem Publisher; daher müssen Partitionen auf dem Publisher auch auf dem Subscriber als gültige Zieltabellen existieren. Sie können selbst Leaf-Partitionen sein, weiter unterpartitioniert sein oder sogar unabhängige Tabellen sein. Publications können auch angeben, dass Änderungen mit der Identität und dem Schema der partitionierten Root-Tabelle repliziert werden sollen statt mit denen der einzelnen Leaf-Partitionen, in denen die Änderungen tatsächlich entstehen; siehe den Parameter `publish_via_partition_root` von `CREATE PUBLICATION`.
- Bei Verwendung von `REPLICA IDENTITY FULL` auf veröffentlichten Tabellen ist wichtig zu beachten, dass `UPDATE`- und `DELETE`-Operationen auf Subscribern nicht angewendet werden können, wenn die Tabellen Attribute mit Datentypen wie `point` oder `box` enthalten, die keine Standard-Operatorklasse für B-Tree oder Hash haben. Diese Einschränkung kann jedoch überwunden werden, indem sichergestellt wird, dass die Tabelle einen Primärschlüssel oder eine Replica Identity definiert hat.

## 29.9. Architektur

Logische Replikation ist mit einer Architektur aufgebaut, die der physischen Streaming-Replikation ähnelt (siehe Abschnitt 26.2.5). Sie wird durch WAL-Sender- und Apply-Prozesse implementiert. Der WAL-Sender-Prozess startet die logische Decodierung des WAL, beschrieben in [Kapitel 47](47_Logische_Decodierung.md), und lädt das Standard-Ausgabe-Plugin für logische Decodierung (`pgoutput`). Das Plugin wandelt die aus dem WAL gelesenen Änderungen in das Protokoll der logischen Replikation um (siehe [Abschnitt 54.5](54_Frontend_Backend_Protokoll.md#545-logisches-streamingreplikationsprotokoll)) und filtert die Daten gemäß der Publication-Spezifikation. Die Daten werden dann kontinuierlich mit dem Streaming-Replikationsprotokoll an den Apply-Worker übertragen, der die Daten lokalen Tabellen zuordnet und die einzelnen Änderungen beim Empfang in der korrekten transaktionalen Reihenfolge anwendet.

Der Apply-Prozess in der Subscriber-Datenbank läuft immer mit `session_replication_role` auf `replica`. Das bedeutet, dass Trigger und Regeln auf einem Subscriber standardmäßig nicht ausgelöst werden. Benutzer können optional mit dem Befehl `ALTER TABLE` und den Klauseln `ENABLE TRIGGER` und `ENABLE RULE` Trigger und Regeln für eine Tabelle aktivieren.

Der Apply-Prozess der logischen Replikation löst derzeit nur Zeilentrigger aus, keine Statement-Trigger. Die anfängliche Tabellensynchronisierung wird jedoch wie ein `COPY`-Befehl implementiert und löst daher sowohl Zeilen- als auch Statement-Trigger für `INSERT` aus.

### 29.9.1. Anfangs-Snapshot

Die Anfangsdaten in vorhandenen abonnierten Tabellen werden in parallelen Instanzen einer speziellen Art von Apply-Prozess gesnapshottet und kopiert. Diese speziellen Apply-Prozesse sind dedizierte Worker für die Tabellensynchronisierung, die für jede zu synchronisierende Tabelle gestartet werden. Jeder Tabellensynchronisierungsprozess erstellt seinen eigenen Replikationsslot und kopiert die vorhandenen Daten. Sobald die Kopie abgeschlossen ist, wird der Tabelleninhalt für andere Backends sichtbar. Nachdem vorhandene Daten kopiert wurden, wechselt der Worker in den Synchronisierungsmodus. Dieser stellt sicher, dass die Tabelle in einen mit dem Haupt-Apply-Prozess synchronisierten Zustand gebracht wird, indem alle Änderungen, die während der anfänglichen Datenkopie aufgetreten sind, mit normaler logischer Replikation gestreamt werden. Während dieser Synchronisierungsphase werden Änderungen in derselben Reihenfolge angewendet und committet, in der sie auf dem Publisher aufgetreten sind. Sobald die Synchronisierung abgeschlossen ist, wird die Kontrolle über die Replikation der Tabelle an den Haupt-Apply-Prozess zurückgegeben, wo die Replikation normal fortgesetzt wird.

> **Hinweis**
>
> Der Publication-Parameter `publish` beeinflusst nur, welche DML-Operationen repliziert werden. Die anfängliche Datensynchronisierung berücksichtigt diesen Parameter beim Kopieren vorhandener Tabellendaten nicht.

> **Hinweis**
>
> Wenn ein Worker für die Tabellensynchronisierung während des Kopierens fehlschlägt, erkennt der Apply-Worker den Fehler und startet den Worker für die Tabellensynchronisierung erneut, um den Synchronisierungsprozess fortzusetzen. Dieses Verhalten stellt sicher, dass vorübergehende Fehler die Replikationskonfiguration nicht dauerhaft unterbrechen. Siehe auch `wal_retrieve_retry_interval`.

## 29.10. Überwachung

Da logische Replikation auf einer ähnlichen Architektur wie physische Streaming-Replikation basiert, ähnelt die Überwachung auf einem Publication-Knoten der Überwachung eines Primärservers der physischen Replikation (siehe Abschnitt 26.2.5.2).

Die Überwachungsinformationen zur Subscription sind in `pg_stat_subscription` sichtbar. Diese Sicht enthält eine Zeile für jeden Subscription-Worker. Eine Subscription kann je nach Zustand null oder mehr aktive Subscription-Worker haben.

Normalerweise läuft für eine aktivierte Subscription ein einzelner Apply-Prozess. Eine deaktivierte oder abgestürzte Subscription hat in dieser Sicht null Zeilen. Wenn die anfängliche Datensynchronisierung einer Tabelle läuft, gibt es zusätzliche Worker für die zu synchronisierenden Tabellen. Wenn die Streaming-Transaktion parallel angewendet wird, kann es außerdem zusätzliche parallele Apply-Worker geben.

## 29.11. Sicherheit

Die für die Replikationsverbindung verwendete Rolle muss das Attribut `REPLICATION` haben oder Superuser sein. Wenn der Rolle `SUPERUSER` und `BYPASSRLS` fehlen, können Row-Security-Policies des Publishers ausgeführt werden. Wenn die Rolle nicht allen Tabelleneigentümern vertraut, nehmen Sie `options=-crow_security=off` in die Verbindungszeichenfolge auf; wenn ein Tabelleneigentümer dann eine Row-Security-Policy hinzufügt, führt diese Einstellung dazu, dass die Replikation anhält, statt die Policy auszuführen. Der Zugriff für die Rolle muss in `pg_hba.conf` konfiguriert sein, und sie muss das Attribut `LOGIN` haben.

Um die anfänglichen Tabellendaten kopieren zu können, muss die für die Replikationsverbindung verwendete Rolle das Privileg `SELECT` auf einer veröffentlichten Tabelle haben oder Superuser sein.

Zum Erstellen einer Publication muss der Benutzer das Privileg `CREATE` in der Datenbank haben.

Um Tabellen zu einer Publication hinzuzufügen, muss der Benutzer Eigentümerrechte an der Tabelle haben. Um alle Tabellen in einem Schema zu einer Publication hinzuzufügen, muss der Benutzer Superuser sein. Um eine Publication zu erstellen, die automatisch alle Tabellen oder alle Tabellen in einem Schema veröffentlicht, muss der Benutzer Superuser sein.

Derzeit gibt es keine Privilegien auf Publications. Jede Subscription, die eine Verbindung herstellen kann, kann auf jede Publication zugreifen. Wenn Sie also bestimmte Informationen vor bestimmten Subscribern verbergen möchten, etwa durch Zeilenfilter oder Spaltenlisten oder indem nicht die ganze Tabelle zur Publication hinzugefügt wird, beachten Sie, dass andere Publications in derselben Datenbank dieselben Informationen offenlegen könnten. Publication-Privilegien könnten PostgreSQL in Zukunft hinzugefügt werden, um feingranularere Zugriffskontrolle zu ermöglichen.

Zum Erstellen einer Subscription muss der Benutzer die Privilegien der Rolle `pg_create_subscription` sowie `CREATE`-Privilegien auf der Datenbank haben.

Der Apply-Prozess der Subscription läuft auf Sitzungsebene mit den Privilegien des Subscription-Eigentümers. Beim Ausführen einer Insert-, Update-, Delete- oder Truncate-Operation auf einer bestimmten Tabelle wechselt er jedoch zur Rolle des Tabelleneigentümers und führt die Operation mit dessen Privilegien aus. Das bedeutet, dass der Subscription-Eigentümer in der Lage sein muss, `SET ROLE` auf jede Rolle auszuführen, der eine replizierte Tabelle gehört.

Wenn die Subscription mit `run_as_owner = true` konfiguriert wurde, erfolgt kein Benutzerwechsel. Stattdessen werden alle Operationen mit den Berechtigungen des Subscription-Eigentümers ausgeführt. In diesem Fall benötigt der Subscription-Eigentümer nur Privilegien für `SELECT`, `INSERT`, `UPDATE` und `DELETE` auf der Zieltabelle und keine Privilegien für `SET ROLE` auf den Tabelleneigentümer. Das bedeutet jedoch auch, dass jeder Benutzer, dem eine Tabelle gehört, in die repliziert wird, beliebigen Code mit den Privilegien des Subscription-Eigentümers ausführen kann. Er könnte dies zum Beispiel tun, indem er einfach einen Trigger an eine seiner Tabellen anhängt. Da es normalerweise unerwünscht ist, einer Rolle zu erlauben, frei die Privilegien einer anderen anzunehmen, sollte diese Option vermieden werden, sofern Benutzersicherheit innerhalb der Datenbank eine Rolle spielt.

Auf dem Publisher werden Privilegien nur einmal zu Beginn einer Replikationsverbindung geprüft und nicht erneut, wenn jeder Änderungsdatensatz gelesen wird.

Auf dem Subscriber werden die Privilegien des Subscription-Eigentümers bei jeder angewendeten Transaktion erneut geprüft. Wenn ein Worker gerade eine Transaktion anwendet, während der Eigentümer der Subscription durch eine gleichzeitige Transaktion geändert wird, wird die Anwendung der aktuellen Transaktion mit den Privilegien des alten Eigentümers fortgesetzt.

## 29.12. Konfigurationseinstellungen

Logische Replikation erfordert mehrere gesetzte Konfigurationsoptionen. Diese Optionen sind nur auf einer Seite der Replikation relevant.

### 29.12.1. Publisher

`wal_level` muss auf `logical` gesetzt sein.

`max_replication_slots` muss mindestens auf die Anzahl der erwarteten verbindenden Subscriptions plus etwas Reserve für die Tabellensynchronisierung gesetzt werden.

Logische Replikationsslots werden außerdem durch `idle_replication_slot_timeout` beeinflusst.

`max_wal_senders` sollte mindestens auf denselben Wert wie `max_replication_slots` gesetzt werden, plus die Anzahl der physischen Replikate, die gleichzeitig verbunden sind.

Der WAL-Sender der logischen Replikation wird außerdem durch `wal_sender_timeout` beeinflusst.

### 29.12.2. Subscriber

`max_active_replication_origins` muss mindestens auf die Anzahl der Subscriptions gesetzt werden, die dem Subscriber hinzugefügt werden, plus etwas Reserve für die Tabellensynchronisierung.

`max_logical_replication_workers` muss mindestens auf die Anzahl der Subscriptions gesetzt werden, für Leader-Apply-Worker, plus etwas Reserve für Worker der Tabellensynchronisierung und parallele Apply-Worker.

`max_worker_processes` muss möglicherweise angepasst werden, um Replikations-Worker aufzunehmen, mindestens auf `max_logical_replication_workers + 1`. Beachten Sie, dass einige Erweiterungen und parallele Abfragen ebenfalls Worker-Slots aus `max_worker_processes` verwenden.

`max_sync_workers_per_subscription` steuert den Grad der Parallelität der anfänglichen Datenkopie während der Initialisierung der Subscription oder wenn neue Tabellen hinzugefügt werden.

`max_parallel_apply_workers_per_subscription` steuert den Grad der Parallelität für das Streaming laufender Transaktionen mit dem Subscription-Parameter `streaming = parallel`.

Worker der logischen Replikation werden außerdem durch `wal_receiver_timeout`, `wal_receiver_status_interval` und `wal_retrieve_retry_interval` beeinflusst.

## 29.13. Upgrade

Die Migration von Clustern der logischen Replikation ist nur möglich, wenn alle Mitglieder der alten logischen Replikationscluster Version 17.0 oder neuer verwenden.

### 29.13.1. Publisher-Upgrades vorbereiten

`pg_upgrade` versucht, logische Slots zu migrieren. Dadurch wird vermieden, dieselben logischen Slots auf dem neuen Publisher manuell definieren zu müssen. Die Migration logischer Slots wird nur unterstützt, wenn der alte Cluster Version 17.0 oder neuer verwendet. Logische Slots auf Clustern vor Version 17.0 werden stillschweigend ignoriert.

Bevor Sie mit dem Upgrade des Publisher-Clusters beginnen, stellen Sie sicher, dass die Subscription vorübergehend deaktiviert ist, indem Sie `ALTER SUBSCRIPTION ... DISABLE` ausführen. Aktivieren Sie die Subscription nach dem Upgrade wieder.

Es gibt einige Voraussetzungen, damit `pg_upgrade` die logischen Slots aktualisieren kann. Wenn diese nicht erfüllt sind, wird ein Fehler gemeldet.

- Der neue Cluster muss `wal_level` auf `logical` gesetzt haben.
- Der neue Cluster muss `max_replication_slots` auf einen Wert konfiguriert haben, der größer oder gleich der Anzahl der im alten Cluster vorhandenen Slots ist.
- Die Ausgabe-Plugins, auf die die Slots im alten Cluster verweisen, müssen im neuen PostgreSQL-Programmverzeichnis installiert sein.
- Der alte Cluster hat alle Transaktionen und Nachrichten der logischen Decodierung zu den Subscribern repliziert.
- Alle Slots im alten Cluster müssen verwendbar sein, das heißt, es darf keine Slots geben, bei denen `pg_replication_slots.conflicting` nicht `true` ist.
- Der neue Cluster darf keine permanenten logischen Slots haben, das heißt, es darf keine Slots geben, bei denen `pg_replication_slots.temporary` `false` ist.

### 29.13.2. Subscriber-Upgrades vorbereiten

Richten Sie die Subscriber-Konfigurationen im neuen Subscriber ein. `pg_upgrade` versucht, Subscription-Abhängigkeiten zu migrieren. Dazu gehören die Tabelleninformationen der Subscription im Systemkatalog `pg_subscription_rel` sowie der Replikationsursprung der Subscription. Dadurch kann die logische Replikation auf dem neuen Subscriber dort fortgesetzt werden, wo der alte Subscriber stand. Die Migration von Subscription-Abhängigkeiten wird nur unterstützt, wenn der alte Cluster Version 17.0 oder neuer verwendet. Subscription-Abhängigkeiten auf Clustern vor Version 17.0 werden stillschweigend ignoriert.

Es gibt einige Voraussetzungen, damit `pg_upgrade` die Subscriptions aktualisieren kann. Wenn diese nicht erfüllt sind, wird ein Fehler gemeldet.

- Alle Subscription-Tabellen im alten Subscriber sollten im Zustand `i` (initialize) oder `r` (ready) sein. Dies kann durch Prüfen von `pg_subscription_rel.srsubstate` verifiziert werden.
- Der Eintrag für den Replikationsursprung, der jeder Subscription entspricht, sollte im alten Cluster existieren. Dies kann durch Prüfen der Systemtabellen `pg_subscription` und `pg_replication_origin` ermittelt werden.
- Der neue Cluster muss `max_active_replication_origins` auf einen Wert konfiguriert haben, der größer oder gleich der Anzahl der im alten Cluster vorhandenen Subscriptions ist.

### 29.13.3. Cluster der logischen Replikation aktualisieren

Während ein Subscriber aktualisiert wird, können auf dem Publisher Schreiboperationen ausgeführt werden. Diese Änderungen werden zum Subscriber repliziert, sobald das Subscriber-Upgrade abgeschlossen ist.

> **Hinweis**
>
> Die Einschränkungen der logischen Replikation gelten auch für Upgrades von Clustern der logischen Replikation. Details finden Sie in Abschnitt 29.8.
>
> Die Voraussetzungen des Publisher-Upgrades gelten ebenfalls für Upgrades von Clustern der logischen Replikation. Details finden Sie in Abschnitt 29.13.1.
>
> Die Voraussetzungen des Subscriber-Upgrades gelten ebenfalls für Upgrades von Clustern der logischen Replikation. Details finden Sie in Abschnitt 29.13.2.

> **Warnung**
>
> Das Upgrade eines Clusters der logischen Replikation erfordert mehrere Schritte auf verschiedenen Knoten. Da nicht alle Operationen transaktional sind, wird empfohlen, Sicherungen wie in Abschnitt 25.3.2 beschrieben zu erstellen.

Die Schritte zum Aktualisieren der folgenden logischen Replikationscluster werden unten beschrieben:

- Folgen Sie den in Abschnitt 29.13.3.1 angegebenen Schritten, um einen logischen Replikationscluster mit zwei Knoten zu aktualisieren.
- Folgen Sie den in Abschnitt 29.13.3.2 angegebenen Schritten, um einen kaskadierten logischen Replikationscluster zu aktualisieren.
- Folgen Sie den in Abschnitt 29.13.3.3 angegebenen Schritten, um einen zirkulären logischen Replikationscluster mit zwei Knoten zu aktualisieren.

#### 29.13.3.1. Schritte zum Aktualisieren eines logischen Replikationsclusters mit zwei Knoten

Angenommen, der Publisher befindet sich auf `node1` und der Subscriber auf `node2`. Der Subscriber `node2` hat eine Subscription `sub1_node1_node2`, die Änderungen von `node1` abonniert.

1. Deaktivieren Sie auf `node2` alle Subscriptions, die Änderungen von `node1` abonnieren, mit `ALTER SUBSCRIPTION ... DISABLE`, zum Beispiel:

```sql
/* node2 # */ ALTER SUBSCRIPTION sub1_node1_node2 DISABLE;
```

2. Stoppen Sie den Publisher-Server auf `node1`, zum Beispiel:

```text
pg_ctl -D /opt/PostgreSQL/data1 stop
```

3. Initialisieren Sie die Instanz `data1_upgraded` mit der erforderlichen neueren Version.

4. Aktualisieren Sie den Server des Publishers `node1` auf die erforderliche neuere Version, zum Beispiel:

```text
pg_upgrade
        --old-datadir "/opt/PostgreSQL/postgres/17/data1"
        --new-datadir "/opt/PostgreSQL/postgres/18/data1_upgraded"
        --old-bindir "/opt/PostgreSQL/postgres/17/bin"
        --new-bindir "/opt/PostgreSQL/postgres/18/bin"
```

5. Starten Sie den aktualisierten Publisher-Server auf `node1`, zum Beispiel:

```text
pg_ctl -D /opt/PostgreSQL/data1_upgraded start -l logfile
```

6. Stoppen Sie den Subscriber-Server auf `node2`, zum Beispiel:

```text
pg_ctl -D /opt/PostgreSQL/data2 stop
```

7. Initialisieren Sie die Instanz `data2_upgraded` mit der erforderlichen neueren Version.

8. Aktualisieren Sie den Server des Subscribers `node2` auf die erforderliche neue Version, zum Beispiel:

```text
pg_upgrade
        --old-datadir "/opt/PostgreSQL/postgres/17/data2"
        --new-datadir "/opt/PostgreSQL/postgres/18/data2_upgraded"
        --old-bindir "/opt/PostgreSQL/postgres/17/bin"
        --new-bindir "/opt/PostgreSQL/postgres/18/bin"
```

9. Starten Sie den aktualisierten Subscriber-Server auf `node2`, zum Beispiel:

```text
pg_ctl -D /opt/PostgreSQL/data2_upgraded start -l logfile
```

10. Erstellen Sie auf `node2` alle Tabellen, die zwischen Schritt 1 und jetzt auf dem aktualisierten Publisher-Server `node1` erstellt wurden, zum Beispiel:

```sql
/* node2 # */ CREATE TABLE distributors (did integer PRIMARY KEY, name varchar(40));
```

11. Aktivieren Sie auf `node2` alle Subscriptions, die Änderungen von `node1` abonnieren, mit `ALTER SUBSCRIPTION ... ENABLE`, zum Beispiel:

```sql
/* node2 # */ ALTER SUBSCRIPTION sub1_node1_node2 ENABLE;
```

12. Aktualisieren Sie die Publications der Subscription von `node2` mit `ALTER SUBSCRIPTION ... REFRESH PUBLICATION`, zum Beispiel:

```sql
/* node2 # */ ALTER SUBSCRIPTION sub1_node1_node2 REFRESH PUBLICATION;
```

> **Hinweis**
>
> In den oben beschriebenen Schritten wird zuerst der Publisher aktualisiert, anschließend der Subscriber. Alternativ können Benutzer mit ähnlichen Schritten zuerst den Subscriber und danach den Publisher aktualisieren.

#### 29.13.3.2. Schritte zum Aktualisieren eines kaskadierten logischen Replikationsclusters

Angenommen, es gibt eine kaskadierte logische Replikationskonfiguration `node1->node2->node3`. Hier abonniert `node2` Änderungen von `node1`, und `node3` abonniert Änderungen von `node2`. `node2` hat eine Subscription `sub1_node1_node2`, die Änderungen von `node1` abonniert. `node3` hat eine Subscription `sub1_node2_node3`, die Änderungen von `node2` abonniert.

1. Deaktivieren Sie auf `node2` alle Subscriptions, die Änderungen von `node1` abonnieren, mit `ALTER SUBSCRIPTION ... DISABLE`.
2. Stoppen Sie den Server auf `node1`.
3. Initialisieren Sie die Instanz `data1_upgraded` mit der erforderlichen neueren Version.
4. Aktualisieren Sie den Server von `node1` auf die erforderliche neuere Version.
5. Starten Sie den aktualisierten Server auf `node1`.
6. Deaktivieren Sie auf `node3` alle Subscriptions, die Änderungen von `node2` abonnieren, mit `ALTER SUBSCRIPTION ... DISABLE`.
7. Stoppen Sie den Server auf `node2`.
8. Initialisieren Sie die Instanz `data2_upgraded` mit der erforderlichen neueren Version.
9. Aktualisieren Sie den Server von `node2` auf die erforderliche neue Version.
10. Starten Sie den aktualisierten Server auf `node2`.
11. Erstellen Sie auf `node2` alle Tabellen, die zwischen Schritt 1 und jetzt auf dem aktualisierten Publisher-Server `node1` erstellt wurden.
12. Aktivieren Sie auf `node2` alle Subscriptions, die Änderungen von `node1` abonnieren, mit `ALTER SUBSCRIPTION ... ENABLE`.
13. Aktualisieren Sie die Publications der Subscription von `node2` mit `ALTER SUBSCRIPTION ... REFRESH PUBLICATION`.
14. Stoppen Sie den Server auf `node3`.
15. Initialisieren Sie die Instanz `data3_upgraded` mit der erforderlichen neueren Version.
16. Aktualisieren Sie den Server von `node3` auf die erforderliche neue Version.
17. Starten Sie den aktualisierten Server auf `node3`.
18. Erstellen Sie auf `node3` alle Tabellen, die zwischen Schritt 6 und jetzt auf dem aktualisierten `node2` erstellt wurden.
19. Aktivieren Sie auf `node3` alle Subscriptions, die Änderungen von `node2` abonnieren, mit `ALTER SUBSCRIPTION ... ENABLE`.
20. Aktualisieren Sie die Publications der Subscription von `node3` mit `ALTER SUBSCRIPTION ... REFRESH PUBLICATION`.

Die Beispielbefehle entsprechen den Befehlen im vorherigen Abschnitt, angepasst auf die jeweiligen Knoten und Datenverzeichnisse, etwa `pg_ctl -D /opt/PostgreSQL/data1 stop`, `pg_ctl -D /opt/PostgreSQL/data2_upgraded start -l logfile` sowie `ALTER SUBSCRIPTION sub1_node2_node3 REFRESH PUBLICATION`.

#### 29.13.3.3. Schritte zum Aktualisieren eines zirkulären logischen Replikationsclusters mit zwei Knoten

Angenommen, es gibt eine zirkuläre logische Replikationskonfiguration `node1->node2` und `node2->node1`. Hier abonniert `node2` Änderungen von `node1`, und `node1` abonniert Änderungen von `node2`. `node1` hat eine Subscription `sub1_node2_node1`, die Änderungen von `node2` abonniert. `node2` hat eine Subscription `sub1_node1_node2`, die Änderungen von `node1` abonniert.

1. Deaktivieren Sie auf `node2` alle Subscriptions, die Änderungen von `node1` abonnieren, mit `ALTER SUBSCRIPTION ... DISABLE`.
2. Stoppen Sie den Server auf `node1`.
3. Initialisieren Sie die Instanz `data1_upgraded` mit der erforderlichen neueren Version.
4. Aktualisieren Sie den Server von `node1` auf die erforderliche neuere Version.
5. Starten Sie den aktualisierten Server auf `node1`.
6. Aktivieren Sie auf `node2` alle Subscriptions, die Änderungen von `node1` abonnieren, mit `ALTER SUBSCRIPTION ... ENABLE`.
7. Erstellen Sie auf `node1` alle Tabellen, die zwischen Schritt 1 und jetzt auf `node2` erstellt wurden.
8. Aktualisieren Sie die Publications der Subscription von `node1` mit `ALTER SUBSCRIPTION ... REFRESH PUBLICATION`, um anfängliche Tabellendaten von `node2` zu kopieren.
9. Deaktivieren Sie auf `node1` alle Subscriptions, die Änderungen von `node2` abonnieren, mit `ALTER SUBSCRIPTION ... DISABLE`.
10. Stoppen Sie den Server auf `node2`.
11. Initialisieren Sie die Instanz `data2_upgraded` mit der erforderlichen neueren Version.
12. Aktualisieren Sie den Server von `node2` auf die erforderliche neue Version.
13. Starten Sie den aktualisierten Server auf `node2`.
14. Aktivieren Sie auf `node1` alle Subscriptions, die Änderungen von `node2` abonnieren, mit `ALTER SUBSCRIPTION ... ENABLE`.
15. Erstellen Sie auf `node2` alle Tabellen, die zwischen Schritt 9 und jetzt auf dem aktualisierten `node1` erstellt wurden.
16. Aktualisieren Sie die Publications der Subscription von `node2` mit `ALTER SUBSCRIPTION ... REFRESH PUBLICATION`, um anfängliche Tabellendaten von `node1` zu kopieren.

Die Beispielbefehle entsprechen den oben gezeigten Formen, etwa:

```sql
/* node2 # */ ALTER SUBSCRIPTION sub1_node1_node2 DISABLE;
/* node1 # */ ALTER SUBSCRIPTION sub1_node2_node1 REFRESH PUBLICATION;
/* node2 # */ ALTER SUBSCRIPTION sub1_node1_node2 REFRESH PUBLICATION;
```

```text
pg_ctl -D /opt/PostgreSQL/data1 stop
pg_ctl -D /opt/PostgreSQL/data1_upgraded start -l logfile
pg_ctl -D /opt/PostgreSQL/data2 stop
pg_ctl -D /opt/PostgreSQL/data2_upgraded start -l logfile
```

## 29.14. Schnelleinrichtung

Setzen Sie zuerst die Konfigurationsoptionen in `postgresql.conf`:

```text
wal_level = logical
```

Die anderen erforderlichen Einstellungen haben Standardwerte, die für eine einfache Einrichtung ausreichen.

`pg_hba.conf` muss angepasst werden, um Replikation zu erlauben. Die Werte hängen von Ihrer tatsächlichen Netzwerkkonfiguration und dem Benutzer ab, den Sie für die Verbindung verwenden möchten:

```text
host          all         repuser            0.0.0.0/0             md5
```

Dann in der Publisher-Datenbank:

```sql
CREATE PUBLICATION mypub FOR TABLE users, departments;
```

Und in der Subscriber-Datenbank:

```sql
CREATE SUBSCRIPTION mysub CONNECTION 'dbname=foo host=bar user=repuser' PUBLICATION mypub;
```

Dies startet den Replikationsprozess, der die anfänglichen Tabelleninhalte der Tabellen `users` und `departments` synchronisiert und anschließend beginnt, inkrementelle Änderungen an diesen Tabellen zu replizieren.
