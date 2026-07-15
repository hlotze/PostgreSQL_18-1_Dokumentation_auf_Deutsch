# Anhang H. Externe Projekte

PostgreSQL ist ein komplexes Softwareprojekt, und die Verwaltung des Projekts ist anspruchsvoll. Es hat sich gezeigt, dass viele Erweiterungen für PostgreSQL effizienter außerhalb des Kernprojekts entwickelt werden können.

### H.1. Client-Schnittstellen

In der Basisdistribution von PostgreSQL sind nur zwei Client-Schnittstellen enthalten:

- `libpq` ist enthalten, weil es die primäre Schnittstelle für die Sprache C ist und weil viele andere Client-Schnittstellen darauf aufbauen.

- ECPG ist enthalten, weil es von der serverseitigen SQL-Grammatik abhängt und daher empfindlich auf Änderungen in PostgreSQL selbst reagiert.

Alle anderen Sprachschnittstellen sind externe Projekte und werden separat verteilt. Eine Liste der Sprachschnittstellen wird im PostgreSQL-Wiki gepflegt:

<https://wiki.postgresql.org/wiki/List_of_drivers>

Beachten Sie, dass einige dieser Pakete nicht unter derselben Lizenz wie PostgreSQL veröffentlicht werden. Weitere Informationen zu jeder Sprachschnittstelle, einschließlich der Lizenzbedingungen, finden Sie auf der jeweiligen Website und in der jeweiligen Dokumentation.

### H.2. Administrationswerkzeuge

Für PostgreSQL stehen mehrere Administrationswerkzeuge zur Verfügung. Das bekannteste ist pgAdmin; außerdem gibt es mehrere kommerziell verfügbare Werkzeuge.

<https://www.pgadmin.org/>

### H.3. Prozedurale Sprachen

PostgreSQL enthält mehrere prozedurale Sprachen in der Basisdistribution: PL/pgSQL, PL/Tcl, PL/Perl und PL/Python.

Darüber hinaus gibt es eine Reihe prozeduraler Sprachen, die außerhalb der PostgreSQL-Kerndistribution entwickelt und gepflegt werden. Eine Liste prozeduraler Sprachen wird im PostgreSQL-Wiki gepflegt:

<https://wiki.postgresql.org/wiki/PL_Matrix>

Beachten Sie, dass einige dieser Projekte nicht unter derselben Lizenz wie PostgreSQL veröffentlicht werden. Weitere Informationen zu jeder prozeduralen Sprache, einschließlich Lizenzinformationen, finden Sie auf der jeweiligen Website und in der jeweiligen Dokumentation.

### H.4. Erweiterungen

PostgreSQL ist so entworfen, dass es leicht erweiterbar ist. Deshalb können Erweiterungen, die in die Datenbank geladen werden, wie eingebaute Funktionen wirken. Das mit dem Quellcode ausgelieferte Verzeichnis `contrib/` enthält mehrere Erweiterungen, die in [Anhang F](F_Weitere_mitgelieferte_Module_und_Erweiterungen.md) beschrieben werden.

Andere Erweiterungen werden unabhängig entwickelt, etwa PostGIS:

<https://postgis.net/>

Sogar Replikationslösungen für PostgreSQL können extern entwickelt werden. Slony-I ist zum Beispiel eine verbreitete Primary/Standby-Replikationslösung, die unabhängig vom Kernprojekt entwickelt wird:

<https://www.slony.info>
