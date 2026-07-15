# 50. OAuth-Validator-Module

PostgreSQL stellt Infrastruktur bereit, um eigene Module zu erstellen, die serverseitig OAuth-Bearer-Token validieren. Da OAuth-Implementierungen sehr unterschiedlich sind und die Validierung von Bearer-Token stark von der ausstellenden Stelle abhängt, kann der Server das Token nicht selbst prüfen; Validator-Module stellen die Integrationsschicht zwischen dem Server und dem verwendeten OAuth-Provider bereit.

OAuth-Validator-Module müssen mindestens aus einer Initialisierungsfunktion (siehe [Abschnitt 50.2](50_OAuth_Validator_Module.md#502-initialisierungsfunktionen)) und dem erforderlichen Callback zur Durchführung der Validierung bestehen (siehe [Abschnitt 50.3.2](50_OAuth_Validator_Module.md#5032-validatecallback)).

> Da ein fehlerhaft arbeitender Validator unbefugten Benutzern Zugang zur Datenbank verschaffen könnte, ist eine korrekte Implementierung für die Serversicherheit entscheidend. Entwurfsüberlegungen finden Sie in [Abschnitt 50.1](50_OAuth_Validator_Module.md#501-ein-validatormodul-sicher-entwerfen).

## 50.1. Ein Validatormodul sicher entwerfen

> Lesen und verstehen Sie diesen gesamten Abschnitt, bevor Sie ein Validatormodul implementieren. Ein fehlerhaft arbeitender Validator ist potenziell schlimmer als gar keine Authentifizierung, sowohl wegen des falschen Sicherheitsgefühls, das er vermittelt, als auch weil er zu Angriffen auf andere Teile eines OAuth-Ökosystems beitragen kann.

### 50.1.1. Verantwortlichkeiten des Validators

Obwohl verschiedene Module sehr unterschiedliche Ansätze zur Tokenvalidierung verfolgen können, müssen Implementierungen im Allgemeinen drei getrennte Aktionen ausführen:

- Das Token validieren

  Der Validator muss zunächst sicherstellen, dass das vorgelegte Token tatsächlich ein gültiges Bearer-Token zur Verwendung in der Clientauthentifizierung ist. Der richtige Weg dafür hängt vom Provider ab, umfasst aber im Allgemeinen entweder kryptographische Operationen, die beweisen, dass das Token von einer vertrauenswürdigen Stelle erstellt wurde (Offline-Validierung), oder die Vorlage des Tokens bei dieser vertrauenswürdigen Stelle, damit diese die Validierung übernimmt (Online-Validierung).

  Online-Validierung, üblicherweise über OAuth Token Introspection implementiert[^1], erfordert weniger Schritte in einem Validatormodul und erlaubt die zentrale Sperrung eines Tokens, falls es gestohlen oder fälschlich ausgestellt wurde. Sie erfordert jedoch, dass das Modul mindestens einen Netzwerkaufruf pro Authentifizierungsversuch ausführt (die alle innerhalb des konfigurierten `authentication_timeout` abgeschlossen sein müssen). Außerdem stellt Ihr Provider möglicherweise keine Introspection-Endpunkte zur Verwendung durch externe Resource Server bereit.

  Offline-Validierung ist deutlich aufwendiger. Typischerweise muss ein Validator eine Liste vertrauenswürdiger Signaturschlüssel für einen Provider verwalten und dann die kryptographische Signatur des Tokens zusammen mit dessen Inhalt prüfen. Implementierungen müssen die Anweisungen des Providers buchstabengetreu befolgen, einschließlich jeder Prüfung von Issuer ("woher stammt dieses Token?"), Audience ("für wen ist dieses Token bestimmt?") und Gültigkeitszeitraum ("wann kann dieses Token verwendet werden?"). Da es keine Kommunikation zwischen dem Modul und dem Provider gibt, können Token mit dieser Methode nicht zentral gesperrt werden; Offline-Validator-Implementierungen können erwägen, die maximale Länge der Gültigkeitsdauer eines Tokens zu beschränken.

  Wenn das Token nicht validiert werden kann, sollte das Modul sofort fehlschlagen. Weitere Authentifizierung oder Autorisierung ist sinnlos, wenn das Bearer-Token nicht von einer vertrauenswürdigen Stelle ausgestellt wurde.

- Den Client autorisieren

  Als Nächstes muss der Validator sicherstellen, dass der Endbenutzer dem Client erlaubt hat, in seinem Namen auf den Server zuzugreifen. Dazu werden im Allgemeinen die Scopes geprüft, die dem Token zugewiesen wurden, um sicherzustellen, dass sie den Datenbankzugriff für die aktuellen HBA-Parameter abdecken.

  Zweck dieses Schritts ist es, zu verhindern, dass ein OAuth-Client ein Token unter falschen Vorwänden erhält. Wenn der Validator verlangt, dass alle Token Scopes tragen, die den Datenbankzugriff abdecken, sollte der Provider den Benutzer im Ablauf deutlich auffordern, diesen Zugriff zu gewähren. Dadurch erhält er die Gelegenheit, die Anforderung abzulehnen, wenn der Client seine Anmeldedaten nicht zum Verbinden mit Datenbanken verwenden soll.

  Es ist zwar möglich, Clientautorisierung ohne explizite Scopes mithilfe außerhalb des Protokolls liegenden Wissens über die bereitgestellte Architektur herzustellen; dadurch wird der Benutzer jedoch aus der Schleife herausgenommen. Das verhindert, dass er Bereitstellungsfehler erkennt, und erlaubt, dass solche Fehler stillschweigend ausgenutzt werden. Der Zugriff auf die Datenbank muss streng auf vertrauenswürdige Clients beschränkt werden[^2], wenn Benutzer nicht zu zusätzlichen Scopes aufgefordert werden.

  Selbst wenn die Autorisierung fehlschlägt, kann ein Modul entscheiden, weiterhin Authentifizierungsinformationen aus dem Token für Auditing und Debugging zu entnehmen.

- Den Endbenutzer authentifizieren

  Schließlich sollte der Validator einen Benutzerbezeichner für das Token bestimmen, entweder indem er den Provider nach dieser Information fragt oder sie aus dem Token selbst extrahiert, und diesen Bezeichner an den Server zurückgeben. Der Server trifft anschließend mithilfe der HBA-Konfiguration eine abschließende Autorisierungsentscheidung. Dieser Bezeichner ist innerhalb der Sitzung über `system_user` verfügbar und wird in den Serverlogs aufgezeichnet, wenn `log_connections` aktiviert ist.

  Verschiedene Provider können eine Vielzahl unterschiedlicher Authentifizierungsinformationen für einen Endbenutzer aufzeichnen, die typischerweise als Claims bezeichnet werden. Provider dokumentieren normalerweise, welche dieser Claims vertrauenswürdig genug sind, um für Autorisierungsentscheidungen verwendet zu werden, und welche nicht. Beispielsweise wäre es wahrscheinlich nicht klug, den vollständigen Namen eines Endbenutzers als Bezeichner für die Authentifizierung zu verwenden, da viele Provider Benutzern erlauben, ihre Anzeigenamen beliebig zu ändern. Letztlich hängt die Wahl, welcher Claim (oder welche Kombination von Claims) verwendet wird, von der Provider-Implementierung und den Anwendungsanforderungen ab.

  Beachten Sie, dass auch anonyme oder pseudonyme Anmeldung möglich ist, indem Usermap-Delegation aktiviert wird; siehe [Abschnitt 50.1.3](50_OAuth_Validator_Module.md#5013-benutzer-autorisieren-usermapdelegation).

[^1]: <https://datatracker.ietf.org/doc/html/rfc7662>

[^2]: "Vertrauenswürdig" bedeutet hier, dass der OAuth-Client und der PostgreSQL-Server von derselben Stelle kontrolliert werden. Insbesondere erfüllt der von `libpq` unterstützte Device-Authorization-Client-Flow diese Anforderung normalerweise nicht, da er für die Verwendung durch öffentliche/nicht vertrauenswürdige Clients entworfen ist.

### 50.1.2. Allgemeine Coding-Richtlinien

Entwickler sollten bei der Implementierung von Tokenvalidierung Folgendes beachten:

- Vertraulichkeit von Token

  Module sollten Token oder Teile von Token nicht in das Serverlog schreiben. Dies gilt auch dann, wenn das Modul das Token für ungültig hält; ein Angreifer, der einen Client dazu bringt, mit dem falschen Provider zu kommunizieren, sollte dieses (ansonsten gültige) Token nicht von der Platte abrufen können.

  Implementierungen, die Token über das Netzwerk senden (zum Beispiel zur Durchführung einer Online-Tokenvalidierung bei einem Provider), müssen den Peer authentifizieren und sicherstellen, dass starke Transportsicherheit verwendet wird.

- Logging

  Module können dieselben Logging-Einrichtungen wie normale Erweiterungen verwenden; allerdings unterscheiden sich die Regeln für das Ausgeben von Logeinträgen an den Client während der Authentifizierungsphase der Verbindung in kleinen, aber wichtigen Details.

  Allgemein gesprochen sollten Module Verifikationsprobleme auf Ebene `COMMERROR` protokollieren und normal zurückkehren, statt mit `ERROR`/`FATAL` den Stack abzuwickeln, um keine Informationen an nicht authentifizierte Clients preiszugeben.

- Unterbrechbarkeit

  Module müssen durch Signale unterbrechbar bleiben, damit der Server Authentifizierungs-Timeouts und Shutdown-Signale von `pg_ctl` korrekt behandeln kann. Beispielsweise sollten blockierende Aufrufe auf Sockets im Allgemeinen durch Code ersetzt werden, der sowohl Socket-Ereignisse als auch Interrupts ohne Race Conditions behandelt (siehe `WaitLatchOrSocket()`, `WaitEventSetWait()` usw.), und lang laufende Schleifen sollten periodisch `CHECK_FOR_INTERRUPTS()` aufrufen. Die Nichtbeachtung dieser Hinweise kann zu nicht reagierenden Backend-Sitzungen führen.

- Tests

  Die volle Breite des Testens eines OAuth-Systems liegt weit außerhalb des Umfangs dieser Dokumentation, aber mindestens negative Tests sollten als verpflichtend betrachtet werden. Es ist trivial, ein Modul zu entwerfen, das autorisierte Benutzer hineinlässt; der ganze Zweck des Systems besteht darin, unbefugte Benutzer draußen zu halten.

- Dokumentation

  Validator-Implementierungen sollten den Inhalt und das Format der authentifizierten ID dokumentieren, die für jeden Endbenutzer an den Server gemeldet wird, da DBAs diese Information möglicherweise zum Erstellen von `pg_ident`-Maps benötigen. Handelt es sich zum Beispiel um eine E-Mail-Adresse, eine organisatorische ID-Nummer oder eine UUID? Sie sollten außerdem dokumentieren, ob es sicher ist, das Modul im Modus `delegate_ident_mapping=1` zu verwenden, und welche zusätzliche Konfiguration dafür erforderlich ist.

### 50.1.3. Benutzer autorisieren (Usermap-Delegation)

Das normale Ergebnis eines Validierungsmoduls ist der Benutzerbezeichner, den der Server anschließend mit allen konfigurierten Zuordnungen in `pg_ident.conf` vergleicht, um festzustellen, ob der Endbenutzer zum Verbinden autorisiert ist. OAuth ist jedoch selbst ein Autorisierungsframework, und Token können Informationen über Benutzerprivilegien enthalten. Beispielsweise kann ein Token mit den Organisationsgruppen verknüpft sein, zu denen ein Benutzer gehört, oder die Rollen auflisten, die ein Benutzer annehmen darf; dieses Wissen für jeden Server in lokale Usermaps zu duplizieren, ist möglicherweise nicht wünschenswert.

Um die Benutzernamenszuordnung vollständig zu umgehen und dem Validatormodul die zusätzliche Verantwortung zu übertragen, Benutzerverbindungen zu autorisieren, kann die HBA mit `delegate_ident_mapping` konfiguriert werden. Das Modul kann dann Token-Scopes oder eine gleichwertige Methode verwenden, um zu entscheiden, ob der Benutzer sich unter der gewünschten Rolle verbinden darf. Der Benutzerbezeichner wird weiterhin vom Server aufgezeichnet, spielt aber keine Rolle bei der Entscheidung, ob die Verbindung fortgesetzt wird.

Bei Verwendung dieses Schemas ist die Authentifizierung selbst optional. Solange das Modul meldet, dass die Verbindung autorisiert ist, wird die Anmeldung fortgesetzt, selbst wenn überhaupt kein Benutzerbezeichner aufgezeichnet wurde. Dadurch wird es möglich, anonymen oder pseudonymen Zugriff auf die Datenbank zu implementieren, bei dem der Drittanbieter-Provider die gesamte notwendige Authentifizierung durchführt, dem Server aber keine benutzeridentifizierenden Informationen bereitstellt. Manche Provider erzeugen stattdessen möglicherweise eine anonymisierte ID-Nummer, die für späteres Auditing aufgezeichnet werden kann.

Usermap-Delegation bietet die größte architektonische Flexibilität, macht das Validatormodul aber zu einem Single Point of Failure für die Verbindungsautorisierung. Verwenden Sie sie mit Vorsicht.

## 50.2. Initialisierungsfunktionen

OAuth-Validator-Module werden dynamisch aus den Shared Libraries geladen, die in `oauth_validator_libraries` aufgeführt sind. Module werden bei Bedarf geladen, wenn sie von einer laufenden Anmeldung angefordert werden. Zum Auffinden der Bibliothek wird der normale Bibliothekssuchpfad verwendet. Um die Validator-Callbacks bereitzustellen und anzuzeigen, dass die Bibliothek ein OAuth-Validator-Modul ist, muss eine Funktion mit dem Namen `_PG_oauth_validator_module_init` bereitgestellt werden. Der Rückgabewert der Funktion muss ein Zeiger auf eine Struktur vom Typ `OAuthValidatorCallbacks` sein, die eine Magic Number und Zeiger auf die Tokenvalidierungsfunktionen des Moduls enthält. Der zurückgegebene Zeiger muss Server-Lebensdauer haben; typischerweise wird dies erreicht, indem er als `static const`-Variable im globalen Gültigkeitsbereich definiert wird.

```c
typedef struct OAuthValidatorCallbacks
{
    uint32        magic;           /* must be set to
                                      PG_OAUTH_VALIDATOR_MAGIC */

    ValidatorStartupCB startup_cb;
    ValidatorShutdownCB shutdown_cb;
    ValidatorValidateCB validate_cb;
} OAuthValidatorCallbacks;

typedef const OAuthValidatorCallbacks *(*OAuthValidatorModuleInit) (void);
```

Nur der Callback `validate_cb` ist erforderlich; die anderen sind optional.

## 50.3. OAuth-Validator-Callbacks

OAuth-Validator-Module implementieren ihre Funktionalität, indem sie eine Menge von Callbacks definieren. Der Server ruft sie nach Bedarf auf, um die Authentifizierungsanforderung des Benutzers zu verarbeiten.

### 50.3.1. Startup-Callback

Der Callback `startup_cb` wird direkt nach dem Laden des Moduls ausgeführt. Dieser Callback kann verwendet werden, um lokalen Zustand einzurichten und bei Bedarf zusätzliche Initialisierung auszuführen. Wenn das Validatormodul Zustand hat, kann es `state->private_data` verwenden, um ihn zu speichern.

```c
typedef void (*ValidatorStartupCB) (ValidatorModuleState *state);
```

### 50.3.2. Validate-Callback

Der Callback `validate_cb` wird während des OAuth-Austauschs ausgeführt, wenn ein Benutzer versucht, sich mit OAuth zu authentifizieren. Jeder in vorherigen Aufrufen gesetzte Zustand ist in `state->private_data` verfügbar.

```c
typedef bool (*ValidatorValidateCB) (const ValidatorModuleState *state,
                                     const char *token,
                                     const char *role,
                                     ValidatorModuleResult *result);
```

`token` enthält das zu validierende Bearer-Token. PostgreSQL hat sichergestellt, dass das Token syntaktisch wohlgeformt ist, aber keine weitere Validierung durchgeführt. `role` enthält die Rolle, unter der sich der Benutzer anmelden möchte. Der Callback muss Ausgabeparameter in der Struktur `result` setzen, die wie folgt definiert ist:

```c
typedef struct ValidatorModuleResult
{
    bool        authorized;
    char       *authn_id;
} ValidatorModuleResult;
```

Die Verbindung wird nur fortgesetzt, wenn das Modul `result->authorized` auf `true` setzt. Um den Benutzer zu authentifizieren, muss der authentifizierte Benutzername (wie er anhand des Tokens bestimmt wurde) mit `palloc` allokiert und im Feld `result->authn_id` zurückgegeben werden. Alternativ kann `result->authn_id` auf `NULL` gesetzt werden, wenn das Token gültig ist, die zugehörige Benutzeridentität aber nicht bestimmt werden kann.

Ein Validator kann `false` zurückgeben, um einen internen Fehler zu signalisieren; in diesem Fall werden alle Ergebnisparameter ignoriert und die Verbindung schlägt fehl. Andernfalls sollte der Validator `true` zurückgeben, um anzuzeigen, dass er das Token verarbeitet und eine Autorisierungsentscheidung getroffen hat.

Das Verhalten nach der Rückkehr von `validate_cb` hängt von der konkreten HBA-Konfiguration ab. Normalerweise muss der Benutzername `result->authn_id` genau mit der Rolle übereinstimmen, unter der sich der Benutzer anmeldet. Dieses Verhalten kann mit einer Usermap geändert werden. Bei Authentifizierung gegen eine HBA-Regel mit aktiviertem `delegate_ident_mapping` führt PostgreSQL jedoch keinerlei Prüfungen des Werts von `result->authn_id` durch; in diesem Fall muss der Validator sicherstellen, dass das Token ausreichende Privilegien trägt, damit sich der Benutzer unter der angegebenen Rolle anmelden darf.

### 50.3.3. Shutdown-Callback

Der Callback `shutdown_cb` wird ausgeführt, wenn der mit der Verbindung verknüpfte Backend-Prozess beendet wird. Wenn das Validatormodul allokierten Zustand besitzt, sollte dieser Callback ihn freigeben, um Ressourcenlecks zu vermeiden.

```c
typedef void (*ValidatorShutdownCB) (ValidatorModuleState *state);
```
