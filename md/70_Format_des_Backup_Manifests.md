# 70. Format des Backup-Manifests

Das von `pg_basebackup` erzeugte Backup-Manifest ist in erster Linie dafür gedacht, dass ein Backup mit `pg_verifybackup` geprüft werden kann. Es ist jedoch auch möglich, dass andere Werkzeuge die Backup-Manifestdatei lesen und die darin enthaltenen Informationen für eigene Zwecke verwenden. Deshalb beschreibt dieses Kapitel das Format der Backup-Manifestdatei.

Ein Backup-Manifest ist ein als UTF-8 codiertes JSON-Dokument. Obwohl JSON-Dokumente im Allgemeinen Unicode sein müssen, erlaubt PostgreSQL, dass die Datentypen `json` und `jsonb` mit jeder unterstützten Servercodierung verwendet werden. Für Backup-Manifeste gibt es keine entsprechende Ausnahme. Das JSON-Dokument ist immer ein Objekt; die in diesem Objekt vorhandenen Schlüssel werden im nächsten Abschnitt beschrieben.

## 70.1. Top-Level-Objekt des Backup-Manifests

Das JSON-Dokument des Backup-Manifests enthält die folgenden Schlüssel:

- `PostgreSQL-Backup-Manifest-Version`
  - Der zugehörige Wert ist eine Ganzzahl. Seit PostgreSQL 17 ist er `2`; in älteren Versionen ist er `1`.

- `System-Identifier`
  - Der Datenbanksystemidentifikator der PostgreSQL-Instanz, auf der das Backup erstellt wurde. Dieses Feld ist nur vorhanden, wenn `PostgreSQL-Backup-Manifest-Version` den Wert `2` hat.

- `Files`
  - Der zugehörige Wert ist immer eine Liste von Objekten, von denen jedes eine im Backup vorhandene Datei beschreibt. Für die WAL-Dateien, die benötigt werden, um das Backup zu verwenden, und für das Backup-Manifest selbst gibt es keine Einträge in dieser Liste. Die Struktur jedes Objekts in der Liste wird in [Abschnitt 70.2](70_Format_des_Backup_Manifests.md#702-dateiobjekt-des-backupmanifests) beschrieben.

- `WAL-Ranges`
  - Der zugehörige Wert ist immer eine Liste von Objekten, von denen jedes einen Bereich von WAL-Records beschreibt, der aus einer bestimmten Timeline lesbar sein muss, um das Backup verwenden zu können. Die Struktur dieser Objekte wird in [Abschnitt 70.3](70_Format_des_Backup_Manifests.md#703-walbereichsobjekt-des-backupmanifests) näher beschrieben.

- `Manifest-Checksum`
  - Dieser Schlüssel ist immer in der letzten Zeile der Backup-Manifestdatei vorhanden. Der zugehörige Wert ist eine SHA-256-Prüfsumme aller vorhergehenden Zeilen. Hier wird eine feste Prüfsummenmethode verwendet, damit Clients das Manifest inkrementell parsen können. Obwohl eine SHA-256-Prüfsumme deutlich teurer ist als eine CRC-32C-Prüfsumme, sollte das Manifest normalerweise klein genug sein, dass der zusätzliche Rechenaufwand keine große Rolle spielt.

## 70.2. Dateiobjekt des Backup-Manifests

Das Objekt, das eine einzelne Datei beschreibt, enthält entweder einen Schlüssel `Path` oder einen Schlüssel `Encoded-Path`. Normalerweise ist der Schlüssel `Path` vorhanden. Der zugehörige String-Wert ist der Pfad der Datei relativ zur Wurzel des Backup-Verzeichnisses. Dateien, die sich in einem benutzerdefinierten Tablespace befinden, haben Pfade, deren erste beiden Komponenten `pg_tblspc` und die OID des Tablespace sind. Wenn der Pfad kein in UTF-8 gültiger String ist oder wenn der Benutzer verlangt, dass codierte Pfade für alle Dateien verwendet werden, ist stattdessen der Schlüssel `Encoded-Path` vorhanden. Er speichert dieselben Daten, aber codiert als String aus hexadezimalen Ziffern. Jedes Paar hexadezimaler Ziffern in diesem String steht für ein einzelnes Oktett.

Die folgenden beiden Schlüssel sind immer vorhanden:

- `Size`
  - Die erwartete Größe dieser Datei als Ganzzahl.

- `Last-Modified`
  - Die letzte Änderungszeit der Datei, so wie sie der Server zum Zeitpunkt des Backups gemeldet hat. Anders als die anderen im Backup gespeicherten Felder wird dieses Feld von `pg_verifybackup` nicht verwendet. Es ist nur zu Informationszwecken enthalten.

Wenn das Backup mit aktivierten Datei-Prüfsummen erstellt wurde, sind die folgenden Schlüssel vorhanden:

- `Checksum-Algorithm`
  - Der Prüfsummenalgorithmus, mit dem die Prüfsumme für diese Datei berechnet wurde. Derzeit ist dies für jede Datei im Backup-Manifest derselbe Algorithmus, aber das kann sich in künftigen Releases ändern. Aktuell werden die Prüfsummenalgorithmen `CRC32C`, `SHA224`, `SHA256`, `SHA384` und `SHA512` unterstützt.

- `Checksum`
  - Die für diese Datei berechnete Prüfsumme, gespeichert als Folge hexadezimaler Zeichen, zwei je Byte der Prüfsumme.

## 70.3. WAL-Bereichsobjekt des Backup-Manifests

Das Objekt, das einen WAL-Bereich beschreibt, hat immer drei Schlüssel:

- `Timeline`
  - Die Timeline für diesen Bereich von WAL-Records als Ganzzahl.

- `Start-LSN`
  - Die LSN, ab der auf der angegebenen Timeline Replay beginnen muss, damit dieses Backup verwendet werden kann. Die LSN wird im normalerweise von PostgreSQL verwendeten Format gespeichert, also als String aus zwei hexadezimalen Zeichenketten mit jeweils 1 bis 8 Zeichen Länge, getrennt durch einen Schrägstrich.

- `End-LSN`
  - Die früheste LSN, an der Replay auf der angegebenen Timeline enden darf, wenn dieses Backup verwendet wird. Sie wird im selben Format gespeichert wie `Start-LSN`.

Normalerweise gibt es nur einen einzigen WAL-Bereich. Wenn ein Backup jedoch von einem Standby erstellt wird, der während des Backups wegen einer Promotion upstream die Timeline wechselt, können mehrere Bereiche vorhanden sein, jeder mit einer anderen Timeline. Für dieselbe Timeline gibt es niemals mehrere WAL-Bereiche.
