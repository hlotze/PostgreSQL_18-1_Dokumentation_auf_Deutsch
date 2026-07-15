# 20. Client-Authentifizierung

Wenn eine Client-Anwendung eine Verbindung zum Datenbankserver herstellt, gibt sie an, als welcher PostgreSQL-Datenbankbenutzername sie sich verbinden möchte, ganz ähnlich wie man sich auf einem Unix-Rechner als bestimmter Benutzer anmeldet. Innerhalb der SQL-Umgebung bestimmt der aktive Datenbankbenutzername die Zugriffsprivilegien auf Datenbankobjekte; weitere Informationen finden Sie in [Kapitel 21](21_Datenbankrollen.md). Daher ist es wesentlich, einzuschränken, welche Datenbankbenutzer sich verbinden können.

> Hinweis: Wie in [Kapitel 21](21_Datenbankrollen.md) erläutert, verwaltet PostgreSQL Privilegien tatsächlich in Begriffen von „Rollen“. In diesem Kapitel verwenden wir durchgehend Datenbankbenutzer im Sinne von „Rolle mit dem Privileg `LOGIN`“.

Authentifizierung ist der Vorgang, mit dem der Datenbankserver die Identität des Clients feststellt und daraus ableitet, ob die Client-Anwendung (oder der Benutzer, der die Client-Anwendung ausführt) sich mit dem angeforderten Datenbankbenutzernamen verbinden darf.

PostgreSQL bietet eine Reihe verschiedener Client-Authentifizierungsmethoden. Die Methode zur Authentifizierung einer bestimmten Client-Verbindung kann auf Grundlage der Hostadresse des Clients, der Datenbank und des Benutzers ausgewählt werden.

PostgreSQL-Datenbankbenutzernamen sind logisch von den Benutzernamen des Betriebssystems getrennt, auf dem der Server läuft. Wenn alle Benutzer eines bestimmten Servers auch Konten auf der Servermaschine haben, ist es sinnvoll, Datenbankbenutzernamen zu vergeben, die ihren Betriebssystembenutzernamen entsprechen. Ein Server, der Remote-Verbindungen akzeptiert, kann jedoch viele Datenbankbenutzer haben, die kein lokales Betriebssystemkonto besitzen; in solchen Fällen muss es keine Verbindung zwischen Datenbankbenutzernamen und Betriebssystembenutzernamen geben.

## 20.1. Die Datei pg_hba.conf

Die Client-Authentifizierung wird durch eine Konfigurationsdatei gesteuert, die traditionell `pg_hba.conf` heißt und im Datenverzeichnis des Datenbankclusters gespeichert ist. (HBA steht für host-based authentication.) Eine standardmäßige Datei `pg_hba.conf` wird installiert, wenn das Datenverzeichnis durch `initdb` initialisiert wird. Es ist jedoch möglich, die Authentifizierungskonfigurationsdatei an anderer Stelle abzulegen; siehe den Konfigurationsparameter `hba_file`.

Die Datei `pg_hba.conf` wird beim Start und immer dann gelesen, wenn der Hauptserverprozess ein `SIGHUP`-Signal empfängt. Wenn Sie die Datei auf einem aktiven System bearbeiten, müssen Sie dem Postmaster ein Signal senden (mit `pg_ctl reload`, durch Aufruf der SQL-Funktion `pg_reload_conf()` oder mit `kill -HUP`), damit er die Datei erneut liest.

> Hinweis: Die vorherige Aussage gilt nicht für Microsoft Windows: Dort werden Änderungen an der Datei `pg_hba.conf` sofort auf nachfolgende neue Verbindungen angewendet.

Die System-View `pg_hba_file_rules` kann hilfreich sein, um Änderungen an der Datei `pg_hba.conf` vorab zu testen oder Probleme zu diagnostizieren, wenn das Laden der Datei nicht die gewünschten Auswirkungen hatte. Zeilen in der View mit nicht-null Fehlerfeldern weisen auf Probleme in den entsprechenden Zeilen der Datei hin.

Das allgemeine Format der Datei `pg_hba.conf` ist eine Menge von Datensätzen, einer pro Zeile. Leerzeilen werden ignoriert, ebenso jeder Text nach dem Kommentarzeichen `#`. Ein Datensatz kann in der nächsten Zeile fortgesetzt werden, indem die Zeile mit einem Backslash endet. (Backslashes sind nur am Zeilenende besonders.) Ein Datensatz besteht aus mehreren Feldern, die durch Leerzeichen und/oder Tabulatoren getrennt sind. Felder können Whitespace enthalten, wenn der Feldwert in doppelte Anführungszeichen gesetzt wird. Wenn eines der Schlüsselwörter in einem Datenbank-, Benutzer- oder Adressfeld (zum Beispiel `all` oder `replication`) quotiert wird, verliert das Wort seine besondere Bedeutung und passt nur auf eine Datenbank, einen Benutzer oder einen Host mit diesem Namen. Die Zeilenfortsetzung per Backslash gilt auch innerhalb von quotiertem Text oder Kommentaren.

Jeder Authentifizierungsdatensatz gibt einen Verbindungstyp, einen Client-IP-Adressbereich (falls für den Verbindungstyp relevant), einen Datenbanknamen, einen Benutzernamen und die Authentifizierungsmethode an, die für Verbindungen verwendet werden soll, die zu diesen Parametern passen. Der erste Datensatz mit passendem Verbindungstyp, passender Client-Adresse, angeforderter Datenbank und passendem Benutzernamen wird zur Authentifizierung verwendet. Es gibt kein „Fall-through“ und kein „Backup“: Wenn ein Datensatz ausgewählt wird und die Authentifizierung fehlschlägt, werden nachfolgende Datensätze nicht berücksichtigt. Wenn kein Datensatz passt, wird der Zugriff verweigert.

Jeder Datensatz kann eine Include-Direktive oder ein Authentifizierungsdatensatz sein. Include-Direktiven geben Dateien an, die eingebunden werden können und zusätzliche Datensätze enthalten. Die Datensätze werden anstelle der Include-Direktiven eingefügt. Include-Direktiven enthalten nur zwei Felder: die Direktive `include`, `include_if_exists` oder `include_dir` und die einzubindende Datei beziehungsweise das einzubindende Verzeichnis. Datei oder Verzeichnis können ein relativer oder absoluter Pfad sein und können in doppelte Anführungszeichen gesetzt werden. Bei der Form `include_dir` werden alle Dateien eingebunden, die nicht mit `.` beginnen und mit `.conf` enden. Mehrere Dateien innerhalb eines Include-Verzeichnisses werden in Dateinamenreihenfolge verarbeitet (nach den Regeln der C-Locale, das heißt Zahlen vor Buchstaben und Großbuchstaben vor Kleinbuchstaben).

Ein Datensatz kann mehrere Formate haben:

| Typ | Datenbank | Benutzer | Adresse | IP-Adresse | IP-Maske | Methode | Optionen |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `local` | `database` | `user` |  |  |  | `auth-method` | `[auth-options]` |
| `host` | `database` | `user` | `address` |  |  | `auth-method` | `[auth-options]` |
| `hostssl` | `database` | `user` | `address` |  |  | `auth-method` | `[auth-options]` |
| `hostnossl` | `database` | `user` | `address` |  |  | `auth-method` | `[auth-options]` |
| `hostgssenc` | `database` | `user` | `address` |  |  | `auth-method` | `[auth-options]` |
| `hostnogssenc` | `database` | `user` | `address` |  |  | `auth-method` | `[auth-options]` |
| `host` | `database` | `user` |  | `IP-address` | `IP-mask` | `auth-method` | `[auth-options]` |
| `hostssl` | `database` | `user` |  | `IP-address` | `IP-mask` | `auth-method` | `[auth-options]` |
| `hostnossl` | `database` | `user` |  | `IP-address` | `IP-mask` | `auth-method` | `[auth-options]` |
| `hostgssenc` | `database` | `user` |  | `IP-address` | `IP-mask` | `auth-method` | `[auth-options]` |
| `hostnogssenc` | `database` | `user` |  | `IP-address` | `IP-mask` | `auth-method` | `[auth-options]` |
| `include` | `file` |  |  |  |  |  |  |
| `include_if_exists` | `file` |  |  |  |  |  |  |
| `include_dir` | `directory` |  |  |  |  |  |  |

Die Bedeutung der Felder ist wie folgt:

`local`

Dieser Datensatz passt auf Verbindungsversuche über Unix-Domain-Sockets. Ohne einen Datensatz dieses Typs sind Unix-Domain-Socket-Verbindungen nicht erlaubt.

`host`

Dieser Datensatz passt auf Verbindungsversuche über TCP/IP. `host`-Datensätze passen auf SSL- oder Nicht-SSL-Verbindungsversuche sowie auf GSSAPI-verschlüsselte oder nicht GSSAPI-verschlüsselte Verbindungsversuche.

> Hinweis: Remote-TCP/IP-Verbindungen sind nicht möglich, sofern der Server nicht mit einem geeigneten Wert für den Konfigurationsparameter `listen_addresses` gestartet wird, da das Standardverhalten darin besteht, für TCP/IP-Verbindungen nur auf der lokalen Loopback-Adresse `localhost` zu lauschen.

`hostssl`

Dieser Datensatz passt auf Verbindungsversuche über TCP/IP, aber nur, wenn die Verbindung mit SSL-Verschlüsselung hergestellt wird.

Um diese Option zu verwenden, muss der Server mit SSL-Unterstützung gebaut sein. Außerdem muss SSL durch Setzen des Konfigurationsparameters `ssl` aktiviert sein (weitere Informationen finden Sie in [Abschnitt 18.9](18_Servereinrichtung_und_Betrieb.md#189-sichere-tcpipverbindungen-mit-ssl)). Andernfalls wird der `hostssl`-Datensatz ignoriert, abgesehen vom Loggen einer Warnung, dass er auf keine Verbindungen passen kann.

`hostnossl`

Dieser Datensatztyp verhält sich entgegengesetzt zu `hostssl`; er passt nur auf Verbindungsversuche über TCP/IP, die kein SSL verwenden.

`hostgssenc`

Dieser Datensatz passt auf Verbindungsversuche über TCP/IP, aber nur, wenn die Verbindung mit GSSAPI-Verschlüsselung hergestellt wird.

Um diese Option zu verwenden, muss der Server mit GSSAPI-Unterstützung gebaut sein. Andernfalls wird der `hostgssenc`-Datensatz ignoriert, abgesehen vom Loggen einer Warnung, dass er auf keine Verbindungen passen kann.

`hostnogssenc`

Dieser Datensatztyp verhält sich entgegengesetzt zu `hostgssenc`; er passt nur auf Verbindungsversuche über TCP/IP, die keine GSSAPI-Verschlüsselung verwenden.

`database`

Gibt an, auf welche Datenbanknamen dieser Datensatz passt. Der Wert `all` gibt an, dass er auf alle Datenbanken passt. Der Wert `sameuser` gibt an, dass der Datensatz passt, wenn die angeforderte Datenbank denselben Namen hat wie der angeforderte Benutzer. Der Wert `samerole` gibt an, dass der angeforderte Benutzer Mitglied der Rolle mit demselben Namen wie die angeforderte Datenbank sein muss. (`samegroup` ist eine veraltete, aber weiterhin akzeptierte Schreibweise von `samerole`.) Superuser gelten für die Zwecke von `samerole` nicht als Mitglieder einer Rolle, sofern sie nicht ausdrücklich direkt oder indirekt Mitglied der Rolle sind, und nicht nur aufgrund ihrer Superuser-Eigenschaft. Der Wert `replication` gibt an, dass der Datensatz passt, wenn eine physische Replikationsverbindung angefordert wird; er passt jedoch nicht auf logische Replikationsverbindungen. Beachten Sie, dass physische Replikationsverbindungen keine bestimmte Datenbank angeben, logische Replikationsverbindungen dagegen schon. Andernfalls ist dies der Name einer bestimmten PostgreSQL-Datenbank oder ein regulärer Ausdruck. Mehrere Datenbanknamen und/oder reguläre Ausdrücke können durch Kommas getrennt angegeben werden.

Wenn der Datenbankname mit einem Schrägstrich (`/`) beginnt, wird der Rest des Namens als regulärer Ausdruck behandelt. Details zur Syntax regulärer Ausdrücke in PostgreSQL finden Sie in Abschnitt 9.7.3.1.

Eine separate Datei mit Datenbanknamen und/oder regulären Ausdrücken kann angegeben werden, indem dem Dateinamen `@` vorangestellt wird.

`user`

Gibt an, auf welche Datenbankbenutzernamen dieser Datensatz passt. Der Wert `all` gibt an, dass er auf alle Benutzer passt. Andernfalls ist dies entweder der Name eines bestimmten Datenbankbenutzers, ein regulärer Ausdruck (wenn der Name mit einem Schrägstrich (`/`) beginnt) oder ein Gruppenname mit vorangestelltem `+`. (Denken Sie daran, dass es in PostgreSQL keinen wirklichen Unterschied zwischen Benutzern und Gruppen gibt; ein `+` bedeutet tatsächlich „passe auf jede Rolle, die direkt oder indirekt Mitglied dieser Rolle ist“, während ein Name ohne `+` nur auf genau diese Rolle passt.) Für diesen Zweck gilt ein Superuser nur dann als Mitglied einer Rolle, wenn er ausdrücklich direkt oder indirekt Mitglied der Rolle ist, und nicht nur aufgrund seiner Superuser-Eigenschaft. Mehrere Benutzernamen und/oder reguläre Ausdrücke können durch Kommas getrennt angegeben werden.

Wenn der Benutzername mit einem Schrägstrich (`/`) beginnt, wird der Rest des Namens als regulärer Ausdruck behandelt. Details zur Syntax regulärer Ausdrücke in PostgreSQL finden Sie in Abschnitt 9.7.3.1.

Eine separate Datei mit Benutzernamen und/oder regulären Ausdrücken kann angegeben werden, indem dem Dateinamen `@` vorangestellt wird.

`address`

Gibt die Adresse(n) der Client-Maschine an, auf die dieser Datensatz passt. Dieses Feld kann entweder einen Hostnamen, einen IP-Adressbereich oder eines der unten genannten besonderen Schlüsselwörter enthalten.

Ein IP-Adressbereich wird mit der standardmäßigen numerischen Schreibweise für die Startadresse des Bereichs angegeben, gefolgt von einem Schrägstrich (`/`) und einer CIDR-Maskenlänge. Die Maskenlänge gibt die Anzahl der höherwertigen Bits der Client-IP-Adresse an, die übereinstimmen müssen. Bits rechts davon sollten in der angegebenen IP-Adresse null sein. Zwischen IP-Adresse, `/` und CIDR-Maskenlänge darf kein Whitespace stehen.

Typische Beispiele für einen so angegebenen IPv4-Adressbereich sind `172.20.143.89/32` für einen einzelnen Host, `172.20.143.0/24` für ein kleines Netzwerk oder `10.6.0.0/16` für ein größeres. Ein IPv6-Adressbereich könnte wie `::1/128` für einen einzelnen Host aussehen (in diesem Fall die IPv6-Loopback-Adresse) oder wie `fe80::7a31:c1ff:0000:0000/96` für ein kleines Netzwerk. `0.0.0.0/0` steht für alle IPv4-Adressen und `::0/0` für alle IPv6-Adressen. Um einen einzelnen Host anzugeben, verwenden Sie eine Maskenlänge von `32` für IPv4 oder `128` für IPv6. Lassen Sie bei einer Netzwerkadresse keine abschließenden Nullen weg.

Ein Eintrag im IPv4-Format passt nur auf IPv4-Verbindungen, und ein Eintrag im IPv6-Format passt nur auf IPv6-Verbindungen, selbst wenn die dargestellte Adresse im IPv4-in-IPv6-Bereich liegt.

Sie können außerdem `all` schreiben, um auf jede IP-Adresse zu passen, `samehost`, um auf jede eigene IP-Adresse des Servers zu passen, oder `samenet`, um auf jede Adresse in jedem Subnetz zu passen, mit dem der Server direkt verbunden ist.

Wenn ein Hostname angegeben wird (alles, was kein IP-Adressbereich und kein besonderes Schlüsselwort ist, wird als Hostname behandelt), wird dieser Name mit dem Ergebnis einer Reverse-Namensauflösung der IP-Adresse des Clients verglichen (zum Beispiel Reverse-DNS-Lookup, wenn DNS verwendet wird). Hostnamenvergleiche sind nicht case-sensitiv. Wenn es eine Übereinstimmung gibt, wird eine Forward-Namensauflösung (zum Beispiel Forward-DNS-Lookup) für den Hostnamen durchgeführt, um zu prüfen, ob eine der Adressen, zu denen er aufgelöst wird, der IP-Adresse des Clients entspricht. Wenn beide Richtungen übereinstimmen, gilt der Eintrag als passend. (Der Hostname, der in `pg_hba.conf` verwendet wird, sollte derjenige sein, den die Adresse-zu-Name-Auflösung der Client-IP-Adresse zurückgibt; andernfalls passt die Zeile nicht. Einige Hostnamendatenbanken erlauben, eine IP-Adresse mit mehreren Hostnamen zu verknüpfen, aber das Betriebssystem gibt bei der Auflösung einer IP-Adresse nur einen Hostnamen zurück.)

Eine Hostnamenspezifikation, die mit einem Punkt (`.`) beginnt, passt auf ein Suffix des tatsächlichen Hostnamens. `.example.com` würde also auf `foo.example.com` passen (aber nicht nur auf `example.com`).

Wenn Hostnamen in `pg_hba.conf` angegeben werden, sollten Sie sicherstellen, dass die Namensauflösung hinreichend schnell ist. Es kann vorteilhaft sein, einen lokalen Namensauflösungs-Cache wie `nscd` einzurichten. Außerdem möchten Sie möglicherweise den Konfigurationsparameter `log_hostname` aktivieren, um den Hostnamen des Clients statt der IP-Adresse im Log zu sehen.

Diese Felder gelten nicht für `local`-Datensätze.

> Hinweis: Benutzer fragen sich manchmal, warum Hostnamen auf diese scheinbar komplizierte Weise behandelt werden, mit zwei Namensauflösungen einschließlich eines Reverse-Lookups der Client-IP-Adresse. Das erschwert die Nutzung der Funktion, wenn der Reverse-DNS-Eintrag des Clients nicht eingerichtet ist oder einen unerwünschten Hostnamen ergibt. Es geschieht in erster Linie aus Effizienzgründen: So erfordert ein Verbindungsversuch höchstens zwei Resolver-Lookups, einen rückwärts und einen vorwärts. Wenn es ein Resolver-Problem mit einer Adresse gibt, wird es nur zum Problem dieses Clients. Eine hypothetische alternative Implementierung, die nur Forward-Lookups durchführt, müsste bei jedem Verbindungsversuch jeden in `pg_hba.conf` genannten Hostnamen auflösen. Das könnte bei vielen aufgeführten Namen recht langsam sein. Und wenn es ein Resolver-Problem mit einem der Hostnamen gibt, wird es zum Problem aller.
>
> Außerdem ist ein Reverse-Lookup erforderlich, um die Suffix-Matching-Funktion zu implementieren, weil der tatsächliche Client-Hostname bekannt sein muss, um ihn gegen das Muster zu prüfen.
>
> Beachten Sie, dass dieses Verhalten mit anderen verbreiteten Implementierungen hostnamenbasierter Zugriffskontrolle konsistent ist, etwa dem Apache HTTP Server und TCP Wrappers.

`IP-address`

`IP-mask`

Diese beiden Felder können als Alternative zur Schreibweise `IP-address`/Maskenlänge verwendet werden. Statt die Maskenlänge anzugeben, wird die tatsächliche Maske in einer separaten Spalte angegeben. Beispielsweise steht `255.0.0.0` für eine IPv4-CIDR-Maskenlänge von `8`, und `255.255.255.255` steht für eine CIDR-Maskenlänge von `32`.

Diese Felder gelten nicht für `local`-Datensätze.

`auth-method`

Gibt die Authentifizierungsmethode an, die verwendet wird, wenn eine Verbindung auf diesen Datensatz passt. Die möglichen Auswahlwerte sind hier zusammengefasst; Details finden Sie in [Abschnitt 20.3](20_Client_Authentifizierung.md#203-authentifizierungsmethoden). Alle Optionen sind kleingeschrieben und werden case-sensitiv behandelt, daher müssen selbst Akronyme wie `ldap` kleingeschrieben angegeben werden.

`trust`

Erlaubt die Verbindung bedingungslos. Diese Methode erlaubt jedem, der eine Verbindung zum PostgreSQL-Datenbankserver herstellen kann, sich als beliebiger PostgreSQL-Benutzer anzumelden, ohne Passwort oder irgendeine andere Authentifizierung. Details finden Sie in [Abschnitt 20.4](20_Client_Authentifizierung.md#204-trustauthentifizierung).

`reject`

Lehnt die Verbindung bedingungslos ab. Dies ist nützlich, um bestimmte Hosts aus einer Gruppe „herauszufiltern“; beispielsweise könnte eine `reject`-Zeile einen bestimmten Host an der Verbindung hindern, während eine spätere Zeile den übrigen Hosts in einem bestimmten Netzwerk die Verbindung erlaubt.

`scram-sha-256`

Führt SCRAM-SHA-256-Authentifizierung aus, um das Passwort des Benutzers zu prüfen. Details finden Sie in [Abschnitt 20.5](20_Client_Authentifizierung.md#205-passwortauthentifizierung).

`md5`

Führt SCRAM-SHA-256- oder MD5-Authentifizierung aus, um das Passwort des Benutzers zu prüfen. Details finden Sie in Abschnitt 20.5.

> Warnung: Unterstützung für MD5-verschlüsselte Passwörter ist veraltet und wird in einer zukünftigen PostgreSQL-Version entfernt. Details zur Migration auf einen anderen Passworttyp finden Sie in [Abschnitt 20.5](20_Client_Authentifizierung.md#205-passwortauthentifizierung).

`password`

Verlangt vom Client, ein unverschlüsseltes Passwort zur Authentifizierung bereitzustellen. Da das Passwort im Klartext über das Netzwerk gesendet wird, sollte dies in nicht vertrauenswürdigen Netzwerken nicht verwendet werden. Details finden Sie in [Abschnitt 20.5](20_Client_Authentifizierung.md#205-passwortauthentifizierung).

`gss`

Verwendet GSSAPI, um den Benutzer zu authentifizieren. Dies ist nur für TCP/IP-Verbindungen verfügbar. Details finden Sie in [Abschnitt 20.6](20_Client_Authentifizierung.md#206-gssapiauthentifizierung). Es kann zusammen mit GSSAPI-Verschlüsselung verwendet werden.

`sspi`

Verwendet SSPI, um den Benutzer zu authentifizieren. Dies ist nur unter Windows verfügbar. Details finden Sie in [Abschnitt 20.7](20_Client_Authentifizierung.md#207-sspiauthentifizierung).

`ident`

Ermittelt den Betriebssystembenutzernamen des Clients durch Kontaktaufnahme mit dem Ident-Server auf dem Client und prüft, ob er dem angeforderten Datenbankbenutzernamen entspricht. Ident-Authentifizierung kann nur für TCP/IP-Verbindungen verwendet werden. Wenn sie für lokale Verbindungen angegeben wird, wird stattdessen Peer-Authentifizierung verwendet. Details finden Sie in [Abschnitt 20.8](20_Client_Authentifizierung.md#208-identauthentifizierung).

`peer`

Ermittelt den Betriebssystembenutzernamen des Clients vom Betriebssystem und prüft, ob er dem angeforderten Datenbankbenutzernamen entspricht. Dies ist nur für lokale Verbindungen verfügbar. Details finden Sie in [Abschnitt 20.9](20_Client_Authentifizierung.md#209-peerauthentifizierung).

`ldap`

Authentifiziert über einen LDAP-Server. Details finden Sie in [Abschnitt 20.10](20_Client_Authentifizierung.md#2010-ldapauthentifizierung).

`radius`

Authentifiziert über einen RADIUS-Server. Details finden Sie in [Abschnitt 20.11](20_Client_Authentifizierung.md#2011-radiusauthentifizierung).

`cert`

Authentifiziert über SSL-Client-Zertifikate. Details finden Sie in [Abschnitt 20.12](20_Client_Authentifizierung.md#2012-zertifikatsauthentifizierung).

`pam`

Authentifiziert über den vom Betriebssystem bereitgestellten Dienst Pluggable Authentication Modules (PAM). Details finden Sie in [Abschnitt 20.13](20_Client_Authentifizierung.md#2013-pamauthentifizierung).

`bsd`

Authentifiziert über den vom Betriebssystem bereitgestellten BSD-Authentication-Dienst. Details finden Sie in [Abschnitt 20.14](20_Client_Authentifizierung.md#2014-bsdauthentifizierung).

`oauth`

Autorisiert und optional authentifiziert über einen OAuth-2.0-Identitätsanbieter eines Drittanbieters. Details finden Sie in [Abschnitt 20.15](20_Client_Authentifizierung.md#2015-oauthautorisierung-und-authentifizierung).

`auth-options`

Nach dem Feld `auth-method` können Felder der Form `name=value` folgen, die Optionen für die Authentifizierungsmethode angeben. Details dazu, welche Optionen für welche Authentifizierungsmethoden verfügbar sind, erscheinen weiter unten.

Zusätzlich zu den unten aufgeführten methodenspezifischen Optionen gibt es eine methodenunabhängige Authentifizierungsoption `clientcert`, die in jedem `hostssl`-Datensatz angegeben werden kann. Diese Option kann auf `verify-ca` oder `verify-full` gesetzt werden. Beide Optionen verlangen, dass der Client ein gültiges (vertrauenswürdiges) SSL-Zertifikat vorlegt, während `verify-full` zusätzlich erzwingt, dass der `cn` (Common Name) im Zertifikat dem Benutzernamen oder einer anwendbaren Zuordnung entspricht. Dieses Verhalten ähnelt der Authentifizierungsmethode `cert` (siehe [Abschnitt 20.12](20_Client_Authentifizierung.md#2012-zertifikatsauthentifizierung)), ermöglicht aber die Kombination der Prüfung von Client-Zertifikaten mit jeder Authentifizierungsmethode, die `hostssl`-Einträge unterstützt.

In jedem Datensatz, der Client-Zertifikatsauthentifizierung verwendet (also einem Datensatz mit der Authentifizierungsmethode `cert` oder einem Datensatz mit der Option `clientcert`), können Sie mit der Option `clientname` angeben, welcher Teil der Client-Zertifikatsanmeldedaten verglichen werden soll. Diese Option kann einen von zwei Werten haben. Wenn Sie `clientname=CN` angeben, was der Standard ist, wird der Benutzername gegen den Common Name (CN) des Zertifikats verglichen. Wenn Sie stattdessen `clientname=DN` angeben, wird der Benutzername gegen den gesamten Distinguished Name (DN) des Zertifikats verglichen. Diese Option wird wahrscheinlich am besten zusammen mit einer Benutzername-Zuordnung verwendet. Der Vergleich erfolgt mit dem DN im Format RFC 2253. Um den DN eines Client-Zertifikats in diesem Format zu sehen, führen Sie aus:

```text
openssl x509 -in myclient.crt -noout -subject -nameopt RFC2253 |
 sed "s/^subject=//"
```

Bei der Verwendung dieser Option ist Vorsicht geboten, besonders wenn reguläre Ausdrücke gegen den DN verwendet werden.

`include`

Diese Zeile wird durch den Inhalt der angegebenen Datei ersetzt.

`include_if_exists`

Diese Zeile wird durch den Inhalt der angegebenen Datei ersetzt, wenn die Datei existiert. Andernfalls wird eine Meldung geloggt, die angibt, dass die Datei übersprungen wurde.

`include_dir`

Diese Zeile wird durch den Inhalt aller im Verzeichnis gefundenen Dateien ersetzt, sofern diese nicht mit `.` beginnen und mit `.conf` enden; sie werden in Dateinamenreihenfolge verarbeitet (nach den Regeln der C-Locale, das heißt Zahlen vor Buchstaben und Großbuchstaben vor Kleinbuchstaben).

Dateien, die durch `@`-Konstrukte eingebunden werden, werden als Listen von Namen gelesen, die entweder durch Whitespace oder durch Kommas getrennt werden können. Kommentare werden wie in `pg_hba.conf` durch `#` eingeleitet, und verschachtelte `@`-Konstrukte sind erlaubt. Wenn der Dateiname nach `@` kein absoluter Pfad ist, wird er relativ zu dem Verzeichnis interpretiert, das die referenzierende Datei enthält.

Da die Datensätze in `pg_hba.conf` für jeden Verbindungsversuch sequenziell geprüft werden, ist ihre Reihenfolge wichtig. Typischerweise haben frühere Datensätze enge Verbindungsparameter und schwächere Authentifizierungsmethoden, während spätere Datensätze weiter gefasste Verbindungsparameter und stärkere Authentifizierungsmethoden haben. Beispielsweise möchte man vielleicht Trust-Authentifizierung für lokale TCP/IP-Verbindungen verwenden, aber für Remote-TCP/IP-Verbindungen ein Passwort verlangen. In diesem Fall würde ein Datensatz, der Trust-Authentifizierung für Verbindungen von `127.0.0.1` angibt, vor einem Datensatz stehen, der Passwortauthentifizierung für einen breiteren Bereich erlaubter Client-IP-Adressen angibt.

> Tipp: Um sich mit einer bestimmten Datenbank zu verbinden, muss ein Benutzer nicht nur die Prüfungen in `pg_hba.conf` bestehen, sondern auch das Privileg `CONNECT` für die Datenbank haben. Wenn Sie einschränken möchten, welche Benutzer sich mit welchen Datenbanken verbinden können, ist es normalerweise einfacher, dies durch Erteilen/Entziehen des Privilegs `CONNECT` zu steuern, als die Regeln in `pg_hba.conf`-Einträgen abzulegen.

<https://datatracker.ietf.org/doc/html/rfc2253>

Einige Beispiele für `pg_hba.conf`-Einträge werden in Beispiel 20.1 gezeigt. Details zu den verschiedenen Authentifizierungsmethoden finden Sie im nächsten Abschnitt.

Beispiel 20.1. Beispielhafte `pg_hba.conf`-Einträge

```text
# Jedem Benutzer auf dem lokalen System erlauben, sich über
# Unix-Domain-Sockets (der Standard für lokale Verbindungen)
# mit jeder Datenbank und jedem Datenbankbenutzernamen zu verbinden.
#
# TYPE DATABASE         USER            ADDRESS                 METHOD
local    all            all                                     trust

# Dasselbe über lokale Loopback-TCP/IP-Verbindungen.
#
# TYPE DATABASE         USER            ADDRESS                 METHOD
host     all            all             127.0.0.1/32            trust

# Dasselbe wie in der vorherigen Zeile, aber mit separater Netzmaskenspalte.
#
# TYPE DATABASE         USER            IP-ADDRESS      IP-MASK             METHOD
host     all            all             127.0.0.1       255.255.255.255     trust

# Dasselbe über IPv6.
#
# TYPE DATABASE                   USER                   ADDRESS            METHOD
host     all                      all                    ::1/128            trust

# Dasselbe mit einem Hostnamen (würde typischerweise sowohl IPv4 als auch IPv6 abdecken).
#
# TYPE DATABASE         USER            ADDRESS                 METHOD
host     all            all             localhost               trust

# Dasselbe mit einem regulären Ausdruck für DATABASE, der Verbindungen
# zu allen Datenbanken erlaubt, deren Name mit "db" beginnt und mit
# einer zwei- bis vierstelligen Zahl endet (wie "db1234" oder "db12").
#
# TYPE DATABASE         USER            ADDRESS                 METHOD
host     "/^db\d{2,4}$" all             localhost               trust

# Jedem Benutzer von jedem Host mit IP-Adresse 192.168.93.x erlauben,
# sich mit der Datenbank "postgres" unter demselben Benutzernamen zu
# verbinden, den ident für die Verbindung meldet (typischerweise der
# Betriebssystembenutzername).
#
# TYPE DATABASE         USER            ADDRESS                 METHOD
host     postgres       all             192.168.93.0/24         ident

# Jedem Benutzer von Host 192.168.12.10 erlauben, sich mit Datenbank
# "postgres" zu verbinden, wenn das Passwort des Benutzers korrekt
# angegeben wird.
#
# TYPE DATABASE         USER            ADDRESS                 METHOD
host     postgres       all             192.168.12.10/32        scram-sha-256

# Jedem Benutzer von Hosts in der Domain example.com erlauben, sich mit
# jeder Datenbank zu verbinden, wenn das Passwort des Benutzers korrekt
# angegeben wird.
#
# Für die meisten Benutzer SCRAM-Authentifizierung verlangen, aber eine
# Ausnahme für Benutzer 'mike' machen, der einen älteren Client verwendet,
# der SCRAM-Authentifizierung nicht unterstützt.
#
# TYPE DATABASE         USER            ADDRESS                 METHOD
host     all            mike            .example.com            md5
host     all            all             .example.com            scram-sha-256

# Wenn keine vorangehenden "host"-Zeilen vorhanden sind, lehnen diese drei
# Zeilen alle Verbindungen von 192.168.54.1 ab (weil dieser Eintrag zuerst
# passt), erlauben aber GSSAPI-verschlüsselte Verbindungen von überall sonst
# im Internet. Die Nullmaske bewirkt, dass keine Bits der Host-IP-Adresse
# berücksichtigt werden, sodass jeder Host passt. Unverschlüsselte
# GSSAPI-Verbindungen (die zur dritten Zeile "durchfallen", weil "hostgssenc"
# nur auf verschlüsselte GSSAPI-Verbindungen passt) sind erlaubt, aber nur
# von 192.168.12.10.
#
# TYPE DATABASE         USER            ADDRESS                 METHOD
host      all           all             192.168.54.1/32         reject
hostgssenc all          all             0.0.0.0/0               gss
host      all           all             192.168.12.10/32        gss

# Benutzern von 192.168.x.x-Hosts erlauben, sich mit jeder Datenbank zu
# verbinden, wenn sie die ident-Prüfung bestehen. Wenn ident zum Beispiel
# den Benutzer "bryanh" meldet und dieser anfordert, sich als PostgreSQL-
# Benutzer "guest1" zu verbinden, ist die Verbindung erlaubt, wenn es in
# pg_ident.conf für die Zuordnung "omicron" einen Eintrag gibt, der besagt,
# dass "bryanh" sich als "guest1" verbinden darf.
#
# TYPE DATABASE         USER            ADDRESS                 METHOD
host     all            all             192.168.0.0/16          ident map=omicron

# Wenn dies die einzigen vier Zeilen für lokale Verbindungen sind, erlauben
# sie lokalen Benutzern nur Verbindungen zu ihren eigenen Datenbanken
# (Datenbanken mit demselben Namen wie ihr Datenbankbenutzername), außer
# Benutzern, deren Name mit "helpdesk" endet, Administratoren und Mitgliedern
# der Rolle "support", die sich mit allen Datenbanken verbinden können.
# Die Datei $PGDATA/admins enthält eine Namensliste von Administratoren.
# Passwörter sind in allen Fällen erforderlich.
#
# TYPE DATABASE         USER            ADDRESS                 METHOD
local    sameuser       all                                     md5
local    all            /^.*helpdesk$                           md5
local    all            @admins                                 md5
local    all            +support                                md5

# Die letzten beiden Zeilen oben können zu einer einzigen Zeile kombiniert werden:
local   all             @admins,+support                        md5

# Die Datenbankspalte kann ebenfalls Listen und Dateinamen verwenden:
local   db1,db2,@demodbs all                                    md5
```

## 20.2. Benutzername-Zuordnungen

Bei Verwendung eines externen Authentifizierungssystems wie Ident oder GSSAPI ist der Name des Betriebssystembenutzers, der die Verbindung initiiert hat, möglicherweise nicht derselbe wie der zu verwendende Datenbankbenutzer (die Rolle). In diesem Fall kann eine Benutzername-Zuordnung angewendet werden, um den Betriebssystembenutzernamen einem Datenbankbenutzer zuzuordnen. Um Benutzername-Zuordnung zu verwenden, geben Sie `map=map-name` im Optionsfeld von `pg_hba.conf` an. Diese Option wird für alle Authentifizierungsmethoden unterstützt, die externe Benutzernamen empfangen. Da für verschiedene Verbindungen unterschiedliche Zuordnungen benötigt werden können, wird der Name der zu verwendenden Zuordnung im Parameter `map-name` in `pg_hba.conf` angegeben, um festzulegen, welche Zuordnung für jede einzelne Verbindung verwendet werden soll.

Benutzername-Zuordnungen werden in der Ident-Map-Datei definiert, die standardmäßig `pg_ident.conf` heißt und im Datenverzeichnis des Clusters gespeichert ist. (Es ist jedoch möglich, die Map-Datei an anderer Stelle abzulegen; siehe den Konfigurationsparameter `ident_file`.) Die Ident-Map-Datei enthält Zeilen der allgemeinen Formen:

```text
map-name system-username database-username
include file
include_if_exists file
include_dir directory
```

Kommentare, Whitespace und Zeilenfortsetzungen werden genauso behandelt wie in `pg_hba.conf`. `map-name` ist ein beliebiger Name, der verwendet wird, um in `pg_hba.conf` auf diese Zuordnung zu verweisen. Die anderen beiden Felder geben einen Betriebssystembenutzernamen und einen passenden Datenbankbenutzernamen an. Derselbe `map-name` kann wiederholt verwendet werden, um mehrere Benutzerzuordnungen innerhalb einer einzigen Map anzugeben.

Wie bei `pg_hba.conf` können die Zeilen in dieser Datei Include-Direktiven sein, die denselben Regeln folgen.

Die Datei `pg_ident.conf` wird beim Start und immer dann gelesen, wenn der Hauptserverprozess ein `SIGHUP`-Signal empfängt. Wenn Sie die Datei auf einem aktiven System bearbeiten, müssen Sie dem Postmaster ein Signal senden (mit `pg_ctl reload`, durch Aufruf der SQL-Funktion `pg_reload_conf()` oder mit `kill -HUP`), damit er die Datei erneut liest.

Die System-View `pg_ident_file_mappings` kann hilfreich sein, um Änderungen an der Datei `pg_ident.conf` vorab zu testen oder Probleme zu diagnostizieren, wenn das Laden der Datei nicht die gewünschten Auswirkungen hatte. Zeilen in der View mit nicht-null Fehlerfeldern weisen auf Probleme in den entsprechenden Zeilen der Datei hin.

Es gibt keine Beschränkung, wie vielen Datenbankbenutzern ein bestimmter Betriebssystembenutzer entsprechen kann, und umgekehrt. Daher sollten Einträge in einer Map als „dieser Betriebssystembenutzer darf sich als dieser Datenbankbenutzer verbinden“ verstanden werden, nicht als Aussage, dass sie äquivalent sind. Die Verbindung wird erlaubt, wenn es irgendeinen Map-Eintrag gibt, der den vom externen Authentifizierungssystem erhaltenen Benutzernamen mit dem Datenbankbenutzernamen paart, als den der Benutzer sich verbinden möchte. Der Wert `all` kann als `database-username` verwendet werden, um anzugeben, dass dieser Benutzer sich als jeder der existierenden Datenbankbenutzer anmelden darf, wenn `system-username` passt. Wird `all` quotiert, verliert das Schlüsselwort seine besondere Bedeutung.

Wenn `database-username` mit einem `+`-Zeichen beginnt, kann der Betriebssystembenutzer sich als jeder Benutzer anmelden, der zu dieser Rolle gehört, ähnlich wie Benutzernamen mit vorangestelltem `+` in `pg_hba.conf` behandelt werden. Ein `+` bedeutet also „passe auf jede Rolle, die direkt oder indirekt Mitglied dieser Rolle ist“, während ein Name ohne `+` nur auf genau diese Rolle passt. Wird ein Benutzername quotiert, der mit `+` beginnt, verliert das `+` seine besondere Bedeutung.

Wenn das Feld `system-username` mit einem Schrägstrich (`/`) beginnt, wird der Rest des Feldes als regulärer Ausdruck behandelt. Details zur Syntax regulärer Ausdrücke in PostgreSQL finden Sie in Abschnitt 9.7.3.1. Der reguläre Ausdruck kann eine einzelne Capture-Gruppe oder einen geklammerten Teilausdruck enthalten. Der Teil des Systembenutzernamens, der auf die Capture-Gruppe passte, kann dann im Feld `database-username` als `\1` (Backslash-eins) referenziert werden. Dies erlaubt die Zuordnung mehrerer Benutzernamen in einer einzigen Zeile, was besonders für einfache Syntaxersetzungen nützlich ist. Beispielsweise entfernen diese Einträge

```text
mymap       /^(.*)@mydomain\.com$                     \1
mymap       /^(.*)@otherdomain\.com$                  guest
```

den Domainteil für Benutzer mit Systembenutzernamen, die auf `@mydomain.com` enden, und erlauben jedem Benutzer, dessen Systemname auf `@otherdomain.com` endet, sich als `guest` anzumelden. Das Quotieren eines `database-username`, der `\1` enthält, nimmt `\1` nicht seine besondere Bedeutung.

Wenn das Feld `database-username` mit einem Schrägstrich (`/`) beginnt, wird der Rest des Feldes als regulärer Ausdruck behandelt. Wenn das Feld `database-username` ein regulärer Ausdruck ist, ist es nicht möglich, darin `\1` zu verwenden, um auf eine Capture-Gruppe aus dem Feld `system-username` zu verweisen.

> Tipp: Denken Sie daran, dass ein regulärer Ausdruck standardmäßig nur auf einen Teil eines Strings passen kann. Es ist normalerweise sinnvoll, `^` und `$` zu verwenden, wie im obigen Beispiel, um zu erzwingen, dass der gesamte Systembenutzername passt.

Eine Datei `pg_ident.conf`, die zusammen mit der Datei `pg_hba.conf` in Beispiel 20.1 verwendet werden könnte, wird in Beispiel 20.2 gezeigt. In diesem Beispiel würde niemand, der auf einer Maschine im Netzwerk `192.168` angemeldet ist und nicht den Betriebssystembenutzernamen `bryanh`, `ann` oder `robert` hat, Zugriff erhalten. Der Unix-Benutzer `robert` dürfte nur zugreifen, wenn er versucht, sich als PostgreSQL-Benutzer `bob` zu verbinden, nicht als `robert` oder jemand anders. `ann` dürfte sich nur als `ann` verbinden. Benutzer `bryanh` dürfte sich entweder als `bryanh` oder als `guest1` verbinden.

Beispiel 20.2. Eine beispielhafte Datei `pg_ident.conf`

```text
# MAPNAME        SYSTEM-USERNAME        PG-USERNAME

omicron          bryanh                 bryanh
omicron          ann                    ann
# bob hat auf diesen Maschinen den Benutzernamen robert
omicron          robert                 bob
# bryanh kann sich auch als guest1 verbinden
omicron          bryanh                 guest1
```

## 20.3. Authentifizierungsmethoden

PostgreSQL stellt verschiedene Methoden zur Authentifizierung von Benutzern bereit:

- Trust-Authentifizierung, die einfach darauf vertraut, dass Benutzer die sind, für die sie sich ausgeben.
- Passwortauthentifizierung, die verlangt, dass Benutzer ein Passwort senden.
- GSSAPI-Authentifizierung, die auf einer GSSAPI-kompatiblen Sicherheitsbibliothek beruht. Typischerweise wird dies verwendet, um auf einen Authentifizierungsserver wie einen Kerberos- oder Microsoft-Active-Directory-Server zuzugreifen.
- SSPI-Authentifizierung, die ein Windows-spezifisches Protokoll ähnlich GSSAPI verwendet.
- Ident-Authentifizierung, die auf einem Dienst „Identification Protocol“ (RFC 1413) auf der Maschine des Clients beruht. (Bei lokalen Unix-Socket-Verbindungen wird dies als Peer-Authentifizierung behandelt.)
- Peer-Authentifizierung, die Betriebssystemfunktionen nutzt, um den Prozess am anderen Ende einer lokalen Verbindung zu identifizieren. Dies wird für Remote-Verbindungen nicht unterstützt.
- LDAP-Authentifizierung, die auf einem LDAP-Authentifizierungsserver beruht.
- RADIUS-Authentifizierung, die auf einem RADIUS-Authentifizierungsserver beruht.
- Zertifikatsauthentifizierung, die eine SSL-Verbindung erfordert und Benutzer authentifiziert, indem das von ihnen gesendete SSL-Zertifikat geprüft wird.
- PAM-Authentifizierung, die auf einer PAM-Bibliothek (Pluggable Authentication Modules) beruht.
- BSD-Authentifizierung, die auf dem BSD-Authentication-Framework beruht (derzeit nur unter OpenBSD verfügbar).
- OAuth-Autorisierung/-Authentifizierung, die auf einem externen OAuth-2.0-Identitätsanbieter beruht.

Peer-Authentifizierung ist normalerweise für lokale Verbindungen empfehlenswert, auch wenn Trust-Authentifizierung unter bestimmten Umständen ausreichen kann. Passwortauthentifizierung ist die einfachste Wahl für Remote-Verbindungen. Alle anderen Optionen erfordern irgendeine Art externer Sicherheitsinfrastruktur (normalerweise einen Authentifizierungsserver oder eine Zertifizierungsstelle zur Ausgabe von SSL-Zertifikaten) oder sind plattformspezifisch.

Die folgenden Abschnitte beschreiben jede dieser Authentifizierungsmethoden ausführlicher.

## 20.4. Trust-Authentifizierung

Wenn Trust-Authentifizierung angegeben ist, nimmt PostgreSQL an, dass jeder, der eine Verbindung zum Server herstellen kann, berechtigt ist, mit dem angegebenen Datenbankbenutzernamen auf die Datenbank zuzugreifen (sogar mit Superuser-Namen). Natürlich gelten weiterhin Einschränkungen, die in den Spalten `database` und `user` vorgenommen wurden. Diese Methode sollte nur verwendet werden, wenn Verbindungen zum Server auf Betriebssystemebene ausreichend geschützt sind.

Trust-Authentifizierung ist für lokale Verbindungen auf einer Einzelbenutzer-Workstation geeignet und sehr bequem. Auf einer Mehrbenutzermaschine ist sie für sich genommen normalerweise nicht geeignet. Sie können `trust` jedoch möglicherweise auch auf einer Mehrbenutzermaschine verwenden, wenn Sie den Zugriff auf die Unix-Domain-Socket-Datei des Servers mit Dateisystemberechtigungen einschränken. Setzen Sie dazu die Konfigurationsparameter `unix_socket_permissions` (und möglicherweise `unix_socket_group`), wie in [Abschnitt 19.3](19_Serverkonfiguration.md#193-verbindungen-und-authentifizierung) beschrieben. Oder Sie setzen den Konfigurationsparameter `unix_socket_directories`, um die Socket-Datei in einem geeignet eingeschränkten Verzeichnis abzulegen.

Das Setzen von Dateisystemberechtigungen hilft nur bei Unix-Socket-Verbindungen. Lokale TCP/IP-Verbindungen werden nicht durch Dateisystemberechtigungen eingeschränkt. Wenn Sie also Dateisystemberechtigungen für lokale Sicherheit verwenden möchten, entfernen Sie die Zeile `host ... 127.0.0.1 ...` aus `pg_hba.conf` oder ändern Sie sie auf eine andere Authentifizierungsmethode als `trust`.

Trust-Authentifizierung ist für TCP/IP-Verbindungen nur geeignet, wenn Sie jedem Benutzer auf jeder Maschine vertrauen, der durch die `pg_hba.conf`-Zeilen, die `trust` angeben, eine Verbindung zum Server herstellen darf. Es ist selten sinnvoll, `trust` für andere TCP/IP-Verbindungen zu verwenden als solche von `localhost` (`127.0.0.1`).

## 20.5. Passwortauthentifizierung

Es gibt mehrere passwortbasierte Authentifizierungsmethoden. Diese Methoden funktionieren ähnlich, unterscheiden sich aber darin, wie die Passwörter der Benutzer auf dem Server gespeichert werden und wie das von einem Client bereitgestellte Passwort über die Verbindung gesendet wird.

`scram-sha-256`

Die Methode `scram-sha-256` führt SCRAM-SHA-256-Authentifizierung aus, wie in RFC 7677 beschrieben. Sie ist ein Challenge-Response-Verfahren, das Passwort-Sniffing auf nicht vertrauenswürdigen Verbindungen verhindert und die Speicherung von Passwörtern auf dem Server in einer kryptographisch gehashten Form unterstützt, die als sicher gilt.

Dies ist die sicherste der derzeit bereitgestellten Methoden, wird aber von älteren Client-Bibliotheken nicht unterstützt.

`md5`

Die Methode `md5` verwendet einen eigenen, weniger sicheren Challenge-Response-Mechanismus. Sie verhindert Passwort-Sniffing und vermeidet das Speichern von Passwörtern auf dem Server im Klartext, bietet aber keinen Schutz, wenn ein Angreifer den Passwort-Hash vom Server stehlen kann. Außerdem gilt der MD5-Hashalgorithmus heute nicht mehr als sicher gegen entschlossene Angriffe.

Um den Übergang von der Methode `md5` zur neueren SCRAM-Methode zu erleichtern, wird automatisch stattdessen SCRAM-basierte Authentifizierung gewählt, wenn `md5` als Methode in `pg_hba.conf` angegeben ist, das Passwort des Benutzers auf dem Server aber für SCRAM verschlüsselt ist (siehe unten).

> Warnung: Unterstützung für MD5-verschlüsselte Passwörter ist veraltet und wird in einer zukünftigen PostgreSQL-Version entfernt. Details zur Migration auf einen anderen Passworttyp finden Sie im folgenden Text.

`password`

Die Methode `password` sendet das Passwort im Klartext und ist daher anfällig für Passwort-„Sniffing“-Angriffe. Sie sollte nach Möglichkeit immer vermieden werden. Wenn die Verbindung durch SSL-Verschlüsselung geschützt ist, kann `password` jedoch sicher verwendet werden. (SSL-Zertifikatsauthentifizierung könnte allerdings die bessere Wahl sein, wenn man ohnehin auf SSL angewiesen ist.)

<https://datatracker.ietf.org/doc/html/rfc7677>

PostgreSQL-Datenbankpasswörter sind von Betriebssystembenutzerpasswörtern getrennt. Das Passwort jedes Datenbankbenutzers wird im Systemkatalog `pg_authid` gespeichert. Passwörter können mit den SQL-Befehlen `CREATE ROLE` und `ALTER ROLE` verwaltet werden, zum Beispiel:

```sql
CREATE ROLE foo WITH LOGIN PASSWORD 'secret';
```

oder mit dem `psql`-Befehl `\password`. Wenn für einen Benutzer kein Passwort eingerichtet wurde, ist das gespeicherte Passwort null, und Passwortauthentifizierung schlägt für diesen Benutzer immer fehl.

Die Verfügbarkeit der verschiedenen passwortbasierten Authentifizierungsmethoden hängt davon ab, wie das Passwort eines Benutzers auf dem Server verschlüsselt (genauer: gehasht) ist. Dies wird zum Zeitpunkt des Setzens des Passworts durch den Konfigurationsparameter `password_encryption` gesteuert. Wenn ein Passwort mit der Einstellung `scram-sha-256` verschlüsselt wurde, kann es für die Authentifizierungsmethoden `scram-sha-256` und `password` verwendet werden (im letzteren Fall wird das Passwort allerdings im Klartext übertragen). Die Authentifizierungsmethodenspezifikation `md5` wechselt in diesem Fall automatisch zur Methode `scram-sha-256`, wie oben erläutert, und funktioniert daher ebenfalls. Wenn ein Passwort mit der Einstellung `md5` verschlüsselt wurde, kann es nur für die Authentifizierungsmethodenspezifikationen `md5` und `password` verwendet werden (wiederum mit Klartextübertragung des Passworts im letzteren Fall). (Frühere PostgreSQL-Versionen unterstützten das Speichern des Passworts auf dem Server im Klartext. Das ist nicht mehr möglich.) Um die derzeit gespeicherten Passwort-Hashes zu prüfen, sehen Sie im Systemkatalog `pg_authid` nach.

Um eine bestehende Installation von `md5` auf `scram-sha-256` zu aktualisieren, stellen Sie zunächst sicher, dass alle verwendeten Client-Bibliotheken neu genug sind, um SCRAM zu unterstützen. Setzen Sie dann `password_encryption = 'scram-sha-256'` in `postgresql.conf`, lassen Sie alle Benutzer neue Passwörter setzen und ändern Sie die Authentifizierungsmethodenspezifikationen in `pg_hba.conf` auf `scram-sha-256`.

## 20.6. GSSAPI-Authentifizierung

GSSAPI ist ein Industriestandardprotokoll für sichere Authentifizierung, definiert in RFC 2743. PostgreSQL unterstützt GSSAPI für Authentifizierung, Kommunikationsverschlüsselung oder beides. GSSAPI bietet automatische Authentifizierung (Single Sign-on) für Systeme, die dies unterstützen. Die Authentifizierung selbst ist sicher. Wenn GSSAPI-Verschlüsselung oder SSL-Verschlüsselung verwendet wird, werden die über die Datenbankverbindung gesendeten Daten verschlüsselt; andernfalls nicht.

GSSAPI-Unterstützung muss beim Bau von PostgreSQL aktiviert werden; weitere Informationen finden Sie in [Kapitel 17](17_Installation_aus_dem_Quellcode.md).

Wenn GSSAPI Kerberos verwendet, nutzt es einen standardmäßigen Service Principal (Authentifizierungsidentität) im Format `servicename/hostname@realm`. Der Principal-Name, den eine bestimmte Installation verwendet, ist im PostgreSQL-Server in keiner Weise fest kodiert; vielmehr wird er in der Keytab-Datei angegeben, die der Server liest, um seine Identität zu bestimmen. Wenn mehrere Principals in der Keytab-Datei aufgeführt sind, akzeptiert der Server jeden davon. Der Realm-Name des Servers ist der bevorzugte Realm, der in den für den Server zugänglichen Kerberos-Konfigurationsdateien angegeben ist.

Beim Verbinden muss der Client den Principal-Namen des Servers kennen, mit dem er sich verbinden möchte. Der Teil `servicename` des Principal ist gewöhnlich `postgres`, aber über den libpq-Verbindungsparameter `krbsrvname` kann ein anderer Wert gewählt werden. Der Teil `hostname` ist der vollständig qualifizierte Hostname, mit dem sich libpq verbinden soll. Der Realm-Name ist der bevorzugte Realm, der in den für den Client zugänglichen Kerberos-Konfigurationsdateien angegeben ist.

Der Client hat außerdem einen Principal-Namen für seine eigene Identität (und muss ein gültiges Ticket für diesen Principal besitzen). Um GSSAPI für Authentifizierung zu verwenden, muss der Client-Principal mit einem PostgreSQL-Datenbankbenutzernamen verknüpft sein. Die Konfigurationsdatei `pg_ident.conf` kann verwendet werden, um Principals Benutzernamen zuzuordnen; zum Beispiel könnte `pgusername@realm` einfach auf `pgusername` abgebildet werden. Alternativ können Sie den vollständigen Principal `username@realm` ohne Zuordnung als Rollennamen in PostgreSQL verwenden.

PostgreSQL unterstützt auch das Zuordnen von Client-Principals zu Benutzernamen, indem lediglich der Realm vom Principal entfernt wird. Diese Methode wird aus Gründen der Abwärtskompatibilität unterstützt und ist dringend abzuraten, weil es dann unmöglich ist, verschiedene Benutzer mit demselben Benutzernamen, aber aus unterschiedlichen Realms zu unterscheiden. Um dies zu aktivieren, setzen Sie `include_realm` auf `0`. Für einfache Single-Realm-Installationen ist dies in Kombination mit dem Setzen des Parameters `krb_realm` (der prüft, dass der Realm des Principal genau dem Wert in `krb_realm` entspricht) weiterhin sicher; verglichen mit der Angabe einer expliziten Zuordnung in `pg_ident.conf` ist dies aber ein weniger leistungsfähiger Ansatz.

<https://datatracker.ietf.org/doc/html/rfc2743>

Der Speicherort der Keytab-Datei des Servers wird durch den Konfigurationsparameter `krb_server_keyfile` angegeben. Aus Sicherheitsgründen wird empfohlen, eine separate Keytab nur für den PostgreSQL-Server zu verwenden, statt dem Server zu erlauben, die System-Keytab-Datei zu lesen. Stellen Sie sicher, dass Ihre Server-Keytab-Datei für das PostgreSQL-Serverkonto lesbar ist (und vorzugsweise nur lesbar, nicht schreibbar). Siehe auch [Abschnitt 18.1](18_Servereinrichtung_und_Betrieb.md#181-das-postgresqlbenutzerkonto).

Die Keytab-Datei wird mit der Kerberos-Software erzeugt; Details finden Sie in der Kerberos-Dokumentation. Das folgende Beispiel zeigt dies mit dem Werkzeug `kadmin` von MIT Kerberos:

```text
kadmin% addprinc -randkey postgres/server.my.domain.org
kadmin% ktadd -k krb5.keytab postgres/server.my.domain.org
```

Die folgenden Authentifizierungsoptionen werden für die GSSAPI-Authentifizierungsmethode unterstützt:

`include_realm`

Wenn auf `0` gesetzt, wird der Realm-Name aus dem authentifizierten Benutzer-Principal entfernt, bevor er durch die Benutzername-Zuordnung läuft ([Abschnitt 20.2](20_Client_Authentifizierung.md#202-benutzernamezuordnungen)). Davon wird abgeraten; dies ist in erster Linie aus Gründen der Abwärtskompatibilität verfügbar, da es in Multi-Realm-Umgebungen nicht sicher ist, sofern nicht auch `krb_realm` verwendet wird. Es wird empfohlen, `include_realm` auf dem Standardwert (`1`) zu belassen und eine explizite Zuordnung in `pg_ident.conf` bereitzustellen, um Principal-Namen in PostgreSQL-Benutzernamen umzuwandeln.

`map`

Erlaubt die Zuordnung von Client-Principals zu Datenbankbenutzernamen. Details finden Sie in [Abschnitt 20.2](20_Client_Authentifizierung.md#202-benutzernamezuordnungen). Für einen GSSAPI/Kerberos-Principal wie `username@EXAMPLE.COM` (oder, seltener, `username/hostbased@EXAMPLE.COM`) ist der für die Zuordnung verwendete Benutzername `username@EXAMPLE.COM` (beziehungsweise `username/hostbased@EXAMPLE.COM`), sofern `include_realm` nicht auf `0` gesetzt wurde; in diesem Fall erscheint `username` (beziehungsweise `username/hostbased`) bei der Zuordnung als Systembenutzername.

`krb_realm`

Setzt den Realm, gegen den Benutzer-Principal-Namen geprüft werden. Wenn dieser Parameter gesetzt ist, werden nur Benutzer dieses Realms akzeptiert. Wenn er nicht gesetzt ist, können sich Benutzer jedes Realms verbinden, vorbehaltlich der jeweils durchgeführten Benutzername-Zuordnung.

Zusätzlich zu diesen Einstellungen, die sich für verschiedene `pg_hba.conf`-Einträge unterscheiden können, gibt es den serverweiten Konfigurationsparameter `krb_caseins_users`. Wenn dieser auf `true` gesetzt ist, werden Client-Principals case-insensitiv gegen User-Map-Einträge abgeglichen. `krb_realm` wird, falls gesetzt, ebenfalls case-insensitiv verglichen.

## 20.7. SSPI-Authentifizierung

SSPI ist eine Windows-Technologie für sichere Authentifizierung mit Single Sign-on. PostgreSQL verwendet SSPI im Negotiate-Modus, der Kerberos verwendet, wenn möglich, und in anderen Fällen automatisch auf NTLM zurückfällt. SSPI und GSSAPI arbeiten als Clients und Server zusammen; zum Beispiel kann sich ein SSPI-Client an einem GSSAPI-Server authentifizieren. Es wird empfohlen, SSPI auf Windows-Clients und -Servern und GSSAPI auf Nicht-Windows-Plattformen zu verwenden.

Bei Verwendung von Kerberos-Authentifizierung funktioniert SSPI genauso wie GSSAPI; Details finden Sie in [Abschnitt 20.6](20_Client_Authentifizierung.md#206-gssapiauthentifizierung).

Die folgenden Konfigurationsoptionen werden für SSPI unterstützt:

`include_realm`

Wenn auf `0` gesetzt, wird der Realm-Name aus dem authentifizierten Benutzer-Principal entfernt, bevor er durch die Benutzername-Zuordnung läuft ([Abschnitt 20.2](20_Client_Authentifizierung.md#202-benutzernamezuordnungen)). Davon wird abgeraten; dies ist in erster Linie aus Gründen der Abwärtskompatibilität verfügbar, da es in Multi-Realm-Umgebungen nicht sicher ist, sofern nicht auch `krb_realm` verwendet wird. Es wird empfohlen, `include_realm` auf dem Standardwert (`1`) zu belassen und eine explizite Zuordnung in `pg_ident.conf` bereitzustellen, um Principal-Namen in PostgreSQL-Benutzernamen umzuwandeln.

`compat_realm`

Wenn auf `1` gesetzt, wird der SAM-kompatible Name der Domain (auch als NetBIOS-Name bekannt) für die Option `include_realm` verwendet. Dies ist der Standard. Wenn auf `0` gesetzt, wird der tatsächliche Realm-Name aus dem Kerberos-Benutzer-Principal-Namen verwendet.

Deaktivieren Sie diese Option nur, wenn Ihr Server unter einem Domain-Konto läuft (dazu gehören virtuelle Dienstkonten auf einem Domain-Member-System) und alle Clients, die sich über SSPI authentifizieren, ebenfalls Domain-Konten verwenden; andernfalls schlägt die Authentifizierung fehl.

`upn_username`

Wenn diese Option zusammen mit `compat_realm` aktiviert ist, wird der Benutzername aus dem Kerberos-UPN für die Authentifizierung verwendet. Wenn sie deaktiviert ist (der Standard), wird der SAM-kompatible Benutzername verwendet. Standardmäßig sind diese beiden Namen für neue Benutzerkonten identisch.

Beachten Sie, dass libpq den SAM-kompatiblen Namen verwendet, wenn kein expliziter Benutzername angegeben ist. Wenn Sie libpq oder einen darauf basierenden Treiber verwenden, sollten Sie diese Option deaktiviert lassen oder den Benutzernamen explizit in der Verbindungszeichenkette angeben.

`map`

Erlaubt die Zuordnung zwischen System- und Datenbankbenutzernamen. Details finden Sie in [Abschnitt 20.2](20_Client_Authentifizierung.md#202-benutzernamezuordnungen). Für einen SSPI/Kerberos-Principal wie `username@EXAMPLE.COM` (oder, seltener, `username/hostbased@EXAMPLE.COM`) ist der für die Zuordnung verwendete Benutzername `username@EXAMPLE.COM` (beziehungsweise `username/hostbased@EXAMPLE.COM`), sofern `include_realm` nicht auf `0` gesetzt wurde; in diesem Fall erscheint `username` (beziehungsweise `username/hostbased`) bei der Zuordnung als Systembenutzername.

`krb_realm`

Setzt den Realm, gegen den Benutzer-Principal-Namen geprüft werden. Wenn dieser Parameter gesetzt ist, werden nur Benutzer dieses Realms akzeptiert. Wenn er nicht gesetzt ist, können sich Benutzer jedes Realms verbinden, vorbehaltlich der jeweils durchgeführten Benutzername-Zuordnung.

## 20.8. Ident-Authentifizierung

Die Ident-Authentifizierungsmethode funktioniert, indem sie den Betriebssystembenutzernamen des Clients von einem Ident-Server abfragt und ihn als erlaubten Datenbankbenutzernamen verwendet (mit optionaler Benutzername-Zuordnung). Dies wird nur für TCP/IP-Verbindungen unterstützt.

> Hinweis: Wenn `ident` für eine lokale Verbindung (nicht TCP/IP) angegeben wird, wird stattdessen Peer-Authentifizierung verwendet (siehe Abschnitt 20.9).

Die folgenden Konfigurationsoptionen werden für Ident unterstützt:

`map`

Erlaubt die Zuordnung zwischen System- und Datenbankbenutzernamen. Details finden Sie in [Abschnitt 20.2](20_Client_Authentifizierung.md#202-benutzernamezuordnungen).

Das „Identification Protocol“ ist in RFC 1413 beschrieben. Praktisch jedes Unix-ähnliche Betriebssystem wird mit einem Ident-Server ausgeliefert, der standardmäßig auf TCP-Port 113 lauscht. Die grundlegende Funktion eines Ident-Servers besteht darin, Fragen wie „Welcher Benutzer hat die Verbindung gestartet, die aus deinem Port X herausgeht und sich mit meinem Port Y verbindet?“ zu beantworten. Da PostgreSQL sowohl X als auch Y kennt, wenn eine physische Verbindung aufgebaut wird, kann es den Ident-Server auf dem Host des verbindenden Clients befragen und theoretisch den Betriebssystembenutzer für jede gegebene Verbindung bestimmen.

Der Nachteil dieses Verfahrens ist, dass es von der Integrität des Clients abhängt: Wenn die Client-Maschine nicht vertrauenswürdig oder kompromittiert ist, könnte ein Angreifer praktisch jedes Programm auf Port 113 ausführen und einen beliebigen Benutzernamen zurückgeben. Diese Authentifizierungsmethode ist daher nur für geschlossene Netzwerke geeignet, in denen jede Client-Maschine streng kontrolliert wird und Datenbank- und Systemadministratoren eng zusammenarbeiten. Mit anderen Worten: Sie müssen der Maschine vertrauen, auf der der Ident-Server läuft. Beachten Sie die Warnung:

> Das Identification Protocol ist nicht als Autorisierungs- oder Zugriffskontrollprotokoll gedacht.
>
> —RFC 1413

Einige Ident-Server haben eine nicht standardmäßige Option, die dazu führt, dass der zurückgegebene Benutzername mit einem Schlüssel verschlüsselt wird, den nur der Administrator der Ursprungsmaschine kennt. Diese Option darf bei Verwendung des Ident-Servers mit PostgreSQL nicht genutzt werden, da PostgreSQL keine Möglichkeit hat, den zurückgegebenen String zu entschlüsseln, um den tatsächlichen Benutzernamen zu bestimmen.

## 20.9. Peer-Authentifizierung

Die Peer-Authentifizierungsmethode funktioniert, indem sie den Betriebssystembenutzernamen des Clients vom Kernel abfragt und ihn als erlaubten Datenbankbenutzernamen verwendet (mit optionaler Benutzername-Zuordnung). Diese Methode wird nur für lokale Verbindungen unterstützt.

Die folgenden Konfigurationsoptionen werden für Peer unterstützt:

`map`

Erlaubt die Zuordnung zwischen System- und Datenbankbenutzernamen. Details finden Sie in [Abschnitt 20.2](20_Client_Authentifizierung.md#202-benutzernamezuordnungen).

Peer-Authentifizierung ist nur auf Betriebssystemen verfügbar, die die Funktion `getpeereid()`, den Socket-Parameter `SO_PEERCRED` oder ähnliche Mechanismen bereitstellen. Derzeit umfasst dies Linux, die meisten BSD-Varianten einschließlich macOS sowie Solaris.

## 20.10. LDAP-Authentifizierung

Diese Authentifizierungsmethode funktioniert ähnlich wie `password`, verwendet jedoch LDAP als Methode zur Passwortprüfung. LDAP wird nur verwendet, um Benutzername/Passwort-Paare zu validieren. Daher muss der Benutzer bereits in der Datenbank existieren, bevor LDAP zur Authentifizierung verwendet werden kann.

LDAP-Authentifizierung kann in zwei Modi arbeiten. Im ersten Modus, den wir Simple-Bind-Modus nennen, bindet der Server an den Distinguished Name, der als `prefix` `username` `suffix` konstruiert wird. Typischerweise wird der Parameter `prefix` verwendet, um `cn=` oder in einer Active-Directory-Umgebung `DOMAIN\` anzugeben. `suffix` wird verwendet, um den verbleibenden Teil des DN in einer Nicht-Active-Directory-Umgebung anzugeben.

Im zweiten Modus, den wir Search+Bind-Modus nennen, bindet der Server zunächst mit einem festen Benutzernamen und Passwort an das LDAP-Verzeichnis, angegeben mit `ldapbinddn` und `ldapbindpasswd`, und führt eine Suche nach dem Benutzer durch, der sich an der Datenbank anmelden möchte. Wenn kein Benutzer und Passwort konfiguriert ist, wird ein anonymer Bind an das Verzeichnis versucht. Die Suche wird über den Teilbaum bei `ldapbasedn` ausgeführt und versucht, eine exakte Übereinstimmung mit dem in `ldapsearchattribute` angegebenen Attribut zu finden. Sobald der Benutzer in dieser Suche gefunden wurde, bindet der Server erneut als dieser Benutzer an das Verzeichnis, wobei er das vom Client angegebene Passwort verwendet, um zu prüfen, ob die Anmeldung korrekt ist. Dieser Modus entspricht dem, der von LDAP-Authentifizierungsschemata in anderer Software verwendet wird, etwa Apache `mod_authnz_ldap` und `pam_ldap`. Diese Methode erlaubt deutlich mehr Flexibilität dabei, wo sich Benutzerobjekte im Verzeichnis befinden, verursacht aber zwei zusätzliche Anfragen an den LDAP-Server.

<https://datatracker.ietf.org/doc/html/rfc1413>

Die folgenden Konfigurationsoptionen werden in beiden Modi verwendet:

`ldapserver`

Namen oder IP-Adressen der LDAP-Server, zu denen eine Verbindung hergestellt werden soll. Mehrere Server können durch Leerzeichen getrennt angegeben werden.

`ldapport`

Portnummer auf dem LDAP-Server, zu dem verbunden werden soll. Wenn kein Port angegeben ist, wird die Standardeinstellung der LDAP-Bibliothek für den Port verwendet.

`ldapscheme`

Auf `ldaps` setzen, um LDAPS zu verwenden. Dies ist eine nicht standardisierte Art, LDAP über SSL zu verwenden, die von einigen LDAP-Serverimplementierungen unterstützt wird. Siehe auch die Option `ldaptls` als Alternative.

`ldaptls`

Auf `1` setzen, damit die Verbindung zwischen PostgreSQL und dem LDAP-Server TLS-Verschlüsselung verwendet. Dies nutzt die StartTLS-Operation gemäß RFC 4513. Siehe auch die Option `ldapscheme` als Alternative.

Beachten Sie, dass die Verwendung von `ldapscheme` oder `ldaptls` nur den Verkehr zwischen dem PostgreSQL-Server und dem LDAP-Server verschlüsselt. Die Verbindung zwischen dem PostgreSQL-Server und dem PostgreSQL-Client bleibt unverschlüsselt, sofern dort nicht ebenfalls SSL verwendet wird.

Die folgenden Optionen werden nur im Simple-Bind-Modus verwendet:

`ldapprefix`

String, der dem Benutzernamen vorangestellt wird, wenn bei Simple-Bind-Authentifizierung der DN gebildet wird, an den gebunden werden soll.

`ldapsuffix`

String, der an den Benutzernamen angehängt wird, wenn bei Simple-Bind-Authentifizierung der DN gebildet wird, an den gebunden werden soll.

Die folgenden Optionen werden nur im Search+Bind-Modus verwendet:

`ldapbasedn`

Root-DN, bei dem die Suche nach dem Benutzer bei Search+Bind-Authentifizierung beginnen soll.

`ldapbinddn`

DN des Benutzers, mit dem zum Ausführen der Suche bei Search+Bind-Authentifizierung an das Verzeichnis gebunden werden soll.

`ldapbindpasswd`

Passwort des Benutzers, mit dem zum Ausführen der Suche bei Search+Bind-Authentifizierung an das Verzeichnis gebunden werden soll.

<https://datatracker.ietf.org/doc/html/rfc4513>

`ldapsearchattribute`

Attribut, das bei Search+Bind-Authentifizierung in der Suche gegen den Benutzernamen abgeglichen werden soll. Wenn kein Attribut angegeben ist, wird das Attribut `uid` verwendet.

`ldapsearchfilter`

Der Suchfilter, der bei Search+Bind-Authentifizierung verwendet werden soll. Vorkommen von `$username` werden durch den Benutzernamen ersetzt. Dies erlaubt flexiblere Suchfilter als `ldapsearchattribute`.

Die folgende Option kann als alternative Möglichkeit verwendet werden, einige der obigen LDAP-Optionen in kompakterer und standardisierter Form zu schreiben:

`ldapurl`

Eine LDAP-URL nach RFC 4516. Das Format ist:

```text
ldap[s]://host[:port]/basedn[?[attribute][?[scope][?[filter]]]]
```

`scope` muss einer von `base`, `one`, `sub` sein, typischerweise der letzte. (Der Standard ist `base`, was in dieser Anwendung normalerweise nicht nützlich ist.) `attribute` kann ein einzelnes Attribut benennen; in diesem Fall wird es als Wert für `ldapsearchattribute` verwendet. Wenn `attribute` leer ist, kann `filter` als Wert für `ldapsearchfilter` verwendet werden.

Das URL-Schema `ldaps` wählt die LDAPS-Methode für LDAP-Verbindungen über SSL, entsprechend der Verwendung von `ldapscheme=ldaps`. Um verschlüsselte LDAP-Verbindungen mit der StartTLS-Operation zu verwenden, verwenden Sie das normale URL-Schema `ldap` und geben zusätzlich zu `ldapurl` die Option `ldaptls` an.

Für nicht anonyme Binds müssen `ldapbinddn` und `ldapbindpasswd` als separate Optionen angegeben werden.

LDAP-URLs werden derzeit nur mit OpenLDAP unterstützt, nicht unter Windows.

Es ist ein Fehler, Konfigurationsoptionen für Simple Bind mit Optionen für Search+Bind zu mischen. Um `ldapurl` im Simple-Bind-Modus zu verwenden, darf die URL weder `basedn` noch Query-Elemente enthalten.

Bei Verwendung des Search+Bind-Modus kann die Suche mit einem einzelnen, durch `ldapsearchattribute` angegebenen Attribut oder mit einem benutzerdefinierten, durch `ldapsearchfilter` angegebenen Suchfilter durchgeführt werden. Die Angabe von `ldapsearchattribute=foo` entspricht der Angabe von `ldapsearchfilter="(foo=$username)"`. Wenn keine der beiden Optionen angegeben ist, ist der Standard `ldapsearchattribute=uid`.

Wenn PostgreSQL mit OpenLDAP als LDAP-Clientbibliothek kompiliert wurde, kann die Einstellung `ldapserver` weggelassen werden. In diesem Fall wird eine Liste von Hostnamen und Ports über DNS-SRV-Records gemäß RFC 2782 nachgeschlagen. Der Name `_ldap._tcp.DOMAIN` wird nachgeschlagen, wobei `DOMAIN` aus `ldapbasedn` extrahiert wird.

Hier ist ein Beispiel für eine Simple-Bind-LDAP-Konfiguration:

```text
host ... ldap ldapserver=ldap.example.net ldapprefix="cn="
 ldapsuffix=", dc=example, dc=net"
```

Wenn eine Verbindung zum Datenbankserver als Datenbankbenutzer `someuser` angefordert wird, versucht PostgreSQL, sich mit dem DN `cn=someuser, dc=example, dc=net` und dem vom Client bereitgestellten Passwort an den LDAP-Server zu binden. Wenn diese Verbindung erfolgreich ist, wird der Datenbankzugriff gewährt.

<https://datatracker.ietf.org/doc/html/rfc4516>
<https://datatracker.ietf.org/doc/html/rfc2782>

Hier ist eine andere Simple-Bind-Konfiguration, die das LDAPS-Schema und eine benutzerdefinierte Portnummer verwendet, als URL geschrieben:

```text
host ... ldap ldapurl="ldaps://ldap.example.net:49151"
 ldapprefix="cn=" ldapsuffix=", dc=example, dc=net"
```

Dies ist etwas kompakter, als `ldapserver`, `ldapscheme` und `ldapport` separat anzugeben.

Hier ist ein Beispiel für eine Search+Bind-Konfiguration:

```text
host ... ldap ldapserver=ldap.example.net ldapbasedn="dc=example,
 dc=net" ldapsearchattribute=uid
```

Wenn eine Verbindung zum Datenbankserver als Datenbankbenutzer `someuser` angefordert wird, versucht PostgreSQL, anonym an den LDAP-Server zu binden (da `ldapbinddn` nicht angegeben wurde), und führt eine Suche nach `(uid=someuser)` unter dem angegebenen Base-DN durch. Wenn ein Eintrag gefunden wird, versucht es anschließend, mit den gefundenen Informationen und dem vom Client bereitgestellten Passwort zu binden. Wenn dieser zweite Bind erfolgreich ist, wird der Datenbankzugriff gewährt.

Hier ist dieselbe Search+Bind-Konfiguration als URL geschrieben:

```text
host ... ldap ldapurl="ldap://ldap.example.net/dc=example,dc=net?uid?sub"
```

Einige andere Software, die Authentifizierung gegen LDAP unterstützt, verwendet dasselbe URL-Format, sodass die Konfiguration leichter gemeinsam genutzt werden kann.

Hier ist ein Beispiel für eine Search+Bind-Konfiguration, die `ldapsearchfilter` statt `ldapsearchattribute` verwendet, um Authentifizierung per Benutzer-ID oder E-Mail-Adresse zu erlauben:

```text
host ... ldap ldapserver=ldap.example.net ldapbasedn="dc=example,
 dc=net" ldapsearchfilter="(|(uid=$username)(mail=$username))"
```

Hier ist ein Beispiel für eine Search+Bind-Konfiguration, die DNS-SRV-Discovery verwendet, um Hostname(n) und Port(s) für den LDAP-Dienst des Domainnamens `example.net` zu finden:

```text
host ... ldap ldapbasedn="dc=example,dc=net"
```

> Tipp: Da LDAP häufig Kommas und Leerzeichen verwendet, um die verschiedenen Teile eines DN zu trennen, ist es bei der Konfiguration von LDAP-Optionen oft nötig, Parameterwerte in doppelte Anführungszeichen zu setzen, wie in den Beispielen gezeigt.

## 20.11. RADIUS-Authentifizierung

Diese Authentifizierungsmethode funktioniert ähnlich wie `password`, verwendet jedoch RADIUS als Methode zur Passwortprüfung. RADIUS wird nur verwendet, um Benutzername/Passwort-Paare zu validieren. Daher muss der Benutzer bereits in der Datenbank existieren, bevor RADIUS zur Authentifizierung verwendet werden kann.

Bei Verwendung von RADIUS-Authentifizierung wird eine Access-Request-Nachricht an den konfigurierten RADIUS-Server gesendet. Diese Anfrage ist vom Typ Authenticate Only und enthält Parameter für Benutzername, Passwort (verschlüsselt) und NAS Identifier. Die Anfrage wird mit einem mit dem Server geteilten Secret verschlüsselt. Der RADIUS-Server beantwortet diese Anfrage entweder mit Access Accept oder Access Reject. RADIUS-Accounting wird nicht unterstützt.

Mehrere RADIUS-Server können angegeben werden; in diesem Fall werden sie nacheinander ausprobiert. Wenn von einem Server eine negative Antwort empfangen wird, schlägt die Authentifizierung fehl. Wenn keine Antwort empfangen wird, wird der nächste Server in der Liste ausprobiert. Um mehrere Server anzugeben, trennen Sie die Servernamen durch Kommas und setzen die Liste in doppelte Anführungszeichen. Wenn mehrere Server angegeben sind, können die anderen RADIUS-Optionen ebenfalls als kommagetrennte Listen angegeben werden, um individuelle Werte für jeden Server bereitzustellen. Sie können auch als einzelner Wert angegeben werden; dann gilt dieser Wert für alle Server.

Die folgenden Konfigurationsoptionen werden für RADIUS unterstützt:

`radiusservers`

Die DNS-Namen oder IP-Adressen der RADIUS-Server, zu denen eine Verbindung hergestellt werden soll. Dieser Parameter ist erforderlich.

`radiussecrets`

Die Shared Secrets, die bei der sicheren Kommunikation mit den RADIUS-Servern verwendet werden. Dies muss auf PostgreSQL- und RADIUS-Servern exakt denselben Wert haben. Es wird empfohlen, hierfür einen String mit mindestens 16 Zeichen zu verwenden. Dieser Parameter ist erforderlich.

> Hinweis: Der verwendete Verschlüsselungsvektor ist nur dann kryptographisch stark, wenn PostgreSQL mit OpenSSL-Unterstützung gebaut wurde. In anderen Fällen sollte die Übertragung zum RADIUS-Server nur als verschleiert, nicht als gesichert betrachtet werden; bei Bedarf sollten externe Sicherheitsmaßnahmen angewendet werden.

`radiusports`

Die Portnummern, zu denen auf den RADIUS-Servern verbunden werden soll. Wenn kein Port angegeben ist, wird der Standard-RADIUS-Port (`1812`) verwendet.

`radiusidentifiers`

Die Strings, die als NAS Identifier in den RADIUS-Anfragen verwendet werden sollen. Dieser Parameter kann zum Beispiel verwendet werden, um zu identifizieren, zu welchem Datenbankcluster der Benutzer eine Verbindung herzustellen versucht; das kann für Policy-Matching auf dem RADIUS-Server nützlich sein. Wenn kein Identifier angegeben ist, wird standardmäßig `postgresql` verwendet.

Wenn in einem RADIUS-Parameterwert ein Komma oder Whitespace erforderlich ist, kann dies durch doppelte Anführungszeichen um den Wert erreicht werden; es ist jedoch mühsam, weil nun zwei Ebenen doppelter Quotierung nötig sind. Ein Beispiel für Whitespace in RADIUS-Secret-Strings:

```text
host ... radius radiusservers="server1,server2"
 radiussecrets="""secret one"",""secret two"""
```

## 20.12. Zertifikatsauthentifizierung

Diese Authentifizierungsmethode verwendet SSL-Client-Zertifikate zur Authentifizierung. Sie ist daher nur für SSL-Verbindungen verfügbar; Anweisungen zur SSL-Konfiguration finden Sie in Abschnitt 18.9.2. Bei Verwendung dieser Authentifizierungsmethode verlangt der Server, dass der Client ein gültiges, vertrauenswürdiges Zertifikat bereitstellt. Es wird keine Passwortabfrage an den Client gesendet. Das Attribut `cn` (Common Name) des Zertifikats wird mit dem angeforderten Datenbankbenutzernamen verglichen; wenn sie übereinstimmen, wird die Anmeldung erlaubt. Benutzername-Zuordnung kann verwendet werden, damit `cn` sich vom Datenbankbenutzernamen unterscheiden darf.

Die folgenden Konfigurationsoptionen werden für SSL-Zertifikatsauthentifizierung unterstützt:

`map`

Erlaubt die Zuordnung zwischen System- und Datenbankbenutzernamen. Details finden Sie in [Abschnitt 20.2](20_Client_Authentifizierung.md#202-benutzernamezuordnungen).

Es ist redundant, die Option `clientcert` mit `cert`-Authentifizierung zu verwenden, weil `cert`-Authentifizierung effektiv Trust-Authentifizierung mit `clientcert=verify-full` ist.

## 20.13. PAM-Authentifizierung

Diese Authentifizierungsmethode funktioniert ähnlich wie `password`, verwendet jedoch PAM (Pluggable Authentication Modules) als Authentifizierungsmechanismus. Der Standardname des PAM-Dienstes ist `postgresql`. PAM wird nur verwendet, um Benutzername/Passwort-Paare und optional den verbundenen Remote-Hostnamen oder die IP-Adresse zu validieren. Daher muss der Benutzer bereits in der Datenbank existieren, bevor PAM zur Authentifizierung verwendet werden kann. Weitere Informationen zu PAM finden Sie auf der Linux-PAM-Seite.

Die folgenden Konfigurationsoptionen werden für PAM unterstützt:

`pamservice`

PAM-Dienstname.

`pam_use_hostname`

Bestimmt, ob die Remote-IP-Adresse oder der Hostname über das Element `PAM_RHOST` an PAM-Module übergeben wird. Standardmäßig wird die IP-Adresse verwendet. Setzen Sie diese Option auf `1`, um stattdessen den aufgelösten Hostnamen zu verwenden. Hostnamenauflösung kann zu Anmeldeverzögerungen führen. (Die meisten PAM-Konfigurationen verwenden diese Information nicht, daher muss diese Einstellung nur berücksichtigt werden, wenn eine PAM-Konfiguration ausdrücklich erstellt wurde, um sie zu nutzen.)

> Hinweis: Wenn PAM so eingerichtet ist, dass es `/etc/shadow` liest, schlägt die Authentifizierung fehl, weil der PostgreSQL-Server von einem Nicht-Root-Benutzer gestartet wird. Dies ist jedoch kein Problem, wenn PAM so konfiguriert ist, dass LDAP oder andere Authentifizierungsmethoden verwendet werden.

## 20.14. BSD-Authentifizierung

Diese Authentifizierungsmethode funktioniert ähnlich wie `password`, verwendet jedoch BSD Authentication zur Passwortprüfung. BSD Authentication wird nur verwendet, um Benutzername/Passwort-Paare zu validieren. Daher muss die Rolle des Benutzers bereits in der Datenbank existieren, bevor BSD Authentication zur Authentifizierung verwendet werden kann. Das BSD-Authentication-Framework ist derzeit nur unter OpenBSD verfügbar.

BSD Authentication in PostgreSQL verwendet den Login-Typ `auth-postgresql` und authentifiziert mit der Login-Klasse `postgresql`, wenn diese in `login.conf` definiert ist. Standardmäßig existiert diese Login-Klasse nicht, und PostgreSQL verwendet die Standard-Login-Klasse.

> Hinweis: Um BSD Authentication zu verwenden, muss das PostgreSQL-Benutzerkonto (das heißt der Betriebssystembenutzer, der den Server ausführt) zuerst zur Gruppe `auth` hinzugefügt werden. Die Gruppe `auth` existiert auf OpenBSD-Systemen standardmäßig.

<https://www.kernel.org/pub/linux/libs/pam/>

## 20.15. OAuth-Autorisierung und -Authentifizierung

OAuth 2.0 ist ein Industriestandard-Framework, definiert in RFC 6749, das Drittanbieteranwendungen begrenzten Zugriff auf eine geschützte Ressource ermöglicht. OAuth-Client-Unterstützung muss beim Bau von PostgreSQL aktiviert werden; weitere Informationen finden Sie in [Kapitel 17](17_Installation_aus_dem_Quellcode.md).

Diese Dokumentation verwendet bei der Diskussion des OAuth-Ökosystems die folgende Terminologie:

`Resource Owner` (oder `End User`)

Der Benutzer oder das System, dem geschützte Ressourcen gehören und der beziehungsweise das Zugriff darauf gewähren kann. Diese Dokumentation verwendet auch den Begriff End User, wenn der Resource Owner eine Person ist. Wenn Sie `psql` verwenden, um sich mit OAuth mit der Datenbank zu verbinden, sind Sie der Resource Owner/End User.

`Client`

Das System, das mit Access Tokens auf die geschützten Ressourcen zugreift. Anwendungen, die libpq verwenden, etwa `psql`, sind die OAuth-Clients, wenn sie sich mit einem PostgreSQL-Cluster verbinden.

`Resource Server`

Das System, das die geschützten Ressourcen hostet, auf die der Client zugreift. Der PostgreSQL-Cluster, mit dem verbunden wird, ist der Resource Server.

`Provider`

Die Organisation, der Produktanbieter oder eine andere Entität, die die OAuth-Autorisierungsserver und -Clients für eine bestimmte Anwendung entwickelt und/oder administriert. Verschiedene Provider wählen typischerweise unterschiedliche Implementierungsdetails für ihre OAuth-Systeme; ein Client eines Providers hat im Allgemeinen nicht garantiert Zugriff auf die Server eines anderen.

Diese Verwendung des Begriffs „Provider“ ist nicht standardisiert, scheint aber umgangssprachlich weit verbreitet zu sein. (Er sollte nicht mit dem ähnlichen Begriff „Identity Provider“ von OpenID verwechselt werden. Obwohl die OAuth-Implementierung in PostgreSQL interoperabel und kompatibel mit OpenID Connect/OIDC sein soll, ist sie selbst kein OIDC-Client und erfordert dessen Verwendung nicht.)

`Authorization Server`

Das System, das Anfragen vom Client empfängt und Access Tokens an den Client ausgibt, nachdem der authentifizierte Resource Owner zugestimmt hat. PostgreSQL stellt keinen Authorization Server bereit; dies liegt in der Verantwortung des OAuth-Providers.

`Issuer`

Ein Bezeichner für einen Authorization Server, dargestellt als `https://`-URL, der einen vertrauenswürdigen „Namensraum“ für OAuth-Clients und -Anwendungen bereitstellt. Der Issuer-Bezeichner erlaubt einem einzelnen Authorization Server, mit Clients gegenseitig nicht vertrauender Entitäten zu sprechen, solange diese getrennte Issuer beibehalten.

> Hinweis: Bei kleinen Deployments gibt es möglicherweise keinen sinnvollen Unterschied zwischen „Provider“, „Authorization Server“ und „Issuer“. Bei komplexeren Setups kann es jedoch eine Eins-zu-viele- oder Viele-zu-viele-Beziehung geben: Ein Provider kann mehrere Issuer-Bezeichner an getrennte Mandanten vermieten und dann mehrere Authorization Server, möglicherweise mit unterschiedlichen unterstützten Funktionsmengen, bereitstellen, um mit deren Clients zu interagieren.

<https://datatracker.ietf.org/doc/html/rfc6749>

PostgreSQL unterstützt Bearer Tokens, definiert in RFC 6750. Das sind eine Art Access Token, die mit OAuth 2.0 verwendet wird, wobei das Token ein opaker String ist. Das Format des Access Tokens ist implementierungsspezifisch und wird von jedem Authorization Server gewählt.

Die folgenden Konfigurationsoptionen werden für OAuth unterstützt:

`issuer`

Eine HTTPS-URL, die entweder exakt der Issuer-Bezeichner des Authorization Servers ist, wie in dessen Discovery Document definiert, oder eine Well-known-URI, die direkt auf dieses Discovery Document zeigt. Dieser Parameter ist erforderlich.

Wenn ein OAuth-Client eine Verbindung zum Server herstellt, wird mit dem Issuer-Bezeichner eine URL für das Discovery Document konstruiert. Standardmäßig verwendet diese URL die Konventionen von OpenID Connect Discovery: Der Pfad `/.well-known/openid-configuration` wird an das Ende des Issuer-Bezeichners angehängt. Alternativ wird, wenn der Issuer ein Pfadsegment `/.well-known/` enthält, diese URL unverändert an den Client weitergegeben.

> Warnung: Der OAuth-Client in libpq verlangt, dass die `issuer`-Einstellung des Servers exakt dem Issuer-Bezeichner entspricht, der im Discovery Document bereitgestellt wird, und dieser muss wiederum mit der Einstellung `oauth_issuer` des Clients übereinstimmen. Abweichungen in Groß-/Kleinschreibung oder Formatierung sind nicht erlaubt.

`scope`

Eine durch Leerzeichen getrennte Liste der OAuth-Scopes, die der Server benötigt, um den Client zu autorisieren und den Benutzer zu authentifizieren. Geeignete Werte werden durch den Authorization Server und das verwendete OAuth-Validierungsmodul bestimmt (weitere Informationen zu Validatoren finden Sie in [Kapitel 50](50_OAuth_Validator_Module.md)). Dieser Parameter ist erforderlich.

`validator`

Die Bibliothek, die zum Validieren von Bearer Tokens verwendet werden soll. Wenn angegeben, muss der Name exakt einer der in `oauth_validator_libraries` aufgeführten Bibliotheken entsprechen. Dieser Parameter ist optional, sofern `oauth_validator_libraries` nicht mehr als eine Bibliothek enthält; in diesem Fall ist er erforderlich.

`map`

Erlaubt die Zuordnung zwischen OAuth-Identity-Provider- und Datenbankbenutzernamen. Details finden Sie in [Abschnitt 20.2](20_Client_Authentifizierung.md#202-benutzernamezuordnungen). Wenn keine Map angegeben ist, muss der dem Token zugeordnete Benutzername (wie durch den OAuth-Validator bestimmt) exakt dem angeforderten Rollennamen entsprechen. Dieser Parameter ist optional.

`delegate_ident_mapping`

Eine erweiterte Option, die nicht für den allgemeinen Gebrauch gedacht ist.

Wenn auf `1` gesetzt, wird die Standard-Benutzerzuordnung mit `pg_ident.conf` übersprungen, und der OAuth-Validator übernimmt die volle Verantwortung dafür, End-User-Identitäten Datenbankrollen zuzuordnen. Wenn der Validator das Token autorisiert, vertraut der Server darauf, dass der Benutzer sich unter der angeforderten Rolle verbinden darf, und die Verbindung darf unabhängig vom Authentifizierungsstatus des Benutzers fortgesetzt werden.

Dieser Parameter ist inkompatibel mit `map`.

> Warnung: `delegate_ident_mapping` bietet zusätzliche Flexibilität beim Entwurf des Authentifizierungssystems, erfordert aber auch eine sorgfältige Implementierung des OAuth-Validators, der zusätzlich zu den Standardprüfungen, die von allen Validatoren verlangt werden, bestimmen muss, ob das bereitgestellte Token ausreichende End-User-Privilegien trägt. Mit Vorsicht verwenden.

<https://datatracker.ietf.org/doc/html/rfc6750>

## 20.16. Authentifizierungsprobleme

Authentifizierungsfehler und verwandte Probleme zeigen sich im Allgemeinen durch Fehlermeldungen wie die folgenden:

```text
FATAL: no pg_hba.conf entry for host "123.123.123.123", user
 "andym", database "testdb"
```

Das ist die Meldung, die Sie am wahrscheinlichsten erhalten, wenn Sie den Server erfolgreich kontaktieren, dieser aber nicht mit Ihnen sprechen möchte. Wie die Meldung nahelegt, hat der Server die Verbindungsanfrage abgelehnt, weil er keinen passenden Eintrag in seiner Konfigurationsdatei `pg_hba.conf` gefunden hat.

```text
FATAL:      password authentication failed for user "andym"
```

Meldungen wie diese zeigen an, dass Sie den Server kontaktiert haben und er bereit ist, mit Ihnen zu sprechen, aber erst, nachdem Sie die in der Datei `pg_hba.conf` angegebene Authentifizierungsmethode bestanden haben. Prüfen Sie das angegebene Passwort, oder prüfen Sie Ihre Kerberos- oder Ident-Software, wenn die Meldung einen dieser Authentifizierungstypen erwähnt.

```text
FATAL:      user "andym" does not exist
```

Der angegebene Datenbankbenutzername wurde nicht gefunden.

```text
FATAL:      database "testdb" does not exist
```

Die Datenbank, zu der Sie eine Verbindung herstellen möchten, existiert nicht. Beachten Sie, dass, wenn Sie keinen Datenbanknamen angeben, standardmäßig der Datenbankbenutzername verwendet wird.

> Tipp: Das Serverlog kann mehr Informationen über einen Authentifizierungsfehler enthalten, als dem Client gemeldet wird. Wenn Sie über den Grund eines Fehlers verwirrt sind, prüfen Sie das Serverlog.
