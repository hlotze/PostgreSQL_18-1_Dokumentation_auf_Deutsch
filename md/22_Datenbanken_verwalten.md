# 22. Datenbanken verwalten

Jede Instanz eines laufenden PostgreSQL-Servers verwaltet eine oder mehrere Datenbanken. Datenbanken bilden daher die oberste Hierarchieebene zur Organisation von SQL-Objekten („Datenbankobjekten“). Dieses Kapitel beschreibt die Eigenschaften von Datenbanken und wie man sie erstellt, verwaltet und löscht.

## 22.1. Überblick

Eine kleine Zahl von Objekten, etwa Rollen-, Datenbank- und Tablespace-Namen, wird auf Cluster-Ebene definiert und im Tablespace `pg_global` gespeichert. Innerhalb des Clusters gibt es mehrere Datenbanken, die voneinander isoliert sind, aber auf Objekte der Cluster-Ebene zugreifen können. Innerhalb jeder Datenbank gibt es mehrere Schemata, die Objekte wie Tabellen und Funktionen enthalten. Die vollständige Hierarchie lautet also: Cluster, Datenbank, Schema, Tabelle (oder eine andere Objektart, etwa eine Funktion).

Beim Verbinden mit dem Datenbankserver muss ein Client in seiner Verbindungsanforderung den Datenbanknamen angeben. Es ist nicht möglich, über eine Verbindung auf mehr als eine Datenbank zuzugreifen. Clients können jedoch mehrere Verbindungen zur gleichen Datenbank oder zu verschiedenen Datenbanken öffnen. Sicherheit auf Datenbankebene hat zwei Bestandteile: Zugriffskontrolle (siehe [Abschnitt 20.1](20_Client_Authentifizierung.md#201-die-datei-pghbaconf)), die auf Verbindungsebene verwaltet wird, und Autorisierungskontrolle (siehe [Abschnitt 5.8](05_Datendefinition.md#58-privilegien)), die über das Grant-System verwaltet wird. Foreign Data Wrappers (siehe `postgres_fdw`) erlauben es, Objekte in einer Datenbank als Stellvertreter für Objekte in anderen Datenbanken oder Clustern auftreten zu lassen. Das ältere Modul `dblink` bietet eine ähnliche Fähigkeit. Standardmäßig können alle Benutzer mit allen Verbindungsmethoden eine Verbindung zu allen Datenbanken herstellen.

Wenn ein PostgreSQL-Server-Cluster nicht zusammenhängende Projekte oder Benutzer enthalten soll, die größtenteils nichts voneinander wissen sollen, empfiehlt es sich, sie in getrennte Datenbanken zu legen und Berechtigungen sowie Zugriffskontrollen entsprechend anzupassen. Wenn die Projekte oder Benutzer miteinander verbunden sind und daher die Ressourcen der jeweils anderen verwenden können sollen, sollten sie in dieselbe Datenbank gelegt werden, aber wahrscheinlich in getrennte Schemata; das bietet eine modulare Struktur mit Namensraumisolation und Autorisierungskontrolle. Weitere Informationen zur Verwaltung von Schemata finden Sie in [Abschnitt 5.10](05_Datendefinition.md#510-schemata).

Obwohl innerhalb eines einzelnen Clusters mehrere Datenbanken erstellt werden können, sollte sorgfältig abgewogen werden, ob die Vorteile die Risiken und Einschränkungen überwiegen. Das gilt insbesondere für die Auswirkungen, die ein gemeinsam genutztes WAL (siehe [Kapitel 28](28_Zuverlässigkeit_und_Write_Ahead_Log.md)) auf Backup- und Wiederherstellungsoptionen hat. Einzelne Datenbanken im Cluster sind aus Benutzersicht zwar isoliert, aus Sicht des Datenbankadministrators aber eng miteinander verbunden.

Datenbanken werden mit dem Befehl `CREATE DATABASE` erstellt (siehe [Abschnitt 22.2](22_Datenbanken_verwalten.md#222-eine-datenbank-erstellen)) und mit dem Befehl `DROP DATABASE` gelöscht (siehe [Abschnitt 22.5](22_Datenbanken_verwalten.md#225-eine-datenbank-löschen)). Um die Menge der vorhandenen Datenbanken zu bestimmen, untersuchen Sie den Systemkatalog `pg_database`, zum Beispiel:

```sql
SELECT datname FROM pg_database;
```

Auch der `psql`-Metabefehl `\l` und die Kommandozeilenoption `-l` sind nützlich, um die vorhandenen Datenbanken aufzulisten.

> Hinweis: Der SQL-Standard nennt Datenbanken „Kataloge“, in der Praxis gibt es aber keinen Unterschied.

## 22.2. Eine Datenbank erstellen

Um eine Datenbank zu erstellen, muss der PostgreSQL-Server laufen (siehe [Abschnitt 18.3](18_Servereinrichtung_und_Betrieb.md#183-den-datenbankserver-starten)).

Datenbanken werden mit dem SQL-Befehl `CREATE DATABASE` erstellt:

```sql
CREATE DATABASE name;
```

Dabei folgt `name` den üblichen Regeln für SQL-Bezeichner. Die aktuelle Rolle wird automatisch Eigentümerin der neuen Datenbank. Zum Privileg des Eigentümers einer Datenbank gehört es, sie später zu entfernen (wodurch auch alle darin enthaltenen Objekte entfernt werden, selbst wenn sie einen anderen Eigentümer haben).

Das Erstellen von Datenbanken ist eine eingeschränkte Operation. Wie man die Berechtigung dafür gewährt, ist in [Abschnitt 21.2](21_Datenbankrollen.md#212-rollenattribute) beschrieben.

Da Sie mit dem Datenbankserver verbunden sein müssen, um den Befehl `CREATE DATABASE` auszuführen, bleibt die Frage, wie an einem Standort die erste Datenbank erstellt werden kann. Die erste Datenbank wird immer vom Befehl `initdb` erstellt, wenn der Datenspeicherbereich initialisiert wird (siehe [Abschnitt 18.2](18_Servereinrichtung_und_Betrieb.md#182-einen-datenbankcluster-erstellen)). Diese Datenbank heißt `postgres`. Um die erste „gewöhnliche“ Datenbank zu erstellen, können Sie sich also mit `postgres` verbinden.

Zwei zusätzliche Datenbanken, `template1` und `template0`, werden ebenfalls während der Initialisierung des Datenbankclusters erstellt. Immer wenn innerhalb des Clusters eine neue Datenbank erstellt wird, wird im Wesentlichen `template1` geklont. Das bedeutet, dass alle Änderungen, die Sie in `template1` vornehmen, an alle später erstellten Datenbanken weitergegeben werden. Vermeiden Sie es deshalb, Objekte in `template1` anzulegen, sofern Sie nicht möchten, dass sie in jede neu erstellte Datenbank übernommen werden. `template0` ist als unveränderte Kopie des ursprünglichen Inhalts von `template1` gedacht. Es kann statt `template1` geklont werden, wenn es wichtig ist, eine Datenbank ohne solche standortspezifischen Ergänzungen zu erstellen. Weitere Details finden Sie in [Abschnitt 22.3](22_Datenbanken_verwalten.md#223-templatedatenbanken).

Der Bequemlichkeit halber gibt es ein Programm, das Sie von der Shell aus ausführen können, um neue Datenbanken zu erstellen: `createdb`.

```text
createdb dbname
```

`createdb` macht nichts Magisches. Es verbindet sich mit der Datenbank `postgres` und gibt den Befehl `CREATE DATABASE` aus, genau wie oben beschrieben. Die Referenzseite zu `createdb` enthält die Details zum Aufruf. Beachten Sie, dass `createdb` ohne Argumente eine Datenbank mit dem Namen des aktuellen Benutzers erstellt.

> Hinweis: [Kapitel 20](20_Client_Authentifizierung.md) enthält Informationen darüber, wie Sie einschränken können, wer sich mit einer bestimmten Datenbank verbinden darf.

Manchmal möchten Sie eine Datenbank für jemand anderen erstellen und diese Person zum Eigentümer der neuen Datenbank machen, damit sie sie selbst konfigurieren und verwalten kann. Verwenden Sie dazu einen der folgenden Befehle:

```sql
CREATE DATABASE dbname OWNER rolename;
```

aus der SQL-Umgebung oder:

```text
createdb -O rolename dbname
```

von der Shell aus. Nur der Superuser darf eine Datenbank für jemand anderen erstellen (das heißt für eine Rolle, deren Mitglied Sie nicht sind).

## 22.3. Template-Datenbanken

`CREATE DATABASE` arbeitet tatsächlich, indem es eine vorhandene Datenbank kopiert. Standardmäßig kopiert es die Standard-Systemdatenbank namens `template1`. Diese Datenbank ist also das „Template“, aus dem neue Datenbanken erzeugt werden. Wenn Sie Objekte zu `template1` hinzufügen, werden diese Objekte in später erstellte Benutzerdatenbanken kopiert. Dieses Verhalten erlaubt standortspezifische Änderungen am Standardsatz von Objekten in Datenbanken. Wenn Sie zum Beispiel die prozedurale Sprache PL/Perl in `template1` installieren, steht sie automatisch in Benutzerdatenbanken zur Verfügung, ohne dass beim Erstellen dieser Datenbanken eine zusätzliche Aktion nötig ist.

`CREATE DATABASE` kopiert jedoch keine Berechtigungen auf Datenbankebene (`GRANT`), die an der Quelldatenbank hängen. Die neue Datenbank hat die Standardberechtigungen auf Datenbankebene.

Es gibt eine zweite Standard-Systemdatenbank namens `template0`. Diese Datenbank enthält dieselben Daten wie der ursprüngliche Inhalt von `template1`, also nur die Standardobjekte, die von Ihrer PostgreSQL-Version vordefiniert sind. `template0` sollte nach der Initialisierung des Datenbankclusters niemals geändert werden. Wenn Sie `CREATE DATABASE` anweisen, `template0` statt `template1` zu kopieren, können Sie eine „unveränderte“ Benutzerdatenbank erstellen (eine, in der keine benutzerdefinierten Objekte vorhanden sind und in der die Systemobjekte nicht verändert wurden), die keine der standortspezifischen Ergänzungen in `template1` enthält. Das ist besonders praktisch beim Wiederherstellen eines `pg_dump`-Dumps: Das Dump-Skript sollte in einer unveränderten Datenbank wiederhergestellt werden, um sicherzustellen, dass man den korrekten Inhalt der gesicherten Datenbank neu erzeugt, ohne mit Objekten in Konflikt zu geraten, die später zu `template1` hinzugefügt worden sein könnten.

Ein weiterer häufiger Grund, `template0` statt `template1` zu kopieren, ist, dass beim Kopieren von `template0` neue Kodierungs- und Locale-Einstellungen angegeben werden können, während eine Kopie von `template1` dieselben Einstellungen verwenden muss wie `template1` selbst. Das liegt daran, dass `template1` kodierungs- oder locale-spezifische Daten enthalten könnte, während bekannt ist, dass dies bei `template0` nicht der Fall ist.

Um eine Datenbank durch Kopieren von `template0` zu erstellen, verwenden Sie:

```sql
CREATE DATABASE dbname TEMPLATE template0;
```

aus der SQL-Umgebung oder:

```text
createdb -T template0 dbname
```

von der Shell aus.

Es ist möglich, zusätzliche Template-Datenbanken zu erstellen, und tatsächlich kann man jede Datenbank in einem Cluster kopieren, indem man ihren Namen als Template für `CREATE DATABASE` angibt. Wichtig ist jedoch zu verstehen, dass dies (noch) nicht als allgemeine `COPY DATABASE`-Funktion gedacht ist. Die wichtigste Einschränkung besteht darin, dass keine anderen Sitzungen mit der Quelldatenbank verbunden sein dürfen, während sie kopiert wird. `CREATE DATABASE` schlägt fehl, wenn beim Start eine andere Verbindung existiert; während des Kopiervorgangs werden neue Verbindungen zur Quelldatenbank verhindert.

Für jede Datenbank gibt es in `pg_database` zwei nützliche Flags: die Spalten `datistemplate` und `datallowconn`. `datistemplate` kann gesetzt werden, um anzugeben, dass eine Datenbank als Template für `CREATE DATABASE` gedacht ist. Wenn dieses Flag gesetzt ist, kann die Datenbank von jedem Benutzer mit `CREATEDB`-Privilegien geklont werden; wenn es nicht gesetzt ist, können nur Superuser und der Eigentümer der Datenbank sie klonen. Wenn `datallowconn` `false` ist, werden keine neuen Verbindungen zu dieser Datenbank erlaubt (vorhandene Sitzungen werden aber nicht einfach dadurch beendet, dass das Flag auf `false` gesetzt wird). Die Datenbank `template0` ist normalerweise mit `datallowconn = false` markiert, um ihre Änderung zu verhindern. Sowohl `template0` als auch `template1` sollten immer mit `datistemplate = true` markiert sein.

> Hinweis: `template1` und `template0` haben keinen besonderen Status außer der Tatsache, dass der Name `template1` der Standardname der Quelldatenbank für `CREATE DATABASE` ist. Man könnte zum Beispiel `template1` löschen und aus `template0` neu erstellen, ohne nachteilige Auswirkungen. Dieses Vorgehen kann ratsam sein, wenn man unvorsichtig viel Unrat in `template1` abgelegt hat. (Um `template1` zu löschen, muss `pg_database.datistemplate = false` sein.)
>
> Die Datenbank `postgres` wird ebenfalls erstellt, wenn ein Datenbankcluster initialisiert wird. Diese Datenbank ist als Standarddatenbank gedacht, mit der sich Benutzer und Anwendungen verbinden. Sie ist einfach eine Kopie von `template1` und kann bei Bedarf gelöscht und neu erstellt werden.

## 22.4. Datenbankkonfiguration

Wie aus [Kapitel 19](19_Serverkonfiguration.md) bekannt, stellt der PostgreSQL-Server eine große Zahl von Laufzeit-Konfigurationsvariablen bereit. Für viele dieser Einstellungen können Sie datenbankspezifische Standardwerte setzen.

Wenn Sie zum Beispiel aus irgendeinem Grund den GEQO-Optimierer für eine bestimmte Datenbank deaktivieren möchten, müssten Sie ihn normalerweise entweder für alle Datenbanken deaktivieren oder sicherstellen, dass jeder verbindende Client sorgfältig `SET geqo TO off` ausführt. Um diese Einstellung innerhalb einer bestimmten Datenbank zum Standard zu machen, können Sie den folgenden Befehl ausführen:

```sql
ALTER DATABASE mydb SET geqo TO off;
```

Dadurch wird die Einstellung gespeichert (aber nicht sofort gesetzt). Bei späteren Verbindungen zu dieser Datenbank erscheint es so, als wäre `SET geqo TO off;` unmittelbar vor Beginn der Sitzung ausgeführt worden. Beachten Sie, dass Benutzer diese Einstellung während ihrer Sitzungen weiterhin ändern können; sie ist nur der Standardwert. Um eine solche Einstellung rückgängig zu machen, verwenden Sie `ALTER DATABASE dbname RESET varname`.

## 22.5. Eine Datenbank löschen

Datenbanken werden mit dem Befehl `DROP DATABASE` gelöscht:

```sql
DROP DATABASE name;
```

Nur der Eigentümer der Datenbank oder ein Superuser kann eine Datenbank löschen. Das Löschen einer Datenbank entfernt alle Objekte, die in der Datenbank enthalten waren. Die Zerstörung einer Datenbank kann nicht rückgängig gemacht werden.

Sie können den Befehl `DROP DATABASE` nicht ausführen, während Sie mit der zu löschenden Datenbank verbunden sind. Sie können jedoch mit jeder anderen Datenbank verbunden sein, auch mit der Datenbank `template1`. `template1` wäre die einzige Möglichkeit, um die letzte Benutzerdatenbank eines gegebenen Clusters zu löschen.

Der Bequemlichkeit halber gibt es auch ein Shell-Programm zum Löschen von Datenbanken: `dropdb`.

```text
dropdb dbname
```

(Anders als bei `createdb` ist es nicht die Standardaktion, die Datenbank mit dem Namen des aktuellen Benutzers zu löschen.)

## 22.6. Tablespaces

Tablespaces in PostgreSQL erlauben Datenbankadministratoren, Orte im Dateisystem festzulegen, an denen die Dateien gespeichert werden können, die Datenbankobjekte repräsentieren. Nach der Erstellung kann beim Erzeugen von Datenbankobjekten über den Namen auf einen Tablespace verwiesen werden.

Mit Tablespaces kann ein Administrator das Plattenlayout einer PostgreSQL-Installation steuern. Das ist mindestens auf zwei Arten nützlich. Erstens: Wenn die Partition oder das Volume, auf dem der Cluster initialisiert wurde, keinen freien Platz mehr hat und nicht erweitert werden kann, kann ein Tablespace auf einer anderen Partition erstellt und genutzt werden, bis das System neu konfiguriert werden kann.

Zweitens erlauben Tablespaces einem Administrator, Wissen über das Nutzungsmuster von Datenbankobjekten zur Leistungsoptimierung einzusetzen. Ein sehr stark genutzter Index kann zum Beispiel auf einer sehr schnellen, hochverfügbaren Platte liegen, etwa auf einem teuren Solid-State-Gerät. Gleichzeitig könnte eine Tabelle mit archivierten Daten, die selten genutzt werden oder nicht leistungskritisch sind, auf einem weniger teuren, langsameren Plattensystem gespeichert werden.

> Warnung: Obwohl Tablespaces außerhalb des Hauptdatenverzeichnisses von PostgreSQL liegen, sind sie ein integraler Bestandteil des Datenbankclusters und können nicht als eigenständige Sammlung von Datendateien behandelt werden. Sie hängen von Metadaten ab, die im Hauptdatenverzeichnis enthalten sind, und können daher weder an einen anderen Datenbankcluster angehängt noch einzeln gesichert werden. Ebenso kann der Datenbankcluster unlesbar werden oder nicht mehr starten, wenn Sie einen Tablespace verlieren (Dateilöschung, Plattenausfall usw.). Einen Tablespace auf ein temporäres Dateisystem wie eine RAM-Disk zu legen, gefährdet die Zuverlässigkeit des gesamten Clusters.

Um einen Tablespace zu definieren, verwenden Sie den Befehl `CREATE TABLESPACE`, zum Beispiel:

```sql
CREATE TABLESPACE fastspace LOCATION '/ssd1/postgresql/data';
```

Der Ort muss ein vorhandenes, leeres Verzeichnis sein, das dem PostgreSQL-Betriebssystembenutzer gehört. Alle anschließend innerhalb des Tablespace erstellten Objekte werden in Dateien unterhalb dieses Verzeichnisses gespeichert. Der Ort darf nicht auf entfernbaren oder flüchtigen Speichern liegen, da der Cluster nicht mehr funktionieren könnte, wenn der Tablespace fehlt oder verloren geht.

> Hinweis: Es ist normalerweise wenig sinnvoll, mehr als einen Tablespace pro logischem Dateisystem anzulegen, da Sie die Lage einzelner Dateien innerhalb eines logischen Dateisystems nicht steuern können. PostgreSQL erzwingt eine solche Einschränkung jedoch nicht und kennt die Dateisystemgrenzen auf Ihrem System auch nicht direkt. Es speichert Dateien einfach in den Verzeichnissen, die Sie ihm angeben.

Das Erstellen des Tablespace selbst muss als Datenbank-Superuser erfolgen, danach können Sie jedoch gewöhnlichen Datenbankbenutzern erlauben, ihn zu verwenden. Gewähren Sie ihnen dazu das Privileg `CREATE` darauf.

Tabellen, Indizes und ganze Datenbanken können bestimmten Tablespaces zugewiesen werden. Dazu muss ein Benutzer mit dem `CREATE`-Privileg auf einem gegebenen Tablespace den Namen des Tablespace als Parameter an den entsprechenden Befehl übergeben. Das folgende Beispiel erstellt etwa eine Tabelle im Tablespace `space1`:

```sql
CREATE TABLE foo(i int) TABLESPACE space1;
```

Alternativ verwenden Sie den Parameter `default_tablespace`:

```sql
SET default_tablespace = space1;
CREATE TABLE foo(i int);
```

Wenn `default_tablespace` auf etwas anderes als eine leere Zeichenkette gesetzt ist, liefert er eine implizite `TABLESPACE`-Klausel für Befehle `CREATE TABLE` und `CREATE INDEX`, die keine ausdrückliche solche Klausel enthalten.

Außerdem gibt es den Parameter `temp_tablespaces`, der die Platzierung temporärer Tabellen und Indizes sowie temporärer Dateien festlegt, die etwa zum Sortieren großer Datenmengen verwendet werden. Dies kann eine Liste von Tablespace-Namen sein, nicht nur ein einzelner Name, sodass die mit temporären Objekten verbundene Last über mehrere Tablespaces verteilt werden kann. Jedes Mal, wenn ein temporäres Objekt erstellt werden soll, wird zufällig ein Mitglied der Liste ausgewählt.

Der einer Datenbank zugeordnete Tablespace wird verwendet, um die Systemkataloge dieser Datenbank zu speichern. Außerdem ist er der Standard-Tablespace für Tabellen, Indizes und temporäre Dateien, die innerhalb der Datenbank erstellt werden, wenn keine `TABLESPACE`-Klausel angegeben ist und keine andere Auswahl durch `default_tablespace` oder `temp_tablespaces` (je nach Fall) festgelegt wurde. Wenn eine Datenbank ohne Angabe eines Tablespace erstellt wird, verwendet sie denselben Tablespace wie die Template-Datenbank, von der sie kopiert wird.

Beim Initialisieren des Datenbankclusters werden automatisch zwei Tablespaces erstellt. Der Tablespace `pg_global` wird nur für gemeinsam genutzte Systemkataloge verwendet. Der Tablespace `pg_default` ist der Standard-Tablespace der Datenbanken `template1` und `template0` (und damit auch der Standard-Tablespace anderer Datenbanken, sofern dies nicht durch eine `TABLESPACE`-Klausel in `CREATE DATABASE` überschrieben wird).

Nach seiner Erstellung kann ein Tablespace aus jeder Datenbank verwendet werden, sofern der anfragende Benutzer ausreichende Privilegien besitzt. Das bedeutet, dass ein Tablespace erst gelöscht werden kann, wenn alle Objekte in allen Datenbanken entfernt wurden, die diesen Tablespace verwenden.

Um einen leeren Tablespace zu entfernen, verwenden Sie den Befehl `DROP TABLESPACE`.

Um die Menge der vorhandenen Tablespaces zu bestimmen, untersuchen Sie den Systemkatalog `pg_tablespace`, zum Beispiel:

```sql
SELECT spcname, spcowner::regrole, pg_tablespace_location(oid) FROM
 pg_tablespace;
```

Es ist möglich herauszufinden, welche Datenbanken welche Tablespaces verwenden; siehe Tabelle 9.76. Auch der `psql`-Metabefehl `\db` ist nützlich, um die vorhandenen Tablespaces aufzulisten.

Das Verzeichnis `$PGDATA/pg_tblspc` enthält symbolische Links, die auf jeden der nicht eingebauten Tablespaces zeigen, die im Cluster definiert sind. Obwohl es nicht empfohlen wird, ist es möglich, das Tablespace-Layout von Hand anzupassen, indem diese Links neu definiert werden. Führen Sie diese Operation unter keinen Umständen aus, während der Server läuft.
