# Anhang O. Veraltete oder umbenannte Funktionen

Funktionalität wird gelegentlich aus PostgreSQL entfernt, Namen von Funktionen, Einstellungen oder Dateien ändern sich, oder Dokumentation wandert an andere Stellen. Dieser Abschnitt verweist Benutzerinnen und Benutzer, die aus alten Versionen der Dokumentation oder über externe Links kommen, auf die passende neue Stelle für die gesuchten Informationen.

### O.1. Datei `recovery.conf` in `postgresql.conf` integriert

PostgreSQL 11 und ältere Versionen verwendeten eine Konfigurationsdatei namens `recovery.conf`, um Replikate und Standby-Server zu verwalten. Die Unterstützung für diese Datei wurde in PostgreSQL 12 entfernt. Einzelheiten zu dieser Änderung finden Sie in den Versionshinweisen zu PostgreSQL 12.

Ab PostgreSQL 12 werden Archive Recovery, Streaming Replication und PITR über normale Serverkonfigurationsparameter eingerichtet. Diese werden wie andere Parameter in `postgresql.conf` oder per `ALTER SYSTEM` gesetzt.

Der Server startet nicht, wenn eine Datei `recovery.conf` vorhanden ist.

PostgreSQL 15 und ältere Versionen hatten eine Einstellung `promote_trigger_file`, vor Version 12 `trigger_file`. Verwenden Sie stattdessen `pg_ctl promote` oder rufen Sie `pg_promote()` auf, um einen Standby-Server zu promoten.

Die Einstellung `standby_mode` wurde entfernt. Stattdessen wird eine Datei `standby.signal` im Datenverzeichnis verwendet. Einzelheiten finden Sie unter Standby-Server-Betrieb.

### O.2. Default Roles in vordefinierte Rollen umbenannt

PostgreSQL 13 und ältere Versionen verwendeten den Begriff „Default Roles“. Da diese Rollen tatsächlich aber nicht geändert werden können und bei der Initialisierung als Teil des Systems installiert werden, ist „vordefinierte Rollen“ der passendere Begriff. Die aktuelle Dokumentation zu vordefinierten Rollen finden Sie in [Abschnitt 21.5](21_Datenbankrollen.md#215-vordefinierte-rollen). Einzelheiten zu dieser Änderung finden Sie in den Versionshinweisen zu PostgreSQL 14.

### O.3. `pg_xlogdump` in `pg_waldump` umbenannt

PostgreSQL 9.6 und ältere Versionen stellten einen Befehl namens `pg_xlogdump` zum Lesen von Write-Ahead-Log-Dateien (WAL-Dateien) bereit. Dieser Befehl wurde in `pg_waldump` umbenannt. Die Dokumentation zu `pg_waldump` finden Sie bei `pg_waldump`; Einzelheiten zu dieser Änderung finden Sie in den Versionshinweisen zu PostgreSQL 10.

### O.4. `pg_resetxlog` in `pg_resetwal` umbenannt

PostgreSQL 9.6 und ältere Versionen stellten einen Befehl namens `pg_resetxlog` zum Zurücksetzen von Write-Ahead-Log-Dateien (WAL-Dateien) bereit. Dieser Befehl wurde in `pg_resetwal` umbenannt. Die Dokumentation zu `pg_resetwal` finden Sie bei `pg_resetwal`; Einzelheiten zu dieser Änderung finden Sie in den Versionshinweisen zu PostgreSQL 10.

### O.5. `pg_receivexlog` in `pg_receivewal` umbenannt

PostgreSQL 9.6 und ältere Versionen stellten einen Befehl namens `pg_receivexlog` zum Abrufen von Write-Ahead-Log-Dateien (WAL-Dateien) bereit. Dieser Befehl wurde in `pg_receivewal` umbenannt. Die Dokumentation zu `pg_receivewal` finden Sie bei `pg_receivewal`; Einzelheiten zu dieser Änderung finden Sie in den Versionshinweisen zu PostgreSQL 10.
