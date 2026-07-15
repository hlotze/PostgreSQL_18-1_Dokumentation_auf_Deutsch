# Anhang G. Weitere mitgelieferte Programme

Dieser Anhang und der vorherige enthalten Informationen zu den Modulen im Verzeichnis `contrib` der PostgreSQL-Distribution. Weitere allgemeine Informationen zum Bereich `contrib` sowie zu Servererweiterungen und Plug-ins in `contrib` finden Sie in [Anhang F](F_Weitere_mitgelieferte_Module_und_Erweiterungen.md).

Dieser Anhang behandelt Hilfsprogramme aus `contrib`. Nach der Installation, entweder aus dem Quellcode oder über ein Paketsystem, befinden sie sich im Verzeichnis `bin` der PostgreSQL-Installation und können wie jedes andere Programm verwendet werden.

### G.1. Client-Anwendungen

Dieser Abschnitt behandelt PostgreSQL-Client-Anwendungen in `contrib`. Sie können von überall ausgeführt werden, unabhängig davon, wo der Datenbankserver läuft. Siehe auch die PostgreSQL-Client-Anwendungen für Informationen zu Client-Anwendungen, die Teil der PostgreSQL-Kerndistribution sind.

#### G.1.1. oid2name

`oid2name` löst OIDs und Dateiknoten in einem PostgreSQL-Datenverzeichnis auf.

##### G.1.1.1. Synopsis

```text
oid2name [option...]
```

##### G.1.1.2. Beschreibung

`oid2name` ist ein Hilfsprogramm, das Administratoren dabei hilft, die von PostgreSQL verwendete Dateistruktur zu untersuchen. Um es sinnvoll einzusetzen, müssen Sie mit der Datenbank-Dateistruktur vertraut sein, die in [Kapitel 66](66_Physische_Speicherung_der_Datenbank.md) beschrieben wird.

> **Hinweis:** Der Name `oid2name` ist historisch und eigentlich etwas irreführend, da Sie sich bei der Verwendung meistens für die Filenode-Nummern von Tabellen interessieren, also für die Dateinamen, die in den Datenbankverzeichnissen sichtbar sind. Achten Sie darauf, den Unterschied zwischen Tabellen-OIDs und Tabellen-Filenodes zu verstehen.

`oid2name` verbindet sich mit einer Zieldatenbank und extrahiert OID-, Filenode- und/oder Tabellennameninformationen. Es kann außerdem Datenbank-OIDs oder Tablespace-OIDs anzeigen.

##### G.1.1.3. Optionen

`oid2name` akzeptiert die folgenden Befehlszeilenargumente:

- `-f filenode`, `--filenode=filenode`
  - Zeigt Informationen zur Tabelle mit der Filenode `filenode`.

- `-i`, `--indexes`
  - Bezieht Indizes und Sequenzen in die Ausgabe ein.

- `-o oid`, `--oid=oid`
  - Zeigt Informationen zur Tabelle mit der OID `oid`.

- `-q`, `--quiet`
  - Unterdrückt Kopfzeilen; nützlich für Skripte.

- `-s`, `--tablespaces`
  - Zeigt Tablespace-OIDs.

- `-S`, `--system-objects`
  - Bezieht Systemobjekte ein, also Objekte in den Schemas `information_schema`, `pg_toast` und `pg_catalog`.

- `-t tablename_pattern`, `--table=tablename_pattern`
  - Zeigt Informationen zu Tabellen, die auf `tablename_pattern` passen.

- `-V`, `--version`
  - Gibt die Version von `oid2name` aus und beendet das Programm.

- `-x`, `--extended`
  - Zeigt weitere Informationen zu jedem angezeigten Objekt an: Tablespace-Name, Schema-Name und OID.

- `-?`, `--help`
  - Zeigt Hilfe zu den Befehlszeilenargumenten von `oid2name` an und beendet das Programm.

`oid2name` akzeptiert außerdem die folgenden Verbindungsparameter:

- `-d database`, `--dbname=database`
  - Die Datenbank, zu der eine Verbindung hergestellt werden soll.

- `-h host`, `--host=host`
  - Der Host des Datenbankservers.

- `-H host`
  - Der Host des Datenbankservers. Die Verwendung dieses Parameters ist seit PostgreSQL 12 veraltet.

- `-p port`, `--port=port`
  - Der Port des Datenbankservers.

- `-U username`, `--username=username`
  - Der Benutzername für die Verbindung.

Um bestimmte Tabellen anzuzeigen, wählen Sie die Tabellen mit `-o`, `-f` und/oder `-t` aus. `-o` erwartet eine OID, `-f` eine Filenode und `-t` einen Tabellennamen, tatsächlich ein `LIKE`-Muster, sodass Sie beispielsweise `foo%` verwenden können. Sie können beliebig viele dieser Optionen angeben; die Ausgabe enthält alle Objekte, die von einer dieser Optionen getroffen werden. Beachten Sie jedoch, dass diese Optionen nur Objekte in der mit `-d` angegebenen Datenbank anzeigen können.

Wenn Sie keine der Optionen `-o`, `-f` oder `-t`, aber `-d` angeben, werden alle Tabellen in der durch `-d` benannten Datenbank aufgelistet. In diesem Modus steuern die Optionen `-S` und `-i`, was ausgegeben wird.

Wenn Sie auch `-d` nicht angeben, zeigt das Programm eine Liste der Datenbank-OIDs an. Alternativ können Sie mit `-s` eine Tablespace-Liste ausgeben.

##### G.1.1.4. Umgebung

- `PGHOST`
- `PGPORT`
- `PGUSER`

  Standard-Verbindungsparameter.

Dieses Hilfsprogramm verwendet, wie die meisten anderen PostgreSQL-Hilfsprogramme, außerdem die von `libpq` unterstützten Umgebungsvariablen; siehe [Abschnitt 32.15](32_libpq_C_Bibliothek.md#3215-umgebungsvariablen).

Die Umgebungsvariable `PG_COLOR` legt fest, ob Diagnosemeldungen farbig ausgegeben werden. Mögliche Werte sind `always`, `auto` und `never`.

##### G.1.1.5. Hinweise

`oid2name` benötigt einen laufenden Datenbankserver mit nicht beschädigten Systemkatalogen. Für die Wiederherstellung nach katastrophalen Datenbankbeschädigungen ist es daher nur begrenzt nützlich.

##### G.1.1.6. Beispiele

```text
$ # what's in this database server, anyway?
$ oid2name
All databases:
    Oid Database Name Tablespace
----------------------------------
  17228        alvherre pg_default
  17255     regression pg_default
  17227      template0 pg_default
      1      template1 pg_default

$ oid2name -s
All tablespaces:
     Oid Tablespace Name
-------------------------
    1663       pg_default
    1664         pg_global
  155151          fastdisk
  155152           bigdisk

$ # OK, let's look into database alvherre
$ cd $PGDATA/base/17228

$ # get top 10 db objects in the default tablespace, ordered by size
$ ls -lS * | head -10
-rw------- 1 alvherre alvherre 136536064 sep 14 09:51 155173
-rw------- 1 alvherre alvherre 17965056 sep 14 09:51 1155291
-rw------- 1 alvherre alvherre    1204224 sep 14 09:51 16717
-rw------- 1 alvherre alvherre     581632 sep 6 17:51 1255
-rw------- 1 alvherre alvherre     237568 sep 14 09:50 16674
-rw------- 1 alvherre alvherre     212992 sep 14 09:51 1249
-rw------- 1 alvherre alvherre     204800 sep 14 09:51 16684
-rw------- 1 alvherre alvherre     196608 sep 14 09:50 16700
-rw------- 1 alvherre alvherre     163840 sep 14 09:50 16699
-rw------- 1 alvherre alvherre     122880 sep 6 17:51 16751

$ # What file is 155173?
$ oid2name -d alvherre -f 155173
From database "alvherre":
  Filenode Table Name
----------------------
    155173    accounts

$ # you can ask for more than one object
$ oid2name -d alvherre -f 155173 -f 1155291
From database "alvherre":
  Filenode     Table Name
-------------------------
    155173       accounts
   1155291 accounts_pkey

$ # you can mix the options, and get more details with -x
$ oid2name -d alvherre -t accounts -f 1155291 -x
From database "alvherre":
  Filenode     Table Name      Oid Schema Tablespace
------------------------------------------------------
    155173       accounts   155173 public pg_default
   1155291 accounts_pkey 1155291 public pg_default

$ # show disk space for every db object
$ du [0-9]* |
> while read SIZE FILENODE
> do
>    echo "$SIZE       `oid2name -q -d alvherre -i -f $FILENODE`"
> done
16             1155287 branches_pkey
16             1155289 tellers_pkey
17561             1155291 accounts_pkey
...

$ # same, but sort by size
$ du [0-9]* | sort -rn | while read SIZE FN
> do
>    echo "$SIZE  `oid2name -q -d alvherre -f $FN`"
> done
133466             155173    accounts
17561            1155291 accounts_pkey
1177              16717 pg_proc_proname_args_nsp_index
...

$ # If you want to see what's in tablespaces, use the pg_tblspc directory
$ cd $PGDATA/pg_tblspc
$ oid2name -s
All tablespaces:
     Oid Tablespace Name
-------------------------
    1663       pg_default
    1664         pg_global
  155151          fastdisk
  155152           bigdisk

$ cd 155151
$ ls -l
total 0
drwx------ 2 postgres postgres 1024 sep 13 23:20 17228

$ oid2name
All databases:
    Oid Database Name Tablespace
----------------------------------
  17228       alvherre    pg_default
  17255      regression    pg_default
  17227       template0    pg_default
      1       template1    pg_default

$ # Let's see what objects does this database have in the tablespace.
$ cd 155151/17228
$ ls -l
total 0
-rw------- 1 postgres postgres 0 sep 13 23:20 155156

$ # OK, this is a pretty small table ... but which one is it?
$ oid2name -d alvherre -f 155156
From database "alvherre":
  Filenode Table Name
----------------------
    155156         foo
```

##### G.1.1.7. Autor

B. Palmer `<bpalmer@crimelabs.net>`

#### G.1.2. vacuumlo

`vacuumlo` entfernt verwaiste große Objekte aus einer PostgreSQL-Datenbank.

##### G.1.2.1. Synopsis

```text
vacuumlo [option...] dbname...
```

##### G.1.2.2. Beschreibung

`vacuumlo` ist ein einfaches Hilfsprogramm, das verwaiste Large Objects aus einer PostgreSQL-Datenbank entfernt. Als verwaist gilt ein Large Object, dessen OID in keiner Spalte des Typs `oid` oder `lo` der Datenbank vorkommt.

Wenn Sie dieses Programm verwenden, könnte auch der Trigger `lo_manage` aus dem Modul `lo` für Sie interessant sein. `lo_manage` ist nützlich, um die Entstehung verwaister Large Objects von vornherein zu vermeiden.

Alle auf der Befehlszeile genannten Datenbanken werden verarbeitet.

##### G.1.2.3. Optionen

`vacuumlo` akzeptiert die folgenden Befehlszeilenargumente:

- `-l limit`, `--limit=limit`
  - Entfernt höchstens `limit` Large Objects pro Transaktion; Standardwert ist `1000`. Da der Server für jedes entfernte Large Object eine Sperre anfordert, kann das Entfernen zu vieler Large Objects in einer Transaktion dazu führen, dass `max_locks_per_transaction` überschritten wird. Setzen Sie das Limit auf null, wenn alle Löschungen in einer einzigen Transaktion erfolgen sollen.

- `-n`, `--dry-run`
  - Entfernt nichts, sondern zeigt nur an, was getan würde.

- `-v`, `--verbose`
  - Gibt viele Fortschrittsmeldungen aus.

- `-V`, `--version`
  - Gibt die Version von `vacuumlo` aus und beendet das Programm.

- `-?`, `--help`
  - Zeigt Hilfe zu den Befehlszeilenargumenten von `vacuumlo` an und beendet das Programm.

`vacuumlo` akzeptiert außerdem die folgenden Verbindungsparameter:

- `-h host`, `--host=host`
  - Der Host des Datenbankservers.

- `-p port`, `--port=port`
  - Der Port des Datenbankservers.

- `-U username`, `--username=username`
  - Der Benutzername für die Verbindung.

- `-w`, `--no-password`
  - Fordert niemals ein Passwort an. Wenn der Server Passwortauthentifizierung verlangt und kein Passwort auf anderem Weg, etwa über eine `.pgpass`-Datei, verfügbar ist, schlägt der Verbindungsversuch fehl. Diese Option ist in Batch-Jobs und Skripten nützlich, in denen keine Person ein Passwort eingeben kann.

- `-W`, `--password`
  - Erzwingt, dass `vacuumlo` vor dem Verbindungsaufbau nach einem Passwort fragt.

  Diese Option ist nie zwingend erforderlich, da `vacuumlo` automatisch nach einem Passwort fragt, wenn der Server Passwortauthentifizierung verlangt. Allerdings verschwendet `vacuumlo` einen Verbindungsversuch, um herauszufinden, dass der Server ein Passwort verlangt. In manchen Fällen lohnt es sich, `-W` anzugeben, um diesen zusätzlichen Verbindungsversuch zu vermeiden.

##### G.1.2.4. Umgebung

- `PGHOST`
- `PGPORT`
- `PGUSER`

  Standard-Verbindungsparameter.

Dieses Hilfsprogramm verwendet, wie die meisten anderen PostgreSQL-Hilfsprogramme, außerdem die von `libpq` unterstützten Umgebungsvariablen; siehe [Abschnitt 32.15](32_libpq_C_Bibliothek.md#3215-umgebungsvariablen).

Die Umgebungsvariable `PG_COLOR` legt fest, ob Diagnosemeldungen farbig ausgegeben werden. Mögliche Werte sind `always`, `auto` und `never`.

##### G.1.2.5. Hinweise

`vacuumlo` arbeitet folgendermaßen: Zuerst erstellt es eine temporäre Tabelle, die alle OIDs der Large Objects in der ausgewählten Datenbank enthält. Anschließend durchsucht es alle Spalten der Datenbank, die vom Typ `oid` oder `lo` sind, und entfernt passende Einträge aus der temporären Tabelle. Beachten Sie, dass nur Typen mit genau diesen Namen berücksichtigt werden; insbesondere Domains über diesen Typen werden nicht berücksichtigt. Die verbleibenden Einträge in der temporären Tabelle bezeichnen verwaiste Large Objects. Diese werden entfernt.

##### G.1.2.6. Autor

Peter Mount `<peter@retep.org.uk>`

### G.2. Server-Anwendungen

Einige Anwendungen laufen auf dem PostgreSQL-Server selbst. Derzeit sind keine solchen Anwendungen im Verzeichnis `contrib` enthalten. Siehe auch die PostgreSQL-Serveranwendungen für Informationen zu Serveranwendungen, die Teil der PostgreSQL-Kerndistribution sind.
