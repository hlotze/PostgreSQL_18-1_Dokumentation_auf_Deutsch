# 67. Transaktionsverarbeitung

Dieses Kapitel gibt einen Überblick über die Interna des Transaktionsverwaltungssystems von PostgreSQL. Das Wort Transaktion wird häufig als `xact` abgekürzt.

## 67.1. Transaktionen und Identifikatoren

Transaktionen können mit `BEGIN` oder `START TRANSACTION` explizit begonnen und mit `COMMIT` oder `ROLLBACK` beendet werden. SQL-Anweisungen außerhalb expliziter Transaktionen verwenden automatisch Transaktionen, die jeweils nur eine einzelne Anweisung umfassen.

Jede Transaktion wird durch eine eindeutige `VirtualTransactionId` identifiziert, auch `virtualXID` oder `vxid` genannt. Sie besteht aus der Prozessnummer eines Backends, der `procNumber`, und einer jedem Backend lokal fortlaufend zugewiesenen Nummer, der `localXID`. Die virtuelle Transaktions-ID `4/12532` hat zum Beispiel die `procNumber` `4` und die `localXID` `12532`.

Nicht-virtuelle `TransactionId`s, kurz `xid`, zum Beispiel `278394`, werden Transaktionen fortlaufend aus einem globalen Zähler zugewiesen, der von allen Datenbanken innerhalb des PostgreSQL-Clusters gemeinsam verwendet wird. Diese Zuweisung geschieht, wenn eine Transaktion zum ersten Mal in die Datenbank schreibt. Das bedeutet, dass `xid`s mit kleineren Nummern vor `xid`s mit größeren Nummern zu schreiben begonnen haben. Die Reihenfolge, in der Transaktionen ihren ersten Datenbankschreibzugriff ausführen, kann sich von der Reihenfolge unterscheiden, in der die Transaktionen begonnen wurden, insbesondere wenn eine Transaktion zunächst nur lesende Anweisungen ausgeführt hat.

Der interne Transaktions-ID-Typ `xid` ist 32 Bit breit und läuft alle 4 Milliarden Transaktionen über. Bei jedem Überlauf wird eine 32-Bit-Epoche erhöht. Außerdem gibt es den 64-Bit-Typ `xid8`, der diese Epoche einschließt und daher während der Lebensdauer einer Installation nicht überläuft; er kann durch Cast in `xid` umgewandelt werden. Die Funktionen in Tabelle 9.84 geben `xid8`-Werte zurück. `xid`s dienen als Grundlage für PostgreSQLs MVCC-Mechanismus für Nebenläufigkeit und für Streaming-Replikation.

Wenn eine Top-Level-Transaktion mit einer nicht-virtuellen `xid` committet, wird sie im Verzeichnis `pg_xact` als committed markiert. Zusätzliche Informationen werden im Verzeichnis `pg_commit_ts` aufgezeichnet, wenn `track_commit_timestamp` aktiviert ist.

Neben `vxid` und `xid` werden vorbereiteten Transaktionen auch globale Transaktionsidentifikatoren (`GID`) zugewiesen. `GID`s sind String-Literale mit bis zu 200 Byte Länge, die unter den aktuell vorbereiteten Transaktionen eindeutig sein müssen. Die Zuordnung von `GID` zu `xid` wird in `pg_prepared_xacts` angezeigt.

## 67.2. Transaktionen und Locking

Die Transaktions-IDs aktuell laufender Transaktionen werden in `pg_locks` in den Spalten `virtualxid` und `transactionid` angezeigt. Nur lesende Transaktionen haben `virtualxid`s, aber `NULL`-`transactionid`s; bei schreibenden Transaktionen sind beide Spalten gesetzt.

Einige Lock-Typen warten auf `virtualxid`, andere auf `transactionid`. Row-Level-Lese- und Schreib-Locks werden direkt in den gesperrten Zeilen aufgezeichnet und können mit der Erweiterung `pgrowlocks` untersucht werden. Row-Level-Lese-Locks können außerdem die Zuweisung von MultiXact-IDs (`mxid`; siehe Abschnitt 24.1.5.1) erfordern.

## 67.3. Subtransaktionen

Subtransaktionen werden innerhalb von Transaktionen gestartet und erlauben es, große Transaktionen in kleinere Einheiten aufzuteilen. Subtransaktionen können committen oder abbrechen, ohne ihre übergeordneten Transaktionen zu beeinflussen; die übergeordneten Transaktionen können weiterlaufen. Dadurch lassen sich Fehler leichter behandeln, was ein verbreitetes Muster in der Anwendungsentwicklung ist. Das Wort Subtransaktion wird häufig als `subxact` abgekürzt.

Subtransaktionen können explizit mit dem Befehl `SAVEPOINT` gestartet werden, sie können aber auch auf andere Weise entstehen, etwa durch die `EXCEPTION`-Klausel von PL/pgSQL. PL/Python und PL/Tcl unterstützen ebenfalls explizite Subtransaktionen. Subtransaktionen können auch von anderen Subtransaktionen aus gestartet werden. Die Top-Level-Transaktion und ihre untergeordneten Subtransaktionen bilden eine Hierarchie oder einen Baum; deshalb wird die Haupttransaktion als Top-Level-Transaktion bezeichnet.

Wenn einer Subtransaktion eine nicht-virtuelle Transaktions-ID zugewiesen wird, wird ihre Transaktions-ID als `subxid` bezeichnet. Nur lesenden Subtransaktionen werden keine `subxid`s zugewiesen; sobald sie jedoch zu schreiben versuchen, erhalten sie eine. Dadurch erhalten auch alle Eltern dieser `subxid` bis einschließlich der Top-Level-Transaktion nicht-virtuelle Transaktions-IDs. PostgreSQL stellt sicher, dass die `xid` eines Elternknotens immer kleiner ist als alle `subxid`s seiner Kinder.

Die unmittelbare Eltern-`xid` jeder `subxid` wird im Verzeichnis `pg_subtrans` aufgezeichnet. Für Top-Level-`xid`s wird kein Eintrag angelegt, weil sie kein Elternobjekt haben; für nur lesende Subtransaktionen wird ebenfalls kein Eintrag angelegt.

Wenn eine Subtransaktion committet, gelten alle ihre committeten Kind-Subtransaktionen mit `subxid`s ebenfalls als in dieser Transaktion subcommitted. Wenn eine Subtransaktion abbricht, gelten auch alle ihre Kind-Subtransaktionen als abgebrochen.

Wenn eine Top-Level-Transaktion mit einer `xid` committet, werden alle ihre subcommitted Kind-Subtransaktionen ebenfalls dauerhaft als committed im Unterverzeichnis `pg_xact` aufgezeichnet. Wenn die Top-Level-Transaktion abbricht, werden auch alle ihre Subtransaktionen abgebrochen, selbst wenn sie bereits subcommitted waren.

Je mehr Subtransaktionen jede Transaktion offen hält, also nicht zurückgerollt oder freigegeben hat, desto größer ist der Verwaltungsaufwand. Bis zu 64 offene `subxid`s werden pro Backend im Shared Memory zwischengespeichert; danach steigt der Speicher-I/O-Aufwand deutlich, weil zusätzliche Nachschläge von `subxid`-Einträgen in `pg_subtrans` nötig werden.

## 67.4. Two-Phase-Transaktionen

PostgreSQL unterstützt ein Two-Phase-Commit-Protokoll (2PC), mit dem mehrere verteilte Systeme transaktional zusammenarbeiten können. Die Befehle dafür sind `PREPARE TRANSACTION`, `COMMIT PREPARED` und `ROLLBACK PREPARED`. Two-Phase-Transaktionen sind für externe Transaktionsverwaltungssysteme gedacht. PostgreSQL folgt den Merkmalen und dem Modell, die vom X/Open-XA-Standard vorgeschlagen werden, implementiert jedoch einige seltener verwendete Aspekte nicht.

Wenn der Benutzer `PREPARE TRANSACTION` ausführt, sind danach nur noch `COMMIT PREPARED` oder `ROLLBACK PREPARED` möglich. Im Allgemeinen ist dieser vorbereitete Zustand als sehr kurzlebig gedacht, doch externe Verfügbarkeitsprobleme können dazu führen, dass Transaktionen über längere Zeit in diesem Zustand bleiben. Kurzlebige vorbereitete Transaktionen werden nur im Shared Memory und im WAL gespeichert. Transaktionen, die Checkpoints überdauern, werden im Verzeichnis `pg_twophase` aufgezeichnet. Aktuell vorbereitete Transaktionen können mit `pg_prepared_xacts` untersucht werden.
