# 21. Datenbankrollen

PostgreSQL verwaltet Datenbankzugriffsrechte mit dem Konzept von Rollen. Eine Rolle kann je nach Einrichtung entweder als Datenbankbenutzer oder als Gruppe von Datenbankbenutzern verstanden werden. Rollen können Datenbankobjekte besitzen (zum Beispiel Tabellen und Funktionen) und anderen Rollen Privilegien auf diesen Objekten zuweisen, um zu steuern, wer Zugriff auf welche Objekte hat. Außerdem ist es möglich, einer Rolle die Mitgliedschaft in einer anderen Rolle zu gewähren, sodass die Mitgliedsrolle Privilegien verwenden kann, die einer anderen Rolle zugewiesen wurden.

Das Rollenkonzept umfasst die Konzepte „Benutzer“ und „Gruppen“. In PostgreSQL-Versionen vor 8.1 waren Benutzer und Gruppen verschiedene Arten von Entitäten, aber heute gibt es nur noch Rollen. Jede Rolle kann als Benutzer, als Gruppe oder als beides dienen.

Dieses Kapitel beschreibt, wie Rollen erstellt und verwaltet werden. Weitere Informationen zu den Auswirkungen von Rollenprivilegien auf verschiedene Datenbankobjekte finden Sie in [Abschnitt 5.8](05_Datendefinition.md#58-privilegien).

## 21.1. Datenbankrollen

Datenbankrollen sind konzeptionell vollständig von Betriebssystembenutzern getrennt. In der Praxis kann es bequem sein, eine Entsprechung beizubehalten, erforderlich ist dies aber nicht. Datenbankrollen gelten global innerhalb einer Datenbankcluster-Installation (und nicht pro einzelner Datenbank). Um eine Rolle zu erstellen, verwenden Sie den SQL-Befehl `CREATE ROLE`:

```sql
CREATE ROLE name;
```

`name` folgt den Regeln für SQL-Bezeichner: entweder ungeschmückt ohne Sonderzeichen oder in doppelte Anführungszeichen gesetzt. (In der Praxis möchten Sie dem Befehl normalerweise zusätzliche Optionen hinzufügen, etwa `LOGIN`. Weitere Details erscheinen unten.) Um eine vorhandene Rolle zu entfernen, verwenden Sie den analogen Befehl `DROP ROLE`:

```sql
DROP ROLE name;
```

Zur Bequemlichkeit werden die Programme `createuser` und `dropuser` als Wrapper um diese SQL-Befehle bereitgestellt, die von der Shell-Kommandozeile aufgerufen werden können:

```text
createuser name
dropuser name
```

Um die Menge der vorhandenen Rollen zu bestimmen, untersuchen Sie den Systemkatalog `pg_roles`, zum Beispiel:

```sql
SELECT rolname FROM pg_roles;
```

Oder, um nur diejenigen zu sehen, die sich anmelden können:

```sql
SELECT rolname FROM pg_roles WHERE rolcanlogin;
```

Auch der `psql`-Metabefehl `\du` ist nützlich, um die vorhandenen Rollen aufzulisten.

Um das Datenbanksystem zu bootstrappen, enthält ein frisch initialisiertes System immer eine vordefinierte Rolle, die sich anmelden kann. Diese Rolle ist immer ein „Superuser“ und hat denselben Namen wie der Betriebssystembenutzer, der den Datenbankcluster mit `initdb` initialisiert hat, sofern kein anderer Name angegeben wurde. Diese Rolle heißt häufig `postgres`. Um weitere Rollen zu erstellen, müssen Sie sich zuerst als diese Anfangsrolle verbinden.

Jede Verbindung zum Datenbankserver wird mit dem Namen einer bestimmten Rolle hergestellt, und diese Rolle bestimmt die anfänglichen Zugriffsprivilegien für Befehle, die in dieser Verbindung ausgeführt werden. Der Rollenname, der für eine bestimmte Datenbankverbindung verwendet werden soll, wird vom Client, der die Verbindungsanfrage initiiert, auf anwendungsspezifische Weise angegeben. Beispielsweise verwendet das Programm `psql` die Kommandozeilenoption `-U`, um die Rolle anzugeben, als die verbunden werden soll. Viele Anwendungen nehmen standardmäßig den Namen des aktuellen Betriebssystembenutzers an (einschließlich `createuser` und `psql`). Daher ist es oft bequem, eine Namensentsprechung zwischen Rollen und Betriebssystembenutzern beizubehalten.

Die Menge der Datenbankrollen, als die sich eine gegebene Client-Verbindung verbinden kann, wird durch die Client-Authentifizierungseinrichtung bestimmt, wie in [Kapitel 20](20_Client_Authentifizierung.md) erläutert. (Ein Client ist also nicht darauf beschränkt, sich als die Rolle zu verbinden, die seinem Betriebssystembenutzer entspricht, genauso wie der Anmeldename einer Person nicht ihrem realen Namen entsprechen muss.) Da die Rollenidentität die Menge der Privilegien bestimmt, die einem verbundenen Client zur Verfügung stehen, ist es wichtig, Privilegien beim Einrichten einer Mehrbenutzerumgebung sorgfältig zu konfigurieren.

## 21.2. Rollenattribute

Eine Datenbankrolle kann eine Reihe von Attributen haben, die ihre Privilegien definieren und mit dem Client-Authentifizierungssystem interagieren.

`login`-Privileg

Nur Rollen mit dem Attribut `LOGIN` können als anfänglicher Rollenname für eine Datenbankverbindung verwendet werden. Eine Rolle mit dem Attribut `LOGIN` kann als gleichbedeutend mit einem „Datenbankbenutzer“ betrachtet werden. Um eine Rolle mit Login-Privileg zu erstellen, verwenden Sie entweder:

```sql
CREATE ROLE name LOGIN;
CREATE USER name;
```

(`CREATE USER` ist äquivalent zu `CREATE ROLE`, außer dass `CREATE USER` standardmäßig `LOGIN` einschließt, während `CREATE ROLE` dies nicht tut.)

Superuser-Status

Ein Datenbank-Superuser umgeht alle Berechtigungsprüfungen, mit Ausnahme des Rechts, sich anzumelden. Dies ist ein gefährliches Privileg und sollte nicht leichtfertig verwendet werden; es ist am besten, den Großteil der Arbeit als Rolle auszuführen, die kein Superuser ist. Um einen neuen Datenbank-Superuser zu erstellen, verwenden Sie `CREATE ROLE name SUPERUSER`. Sie müssen dies als Rolle ausführen, die bereits Superuser ist.

Datenbankerstellung

Einer Rolle muss ausdrücklich die Berechtigung gegeben werden, Datenbanken zu erstellen (außer Superusern, da diese alle Berechtigungsprüfungen umgehen). Um eine solche Rolle zu erstellen, verwenden Sie `CREATE ROLE name CREATEDB`.

Rollenerstellung

Einer Rolle muss ausdrücklich die Berechtigung gegeben werden, weitere Rollen zu erstellen (außer Superusern, da diese alle Berechtigungsprüfungen umgehen). Um eine solche Rolle zu erstellen, verwenden Sie `CREATE ROLE name CREATEROLE`. Eine Rolle mit dem Privileg `CREATEROLE` kann Rollen ändern und löschen, die dem `CREATEROLE`-Benutzer mit der Option `ADMIN` gewährt wurden. Eine solche Gewährung erfolgt automatisch, wenn ein `CREATEROLE`-Benutzer, der kein Superuser ist, eine neue Rolle erstellt, sodass ein `CREATEROLE`-Benutzer standardmäßig die von ihm erstellten Rollen ändern und löschen kann. Das Ändern einer Rolle umfasst die meisten Änderungen, die mit `ALTER ROLE` vorgenommen werden können, zum Beispiel das Ändern von Passwörtern. Es umfasst auch Änderungen an einer Rolle, die mit den Befehlen `COMMENT` und `SECURITY LABEL` vorgenommen werden können.

`CREATEROLE` vermittelt jedoch weder die Fähigkeit, `SUPERUSER`-Rollen zu erstellen, noch irgendeine Macht über bereits existierende `SUPERUSER`-Rollen. Außerdem vermittelt `CREATEROLE` nicht die Fähigkeit, `REPLICATION`-Benutzer zu erstellen, das Privileg `REPLICATION` zu gewähren oder zu entziehen oder die Rolleneigenschaften solcher Benutzer zu ändern. Es erlaubt jedoch, `ALTER ROLE ... SET` und `ALTER ROLE ... RENAME` auf `REPLICATION`-Rollen zu verwenden, ebenso `COMMENT ON ROLE`, `SECURITY LABEL ON ROLE` und `DROP ROLE`. Schließlich verleiht `CREATEROLE` nicht die Fähigkeit, das Privileg `BYPASSRLS` zu gewähren oder zu entziehen.

Replikation initiieren

Einer Rolle muss ausdrücklich die Berechtigung gegeben werden, Streaming-Replikation zu initiieren (außer Superusern, da diese alle Berechtigungsprüfungen umgehen). Eine für Streaming-Replikation verwendete Rolle muss außerdem die Berechtigung `LOGIN` haben. Um eine solche Rolle zu erstellen, verwenden Sie `CREATE ROLE name REPLICATION LOGIN`.

Passwort

Ein Passwort ist nur relevant, wenn die Client-Authentifizierungsmethode verlangt, dass der Benutzer beim Verbinden mit der Datenbank ein Passwort angibt. Die Authentifizierungsmethoden `password` und `md5` verwenden Passwörter. Datenbankpasswörter sind von Betriebssystempasswörtern getrennt. Geben Sie ein Passwort bei der Rollenerstellung mit `CREATE ROLE name PASSWORD 'string'` an.

Vererbung von Privilegien

Eine Rolle erbt standardmäßig die Privilegien der Rollen, deren Mitglied sie ist. Um jedoch eine Rolle zu erstellen, die Privilegien standardmäßig nicht erbt, verwenden Sie `CREATE ROLE name NOINHERIT`. Alternativ kann die Vererbung für einzelne Gewährungen mit `WITH INHERIT TRUE` oder `WITH INHERIT FALSE` überschrieben werden.

Umgehen von Row-Level Security

Einer Rolle muss ausdrücklich die Berechtigung gegeben werden, jede Row-Level-Security-Policy (RLS) zu umgehen (außer Superusern, da diese alle Berechtigungsprüfungen umgehen). Um eine solche Rolle zu erstellen, verwenden Sie als Superuser `CREATE ROLE name BYPASSRLS`.

Verbindungslimit

Das Verbindungslimit kann angeben, wie viele gleichzeitige Verbindungen eine Rolle herstellen darf. `-1` (der Standard) bedeutet kein Limit. Geben Sie das Verbindungslimit bei der Rollenerstellung mit `CREATE ROLE name CONNECTION LIMIT 'integer'` an.

Die Attribute einer Rolle können nach der Erstellung mit `ALTER ROLE` geändert werden. Details finden Sie auf den Referenzseiten zu den Befehlen `CREATE ROLE` und `ALTER ROLE`.

Eine Rolle kann außerdem rollenspezifische Standardwerte für viele der in [Kapitel 19](19_Serverkonfiguration.md) beschriebenen Laufzeit-Konfigurationseinstellungen haben. Wenn Sie zum Beispiel aus irgendeinem Grund Index-Scans bei jeder Verbindung deaktivieren möchten (Hinweis: keine gute Idee), können Sie verwenden:

```sql
ALTER ROLE myname SET enable_indexscan TO off;
```

Dies speichert die Einstellung (setzt sie aber nicht sofort). In nachfolgenden Verbindungen dieser Rolle erscheint es, als sei `SET enable_indexscan TO off` unmittelbar vor Sitzungsbeginn ausgeführt worden. Sie können diese Einstellung während der Sitzung weiterhin ändern; sie ist nur der Standard. Um eine rollenspezifische Standardeinstellung zu entfernen, verwenden Sie `ALTER ROLE rolename RESET varname`. Beachten Sie, dass rollenspezifische Standards, die an Rollen ohne `LOGIN`-Privileg hängen, ziemlich nutzlos sind, da sie nie aufgerufen werden.

Wenn ein Nicht-Superuser mit dem Privileg `CREATEROLE` eine Rolle erstellt, wird die erstellte Rolle automatisch dem erstellenden Benutzer zurückgewährt, so als hätte der Bootstrap-Superuser den Befehl `GRANT created_user TO creating_user WITH ADMIN TRUE, SET FALSE, INHERIT FALSE` ausgeführt. Da ein `CREATEROLE`-Benutzer Sonderprivilegien in Bezug auf eine vorhandene Rolle nur ausüben kann, wenn er darauf `ADMIN OPTION` hat, reicht diese Gewährung gerade aus, um einem `CREATEROLE`-Benutzer die Verwaltung der von ihm erstellten Rollen zu erlauben. Da sie jedoch mit `INHERIT FALSE, SET FALSE` erstellt wird, erbt der `CREATEROLE`-Benutzer weder die Privilegien der erstellten Rolle, noch kann er mit `SET ROLE` auf die Privilegien dieser Rolle zugreifen. Da jedoch jeder Benutzer, der `ADMIN OPTION` auf einer Rolle hat, die Mitgliedschaft in dieser Rolle jedem anderen Benutzer gewähren kann, kann der `CREATEROLE`-Benutzer Zugriff auf die erstellte Rolle erlangen, indem er diese Rolle einfach mit den Optionen `INHERIT` und/oder `SET` an sich selbst zurückgewährt. Dass Privilegien standardmäßig nicht vererbt werden und `SET ROLE` standardmäßig nicht gewährt wird, ist daher ein Schutz gegen Versehen, keine Sicherheitsfunktion. Beachten Sie außerdem: Weil diese automatische Gewährung durch den Bootstrap-Superuser erfolgt, kann sie vom `CREATEROLE`-Benutzer nicht entfernt oder geändert werden; jeder Superuser könnte sie jedoch widerrufen, ändern und/oder zusätzliche solche Gewährungen an andere `CREATEROLE`-Benutzer ausgeben. Alle `CREATEROLE`-Benutzer, die zu einem gegebenen Zeitpunkt `ADMIN OPTION` auf einer Rolle haben, können sie verwalten.

## 21.3. Rollenmitgliedschaft

Es ist häufig praktisch, Benutzer zusammenzufassen, um die Verwaltung von Privilegien zu erleichtern: So können Privilegien einer Gruppe als Ganzes gewährt oder entzogen werden. In PostgreSQL geschieht dies, indem eine Rolle erstellt wird, die die Gruppe repräsentiert, und anschließend einzelnen Benutzerrollen die Mitgliedschaft in der Gruppenrolle gewährt wird.

Um eine Gruppenrolle einzurichten, erstellen Sie zuerst die Rolle:

```sql
CREATE ROLE name;
```

Typischerweise hätte eine Rolle, die als Gruppe verwendet wird, nicht das Attribut `LOGIN`, obwohl Sie es setzen können, wenn Sie möchten.

Sobald die Gruppenrolle existiert, können Sie mit den Befehlen `GRANT` und `REVOKE` Mitglieder hinzufügen und entfernen:

```sql
GRANT group_role TO role1, ... ;
REVOKE group_role FROM role1, ... ;
```

Sie können Mitgliedschaft auch anderen Gruppenrollen gewähren (da es eigentlich keinen Unterschied zwischen Gruppenrollen und Nicht-Gruppenrollen gibt). Die Datenbank erlaubt nicht, zirkuläre Mitgliedschaftsschleifen einzurichten. Außerdem ist es nicht erlaubt, `PUBLIC` die Mitgliedschaft in einer Rolle zu gewähren.

Die Mitglieder einer Gruppenrolle können die Privilegien der Rolle auf zwei Arten verwenden. Erstens können Mitgliedsrollen, denen Mitgliedschaft mit der Option `SET` gewährt wurde, `SET ROLE` ausführen, um vorübergehend zur Gruppenrolle zu „werden“. In diesem Zustand hat die Datenbanksitzung Zugriff auf die Privilegien der Gruppenrolle statt auf die der ursprünglichen Login-Rolle, und alle erstellten Datenbankobjekte gelten als von der Gruppenrolle besessen, nicht von der Login-Rolle. Zweitens haben Mitgliedsrollen, denen Mitgliedschaft mit der Option `INHERIT` gewährt wurde, automatisch die Privilegien der Rollen, deren direktes oder indirektes Mitglied sie sind; die Kette endet jedoch bei Mitgliedschaften ohne inherit-Option. Nehmen wir als Beispiel an, wir hätten Folgendes ausgeführt:

```sql
CREATE ROLE joe LOGIN;
CREATE ROLE admin;
CREATE ROLE wheel;
CREATE ROLE island;
GRANT admin TO joe WITH INHERIT TRUE;
GRANT wheel TO admin WITH INHERIT FALSE;
GRANT island TO joe WITH INHERIT TRUE, SET FALSE;
```

Unmittelbar nach dem Verbinden als Rolle `joe` kann eine Datenbanksitzung die direkt `joe` gewährten Privilegien sowie alle `admin` und `island` gewährten Privilegien verwenden, weil `joe` diese Privilegien „erbt“. Privilegien, die `wheel` gewährt wurden, sind jedoch nicht verfügbar, denn obwohl `joe` indirekt Mitglied von `wheel` ist, verläuft die Mitgliedschaft über `admin`, dem sie mit `WITH INHERIT FALSE` gewährt wurde. Nach:

```sql
SET ROLE admin;
```

hätte die Sitzung nur die `admin` gewährten Privilegien, nicht die `joe` oder `island` gewährten. Nach:

```sql
SET ROLE wheel;
```

hätte die Sitzung nur die `wheel` gewährten Privilegien, nicht die `joe` oder `admin` gewährten. Der ursprüngliche Privilegzustand kann mit einem der folgenden Befehle wiederhergestellt werden:

```sql
SET ROLE joe;
SET ROLE NONE;
RESET ROLE;
```

> Hinweis: Der Befehl `SET ROLE` erlaubt immer die Auswahl jeder Rolle, deren direktes oder indirektes Mitglied die ursprüngliche Login-Rolle ist, sofern eine Kette von Mitgliedschaftsgewährungen existiert, von denen jede `SET TRUE` hat (was der Standard ist). Im obigen Beispiel ist es daher nicht nötig, zuerst `admin` zu werden, bevor man `wheel` wird. Andererseits ist es überhaupt nicht möglich, `island` zu werden; `joe` kann auf diese Privilegien nur über Vererbung zugreifen.

> Hinweis: Im SQL-Standard gibt es eine klare Unterscheidung zwischen Benutzern und Rollen, und Benutzer erben Privilegien nicht automatisch, während Rollen dies tun. Dieses Verhalten kann in PostgreSQL erreicht werden, indem Rollen, die als SQL-Rollen verwendet werden, das Attribut `INHERIT` erhalten, während Rollen, die als SQL-Benutzer verwendet werden, das Attribut `NOINHERIT` erhalten. PostgreSQL setzt jedoch aus Gründen der Abwärtskompatibilität mit Versionen vor 8.1, in denen Benutzer immer die Berechtigungen verwenden konnten, die Gruppen gewährt wurden, deren Mitglieder sie waren, standardmäßig für alle Rollen das Attribut `INHERIT`.

Die Rollenattribute `LOGIN`, `SUPERUSER`, `CREATEDB` und `CREATEROLE` können als besondere Privilegien verstanden werden, werden aber niemals so vererbt wie gewöhnliche Privilegien auf Datenbankobjekten. Sie müssen tatsächlich `SET ROLE` auf eine bestimmte Rolle ausführen, die eines dieser Attribute besitzt, um das Attribut zu nutzen. Wenn wir das obige Beispiel fortsetzen, könnten wir wählen, `CREATEDB` und `CREATEROLE` der Rolle `admin` zu gewähren. Dann hätte eine Sitzung, die sich als Rolle `joe` verbindet, diese Privilegien nicht sofort, sondern erst nach `SET ROLE admin`.

Um eine Gruppenrolle zu zerstören, verwenden Sie `DROP ROLE`:

```sql
DROP ROLE name;
```

Alle Mitgliedschaften in der Gruppenrolle werden automatisch widerrufen (die Mitgliedsrollen sind darüber hinaus aber nicht betroffen).

## 21.4. Rollen löschen

Da Rollen Datenbankobjekte besitzen und Privilegien zum Zugriff auf andere Objekte halten können, ist das Löschen einer Rolle häufig nicht einfach nur eine Sache eines schnellen `DROP ROLE`. Alle Objekte, die der Rolle gehören, müssen zuerst gelöscht oder anderen Eigentümern zugewiesen werden; und alle der Rolle gewährten Berechtigungen müssen widerrufen werden.

Die Eigentümerschaft an Objekten kann einzeln mit `ALTER`-Befehlen übertragen werden, zum Beispiel:

```sql
ALTER TABLE bobs_table OWNER TO alice;
```

Alternativ kann der Befehl `REASSIGN OWNED` verwendet werden, um die Eigentümerschaft aller Objekte, die der zu löschenden Rolle gehören, einer einzelnen anderen Rolle zuzuweisen. Da `REASSIGN OWNED` nicht auf Objekte in anderen Datenbanken zugreifen kann, muss der Befehl in jeder Datenbank ausgeführt werden, die Objekte enthält, die der Rolle gehören. (Beachten Sie, dass das erste solche `REASSIGN OWNED` die Eigentümerschaft aller datenbankübergreifend gemeinsam genutzten Objekte ändert, also Datenbanken oder Tablespaces, die der zu löschenden Rolle gehören.)

Sobald alle wertvollen Objekte auf neue Eigentümer übertragen wurden, können alle verbleibenden Objekte, die der zu löschenden Rolle gehören, mit dem Befehl `DROP OWNED` gelöscht werden. Auch dieser Befehl kann nicht auf Objekte in anderen Datenbanken zugreifen, daher muss er in jeder Datenbank ausgeführt werden, die Objekte enthält, die der Rolle gehören. Außerdem löscht `DROP OWNED` keine ganzen Datenbanken oder Tablespaces; wenn die Rolle Datenbanken oder Tablespaces besitzt, die nicht auf neue Eigentümer übertragen wurden, muss dies daher manuell erfolgen.

`DROP OWNED` kümmert sich außerdem darum, alle Privilegien zu entfernen, die der Zielrolle für Objekte gewährt wurden, die ihr nicht gehören. Da `REASSIGN OWNED` solche Objekte nicht berührt, ist es typischerweise notwendig, sowohl `REASSIGN OWNED` als auch `DROP OWNED` (in dieser Reihenfolge!) auszuführen, um die Abhängigkeiten einer zu löschenden Rolle vollständig zu entfernen.

Kurz gesagt lautet das allgemeinste Rezept zum Entfernen einer Rolle, die zum Besitz von Objekten verwendet wurde:

```sql
REASSIGN OWNED BY doomed_role TO successor_role;
DROP OWNED BY doomed_role;
-- repeat the above commands in each database of the cluster
DROP ROLE doomed_role;
```

Wenn nicht alle besessenen Objekte an denselben Nachfolgeeigentümer übertragen werden sollen, ist es am besten, die Ausnahmen manuell zu behandeln und anschließend die obigen Schritte zum Aufräumen auszuführen.

Wenn `DROP ROLE` versucht wird, während noch abhängige Objekte vorhanden sind, gibt es Meldungen aus, die angeben, welche Objekte neu zugewiesen oder gelöscht werden müssen.

## 21.5. Vordefinierte Rollen

PostgreSQL stellt eine Menge vordefinierter Rollen bereit, die Zugriff auf bestimmte, häufig benötigte privilegierte Fähigkeiten und Informationen ermöglichen. Administratoren (einschließlich Rollen mit dem Privileg `CREATEROLE`) können diese Rollen Benutzern und/oder anderen Rollen in ihrer Umgebung mit `GRANT` gewähren und diesen Benutzern damit Zugriff auf die angegebenen Fähigkeiten und Informationen geben. Zum Beispiel:

```sql
GRANT pg_signal_backend TO admin_user;
```

> Warnung: Beim Gewähren dieser Rollen ist Vorsicht geboten, um sicherzustellen, dass sie nur dort verwendet werden, wo sie benötigt werden, und im Bewusstsein, dass diese Rollen Zugriff auf privilegierte Informationen gewähren.

Die vordefinierten Rollen werden unten beschrieben. Beachten Sie, dass sich die konkreten Berechtigungen jeder dieser Rollen in Zukunft ändern können, wenn zusätzliche Fähigkeiten hinzugefügt werden. Administratoren sollten die Release Notes auf Änderungen beobachten.

`pg_checkpoint`

`pg_checkpoint` erlaubt das Ausführen des Befehls `CHECKPOINT`.

`pg_create_subscription`

`pg_create_subscription` erlaubt Benutzern mit der Berechtigung `CREATE` auf der Datenbank, `CREATE SUBSCRIPTION` auszuführen.

`pg_database_owner`

`pg_database_owner` hat immer genau ein implizites Mitglied: den aktuellen Datenbankeigentümer. Ihr kann keine Mitgliedschaft in irgendeiner Rolle gewährt werden, und keiner Rolle kann Mitgliedschaft in `pg_database_owner` gewährt werden. Wie jede andere Rolle kann sie jedoch Objekte besitzen und Zugriffsprivilegien erhalten. Sobald `pg_database_owner` innerhalb einer Template-Datenbank Rechte besitzt, besitzt folglich jeder Eigentümer einer aus diesem Template instanziierten Datenbank diese Rechte. Anfangs besitzt diese Rolle das Schema `public`, sodass jeder Datenbankeigentümer die lokale Nutzung dieses Schemas steuert.

`pg_maintain`

`pg_maintain` erlaubt das Ausführen von `VACUUM`, `ANALYZE`, `CLUSTER`, `REFRESH MATERIALIZED VIEW`, `REINDEX` und `LOCK TABLE` auf allen Relationen, so als hätte die Rolle `MAINTAIN`-Rechte auf diesen Objekten.

`pg_monitor`

`pg_read_all_settings`

`pg_read_all_stats`

`pg_stat_scan_tables`

Diese Rollen sollen Administratoren erlauben, leicht eine Rolle zum Monitoring des Datenbankservers zu konfigurieren. Sie gewähren eine Menge gängiger Privilegien, die der Rolle erlauben, verschiedene nützliche Konfigurationseinstellungen, Statistiken und andere Systeminformationen zu lesen, die normalerweise auf Superuser beschränkt sind.

`pg_monitor` erlaubt das Lesen/Ausführen verschiedener Monitoring-Views und -Funktionen. Diese Rolle ist Mitglied von `pg_read_all_settings`, `pg_read_all_stats` und `pg_stat_scan_tables`.

`pg_read_all_settings` erlaubt das Lesen aller Konfigurationsvariablen, selbst solcher, die normalerweise nur für Superuser sichtbar sind.

`pg_read_all_stats` erlaubt das Lesen aller `pg_stat_*`-Views und die Verwendung verschiedener statistikbezogener Extensions, selbst solcher, die normalerweise nur für Superuser sichtbar sind.

`pg_stat_scan_tables` erlaubt das Ausführen von Monitoring-Funktionen, die möglicherweise `ACCESS SHARE`-Sperren auf Tabellen nehmen, potenziell für längere Zeit (zum Beispiel `pgrowlocks(text)` in der Extension `pgrowlocks`).

`pg_read_all_data`

`pg_write_all_data`

`pg_read_all_data` erlaubt das Lesen aller Daten (Tabellen, Views, Sequenzen), so als hätte die Rolle `SELECT`-Rechte auf diesen Objekten und `USAGE`-Rechte auf allen Schemata. Diese Rolle umgeht keine Row-Level-Security-Policies (RLS). Wenn RLS verwendet wird, möchte ein Administrator möglicherweise `BYPASSRLS` auf Rollen setzen, denen diese Rolle gewährt wird.

`pg_write_all_data` erlaubt das Schreiben aller Daten (Tabellen, Views, Sequenzen), so als hätte die Rolle `INSERT`-, `UPDATE`- und `DELETE`-Rechte auf diesen Objekten und `USAGE`-Rechte auf allen Schemata. Diese Rolle umgeht keine Row-Level-Security-Policies (RLS). Wenn RLS verwendet wird, möchte ein Administrator möglicherweise `BYPASSRLS` auf Rollen setzen, denen diese Rolle gewährt wird.

`pg_read_server_files`

`pg_write_server_files`

`pg_execute_server_program`

Diese Rollen sollen Administratoren erlauben, vertrauenswürdige Nicht-Superuser-Rollen zu haben, die auf Dateien zugreifen und Programme auf dem Datenbankserver als der Benutzer ausführen können, unter dem die Datenbank läuft. Sie umgehen alle Berechtigungsprüfungen auf Datenbankebene beim direkten Zugriff auf Dateien und könnten verwendet werden, um Zugriff auf Superuser-Niveau zu erlangen. Daher ist beim Gewähren dieser Rollen an Benutzer große Vorsicht geboten.

`pg_read_server_files` erlaubt das Lesen von Dateien von jedem Ort auf dem Server, auf den die Datenbank zugreifen kann, mithilfe von `COPY` und anderen Dateizugriffsfunktionen.

`pg_write_server_files` erlaubt das Schreiben in Dateien an jedem Ort auf dem Server, auf den die Datenbank zugreifen kann, mithilfe von `COPY` und anderen Dateizugriffsfunktionen.

`pg_execute_server_program` erlaubt das Ausführen von Programmen auf dem Datenbankserver als der Benutzer, unter dem die Datenbank läuft, mithilfe von `COPY` und anderen Funktionen, die das Ausführen eines serverseitigen Programms erlauben.

`pg_signal_autovacuum_worker`

`pg_signal_autovacuum_worker` erlaubt das Signalisieren von Autovacuum-Workern, um das Vacuum der aktuellen Tabelle abzubrechen oder ihre Sitzung zu beenden. Siehe Abschnitt 9.28.2.

`pg_signal_backend`

`pg_signal_backend` erlaubt das Signalisieren eines anderen Backends, um eine Abfrage abzubrechen oder dessen Sitzung zu beenden. Beachten Sie, dass diese Rolle nicht erlaubt, Backends zu signalisieren, die einem Superuser gehören. Siehe Abschnitt 9.28.2.

`pg_use_reserved_connections`

`pg_use_reserved_connections` erlaubt die Verwendung von Verbindungs-Slots, die über `reserved_connections` reserviert sind.

## 21.6. Funktionssicherheit

Funktionen, Trigger und Row-Level-Security-Policies erlauben Benutzern, Code in den Backend-Server einzufügen, den andere Benutzer unbeabsichtigt ausführen könnten. Daher ermöglichen diese Mechanismen Benutzern mit relativer Leichtigkeit, andere mit „Trojanischen Pferden“ anzugreifen. Der stärkste Schutz ist eine strenge Kontrolle darüber, wer Objekte definieren darf. Wo dies nicht machbar ist, schreiben Sie Abfragen, die nur auf Objekte mit vertrauenswürdigen Eigentümern verweisen. Entfernen Sie aus `search_path` alle Schemata, die nicht vertrauenswürdigen Benutzern das Erstellen von Objekten erlauben.

Funktionen laufen innerhalb des Backend-Serverprozesses mit den Betriebssystemberechtigungen des Datenbankserver-Daemons. Wenn die für die Funktion verwendete Programmiersprache ungeprüfte Speicherzugriffe erlaubt, ist es möglich, die internen Datenstrukturen des Servers zu ändern. Daher können solche Funktionen unter anderem jede Systemzugriffskontrolle umgehen. Funktionssprachen, die solchen Zugriff erlauben, gelten als „nicht vertrauenswürdig“, und PostgreSQL erlaubt nur Superusern, Funktionen in solchen Sprachen zu erstellen.
