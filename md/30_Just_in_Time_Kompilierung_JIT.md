# 30. Just-in-Time-Kompilierung (JIT)

Dieses Kapitel erklärt, was Just-in-Time-Kompilierung ist und wie sie in PostgreSQL konfiguriert werden kann.

## 30.1. Was ist JIT-Kompilierung?

Just-in-Time-Kompilierung (JIT-Kompilierung) ist der Prozess, eine Form interpretierter Programmauswertung zur Laufzeit in ein natives Programm umzuwandeln. Statt beispielsweise allgemeinen Code zu verwenden, der beliebige SQL-Ausdrücke auswerten kann, um ein bestimmtes SQL-Prädikat wie `WHERE a.col = 3` auszuwerten, kann eine Funktion erzeugt werden, die speziell für diesen Ausdruck ist und nativ von der CPU ausgeführt werden kann. Das kann zu einer Beschleunigung führen.

PostgreSQL hat eingebaute Unterstützung für JIT-Kompilierung mit LLVM, wenn PostgreSQL mit `--with-llvm` gebaut wurde.

Weitere Details finden Sie in `src/backend/jit/README`.

### 30.1.1. Durch JIT beschleunigte Operationen

Derzeit unterstützt die JIT-Implementierung von PostgreSQL die Beschleunigung der Ausdrucksauswertung und des Tuple Deforming. Weitere Operationen könnten in Zukunft beschleunigt werden.

Ausdrucksauswertung wird verwendet, um `WHERE`-Klauseln, Ziellisten, Aggregate und Projektionen auszuwerten. Sie kann beschleunigt werden, indem für jeden Einzelfall spezifischer Code erzeugt wird.

Tuple Deforming ist der Prozess, ein auf Platte gespeichertes Tuple (siehe Abschnitt 66.6.1) in seine In-Memory-Darstellung umzuwandeln. Es kann beschleunigt werden, indem eine Funktion erzeugt wird, die speziell auf das Tabellenlayout und die Anzahl der zu extrahierenden Spalten zugeschnitten ist.

### 30.1.2. Inlining

PostgreSQL ist sehr erweiterbar und erlaubt, neue Datentypen, Funktionen, Operatoren und andere Datenbankobjekte zu definieren; siehe [Kapitel 36](36_SQL_erweitern.md). Tatsächlich werden die eingebauten Objekte mit nahezu denselben Mechanismen implementiert. Diese Erweiterbarkeit bringt einen gewissen Overhead mit sich, zum Beispiel durch Funktionsaufrufe (siehe [Abschnitt 36.3](36_SQL_erweitern.md#363-benutzerdefinierte-funktionen)). Um diesen Overhead zu verringern, kann JIT-Kompilierung die Rümpfe kleiner Funktionen in die Ausdrücke inline-en, die diese Funktionen verwenden. Dadurch kann ein erheblicher Teil des Overheads wegoptimiert werden.

### 30.1.3. Optimierung

LLVM unterstützt die Optimierung von erzeugtem Code. Einige Optimierungen sind günstig genug, um immer durchgeführt zu werden, wenn JIT verwendet wird, während andere nur für länger laufende Abfragen nützlich sind. Weitere Details zu Optimierungen finden Sie unter <https://llvm.org/docs/Passes.html#transform-passes>.

## 30.2. Wann JIT verwenden?

JIT-Kompilierung ist hauptsächlich für lang laufende CPU-gebundene Abfragen vorteilhaft. Häufig sind dies analytische Abfragen. Bei kurzen Abfragen ist der zusätzliche Overhead der JIT-Kompilierung oft größer als die Zeit, die sie einsparen kann.

Um zu bestimmen, ob JIT-Kompilierung verwendet werden soll, werden die geschätzten Gesamtkosten einer Abfrage verwendet (siehe Kapitel 69 und Abschnitt 19.7.2). Die geschätzten Kosten der Abfrage werden mit der Einstellung von `jit_above_cost` verglichen. Wenn die Kosten höher sind, wird JIT-Kompilierung durchgeführt. Danach sind zwei weitere Entscheidungen nötig. Erstens: Wenn die geschätzten Kosten höher sind als die Einstellung von `jit_inline_above_cost`, werden kurze Funktionen und Operatoren, die in der Abfrage verwendet werden, inline eingefügt. Zweitens: Wenn die geschätzten Kosten höher sind als die Einstellung von `jit_optimize_above_cost`, werden teure Optimierungen angewendet, um den erzeugten Code zu verbessern. Jede dieser Optionen erhöht den Overhead der JIT-Kompilierung, kann aber die Ausführungszeit der Abfrage erheblich verringern.

Diese kostenbasierten Entscheidungen werden zur Planungszeit getroffen, nicht zur Ausführungszeit. Das bedeutet: Wenn vorbereitete Anweisungen verwendet werden und ein generischer Plan verwendet wird (siehe `PREPARE`), steuern die zum Zeitpunkt der Vorbereitung wirksamen Werte der Konfigurationsparameter die Entscheidungen, nicht die Einstellungen zur Ausführungszeit.

> **Hinweis**
>
> Wenn `jit` auf `off` gesetzt ist oder keine JIT-Implementierung verfügbar ist, zum Beispiel weil der Server ohne `--with-llvm` kompiliert wurde, wird JIT nicht durchgeführt, selbst wenn es nach den obigen Kriterien vorteilhaft wäre. Das Setzen von `jit` auf `off` wirkt sowohl zur Planungs- als auch zur Ausführungszeit.

Mit `EXPLAIN` kann geprüft werden, ob JIT verwendet wird. Als Beispiel folgt eine Abfrage, die JIT nicht verwendet:

```sql
=# EXPLAIN ANALYZE SELECT SUM(relpages) FROM pg_class;
```

```text
                                                 QUERY PLAN
-------------------------------------------------------------------------------------------------------------
 Aggregate (cost=16.27..16.29 rows=1 width=8) (actual time=0.303..0.303 rows=1.00 loops=1)
   Buffers: shared hit=14
   -> Seq Scan on pg_class (cost=0.00..15.42 rows=342 width=4) (actual time=0.017..0.111 rows=356.00 loops=1)
         Buffers: shared hit=14
 Planning Time: 0.116 ms
 Execution Time: 0.365 ms
```

Angesichts der Kosten des Plans ist es völlig plausibel, dass kein JIT verwendet wurde; die Kosten von JIT wären höher gewesen als die möglichen Einsparungen. Eine Anpassung der Kostengrenzen führt zur Verwendung von JIT:

```sql
=# SET jit_above_cost = 10;
```

```text
SET
```

```sql
=# EXPLAIN ANALYZE SELECT SUM(relpages) FROM pg_class;
```

```text
                                                  QUERY PLAN
-------------------------------------------------------------------------------------------------------------
 Aggregate (cost=16.27..16.29 rows=1 width=8) (actual time=6.049..6.049 rows=1.00 loops=1)
    Buffers: shared hit=14
    -> Seq Scan on pg_class (cost=0.00..15.42 rows=342 width=4) (actual time=0.019..0.052 rows=356.00 loops=1)
          Buffers: shared hit=14
 Planning Time: 0.133 ms
 JIT:
    Functions: 3
    Options: Inlining false, Optimization false, Expressions true, Deforming true
    Timing: Generation 1.259 ms (Deform 0.000 ms), Inlining 0.000 ms, Optimization 0.797 ms, Emission 5.048 ms, Total 7.104 ms
 Execution Time: 7.416 ms
```

Wie hier zu sehen ist, wurde JIT verwendet, aber Inlining und teure Optimierung nicht. Wenn `jit_inline_above_cost` oder `jit_optimize_above_cost` ebenfalls gesenkt würden, würde sich das ändern.

## 30.3. Konfiguration

Die Konfigurationsvariable `jit` bestimmt, ob JIT-Kompilierung aktiviert oder deaktiviert ist. Wenn sie aktiviert ist, bestimmen die Konfigurationsvariablen `jit_above_cost`, `jit_inline_above_cost` und `jit_optimize_above_cost`, ob JIT-Kompilierung für eine Abfrage durchgeführt wird und wie viel Aufwand dafür betrieben wird.

`jit_provider` bestimmt, welche JIT-Implementierung verwendet wird. Dies muss nur selten geändert werden. Siehe Abschnitt 30.4.2.

Für Entwicklungs- und Debugging-Zwecke existieren einige zusätzliche Konfigurationsparameter, wie in [Abschnitt 19.17](19_Serverkonfiguration.md#1917-entwickleroptionen) beschrieben.

## 30.4. Erweiterbarkeit

### 30.4.1. Inlining-Unterstützung für Erweiterungen

Die JIT-Implementierung von PostgreSQL kann die Rümpfe von Funktionen der Typen C und `internal` sowie Operatoren, die auf solchen Funktionen basieren, inline einfügen. Damit dies für Funktionen in Erweiterungen möglich ist, müssen die Definitionen dieser Funktionen verfügbar gemacht werden. Wenn PGXS verwendet wird, um eine Erweiterung gegen einen Server zu bauen, der mit LLVM-JIT-Unterstützung kompiliert wurde, werden die relevanten Dateien automatisch gebaut und installiert.

Die relevanten Dateien müssen in `$pkglibdir/bitcode/$extension/` installiert werden und eine Zusammenfassung davon in `$pkglibdir/bitcode/$extension.index.bc`, wobei `$pkglibdir` das von `pg_config --pkglibdir` zurückgegebene Verzeichnis ist und `$extension` der Basisname der Shared Library der Erweiterung.

> **Hinweis**
>
> Für Funktionen, die in PostgreSQL selbst eingebaut sind, wird der Bitcode in `$pkglibdir/bitcode/postgres` installiert.

### 30.4.2. Austauschbare JIT-Provider

PostgreSQL stellt eine auf LLVM basierende JIT-Implementierung bereit. Die Schnittstelle zum JIT-Provider ist austauschbar, und der Provider kann ohne Neukompilieren geändert werden; derzeit stellt der Build-Prozess allerdings nur Inlining-Unterstützungsdaten für LLVM bereit. Der aktive Provider wird über die Einstellung `jit_provider` ausgewählt.

#### 30.4.2.1. JIT-Provider-Schnittstelle

Ein JIT-Provider wird geladen, indem die benannte Shared Library dynamisch geladen wird. Der normale Bibliothekssuchpfad wird verwendet, um die Bibliothek zu finden. Um die erforderlichen JIT-Provider-Callbacks bereitzustellen und anzuzeigen, dass die Bibliothek tatsächlich ein JIT-Provider ist, muss sie eine C-Funktion namens `_PG_jit_provider_init` bereitstellen. Dieser Funktion wird eine Struktur übergeben, die mit Callback-Funktionszeigern für einzelne Aktionen gefüllt werden muss:

```c
struct JitProviderCallbacks
{
    JitProviderResetAfterErrorCB reset_after_error;
    JitProviderReleaseContextCB release_context;
    JitProviderCompileExprCB compile_expr;
};

extern void _PG_jit_provider_init(JitProviderCallbacks *cb);
```
