# MySQL 8.4 における innodb_lock_wait_timeout の意味論

対象は MySQL 8.4 community の公式動作。Aurora がこの変数を上書きしない限り、ここが継承される前提の意味論。Aurora 固有の上書き有無は本ノートでは未調査。

取得日は 2026-10-08。引用は取得時点のページ本文。エラーリファレンスの表紙は "Document generated on: 2026-10-01 (revision: 84962)"。

用語（初出で英語を併記する）: row lock（行ロック）、metadata lock / MDL（メタデータロック）、gap lock（ギャップロック）、next-key lock（ネクストキーロック）、insert intention lock（挿入インテンションロック）、consistent read（一貫性読み取り、非ロック）、locking read（ロック読み取り）、deadlock（デッドロック）、wait-for graph（待ちグラフ）。

## innodb_lock_wait_timeout の定義、スコープ、型、範囲、SET_VAR

### Takeaway
MySQL 8.4 の `innodb_lock_wait_timeout` は、InnoDB トランザクションが row lock を待ってあきらめるまでの秒数。Global かつ Session、動的、整数、デフォルト 50、最小 1、最大 1073741824、単位 seconds。`SET_VAR` ヒントは効かない。

### Cited Findings
- 定義ヘッダは `Command-Line Format --innodb-lock-wait-timeout=#`、`System Variable innodb_lock_wait_timeout`、`Scope Global, Session`、`Dynamic Yes`、`SET_VAR Hint Applies No`、`Type Integer`、`Default Value 50`、`Minimum Value 1`、`Maximum Value 1073741824`、`Unit seconds`。オプション一覧でも Cmd-line / Option file / System Var が Yes、Var Scope は Both、Dynamic は Yes。 — [InnoDB Startup Options and System Variables (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- 本文: "The length of time in seconds an InnoDB transaction waits for a row lock before giving up. The default value is 50 seconds. A transaction that tries to access a row that is locked by another InnoDB transaction waits at most this many seconds for write access to the row before issuing the following error"。表示例は `ERROR 1205 (HY000) : Lock wait timeout exceeded; try restarting transaction`（コロン前に空白があるのはこのページの表記）。 — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- "innodb_lock_wait_timeout can be set at runtime with the SET GLOBAL or SET SESSION statement. Changing the GLOBAL setting requires privileges sufficient to set global system variables ... and affects the operation of all clients that subsequently connect. Any client can change the SESSION setting for innodb_lock_wait_timeout, which affects only that client." — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- 8.0 マニュアルの同じ変数も `Default Value 50`、`Minimum Value 1`、`Maximum Value 1073741824`、`Unit seconds`、`Scope Global, Session`、`Dynamic Yes`、`SET_VAR Hint Applies No`。 — [InnoDB parameters (8.0)](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)

### Inferences
- 型が Integer で単位が seconds なので、公式上は小数秒を設定できない。最小 1 のため 0 で無効化はできない。最大 1073741824 秒は 2^30 秒（約 34.0 年）。この年換算はマニュアルの最大値からの計算であり、マニュアルは秒だけを記載する。
- `SET GLOBAL` は「その後に接続するクライアント」に効く、と書かれている。既に張られているセッションの session 値を書き換える、とは書かれていない。
- マニュアルの "for write access to the row" は文言として排他的だが、同じ変数は共有の locking read の待ちにも関わる（後述のロック節）。この一句だけを「X lock だけ」と読まない。

### Gaps
- `SET PERSIST` / `SET PERSIST_ONLY` がこの変数で使えるかは、変数節に明示がないので未確認。動的であることと、`SET GLOBAL` / `SET SESSION` が書かれていることだけが確認できた。
- Aurora がデフォルト、最小、最大、動的変更可否を上書きしているかは未調査。

## エラー 1205 と、文のロールバック対トランザクションのロールバック

### Takeaway
ロック待ちタイムアウトは error 1205、シンボル `ER_LOCK_WAIT_TIMEOUT`、SQLSTATE `HY000`、メッセージ `Lock wait timeout exceeded; try restarting transaction`。デフォルトでは待っていた文だけをロールバックし、トランザクションは開いたまま。全体を戻すには起動時の `innodb_rollback_on_timeout`（デフォルト OFF、非動的）が必要。デッドロック（1213）は常にトランザクション全体。

### Cited Findings
- Error number 1205、Symbol `ER_LOCK_WAIT_TIMEOUT`、SQLSTATE `HY000`、Message `Lock wait timeout exceeded; try restarting transaction`。"InnoDB reports this error when lock wait timeout expires. The statement that waited too long was rolled back (not the entire transaction)." 続けて、長く待つなら `innodb_lock_wait_timeout` を上げ、長いトランザクションが concurrency を落とすなら下げる、とある。 — [MySQL 8.4 Error Message Reference](https://dev.mysql.com/doc/mysql-errors/8.4/en/server-error-reference.html#error_er_lock_wait_timeout)（表紙の生成日 2026-10-01）
- Error number 1213、Symbol `ER_LOCK_DEADLOCK`、SQLSTATE `40001`、Message `Deadlock found when trying to get lock; try restarting transaction`。"InnoDB reports this error when a transaction encounters a deadlock and is automatically rolled back"。回復は "run all the operations in this transaction again"。 — [同エラーリファレンス](https://dev.mysql.com/doc/mysql-errors/8.4/en/server-error-reference.html#error_er_lock_deadlock)
- エラー処理: "A transaction deadlock causes InnoDB to roll back the entire transaction. Retry the entire transaction when this happens." "A lock wait timeout causes InnoDB to roll back the current statement (the statement that was waiting for the lock and encountered the timeout). To have the entire transaction roll back, start the server with --innodb-rollback-on-timeout enabled. Retry the statement if using the default behavior, or the entire transaction if --innodb-rollback-on-timeout is enabled." — [InnoDB Error Handling (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-error-handling.html)
- 同じ節の別項目: "Other errors are mostly detected by the MySQL layer of code (above the InnoDB storage engine level), and they roll back the corresponding SQL statement. Locks are not released in a rollback of a single SQL statement." この "Locks are not released..." は、ロック待ちタイムアウトの箇条書きではなく、その後の "Other errors" の文に付いている。 — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-error-handling.html)
- `innodb_rollback_on_timeout`: `Command-Line Format --innodb-rollback-on-timeout[={OFF|ON}]`、`Scope Global`、`Dynamic No`、`SET_VAR Hint Applies No`、`Type Boolean`、`Default Value OFF`。"InnoDB rolls back only the last statement on a transaction timeout by default. If --innodb-rollback-on-timeout is specified, a transaction timeout causes InnoDB to abort and roll back the entire transaction." オプション一覧の Dynamic 列も No。 — [InnoDB parameters (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_rollback_on_timeout)
- 変数節も同じ分割を繰り返す: タイムアウト時は "the current statement is rolled back (not the entire transaction)"。全体を戻すにはサーバを `--innodb-rollback-on-timeout` 付きで起動する。 — [innodb_lock_wait_timeout (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- 8.4 ソース `storage/innobase/lock/lock0wait.cc` のコメント: タイムアウト処理は `DB_DEADLOCK` を上書きしない。理由は "it rolls back whole transaction, not just the current query"。タイムアウト側は `trx->error_state = DB_LOCK_WAIT_TIMEOUT`。 — [mysql-server 8.4 lock0wait.cc](https://raw.githubusercontent.com/mysql/mysql-server/8.4/storage/innobase/lock/lock0wait.cc)（2026-10-08 取得）
- Percona（2020-07-27、8.4 専用の実験ではない）: OFF ではタイムアウトした文だけが失敗し、同じトランザクション内で先に成功した UPDATE は COMMIT すれば残る。ON では先に成功した変更も含めトランザクション全体が戻る。 — [Percona: The Transaction Behavior Impact of innodb_rollback_on_timeout](https://www.percona.com/blog/the-transaction-behavior-impact-of-innodb_rollback_on_timeout-in-mysql/)

### Inferences
- 1205 の SQLSTATE は汎用の `HY000` であり、デッドロックの `40001`（serialization failure）とは違う。SQLSTATE だけを見てリトライする実装は 1205 を取りこぼす。
- デフォルト（`innodb_rollback_on_timeout=OFF`）では、タイムアウトした文より前の文の変更はトランザクションに残る。クライアントがエラーを無視して COMMIT すると、その先行変更は永続化する。Percona の実験はこの公式文と一致する。実験自体は 2020 年で、8.4 上の再実験ではない。
- "Locks are not released in a rollback of a single SQL statement." を 1205 にそのまま適用するのは、マニュアルの箇条書き構造上は飛躍。1205 についてマニュアルが直接言うのは「待っていた文のロールバック」であり、先行文が取ったロックはトランザクションが開いたままなので保持される、というところまでが安全な読み。タイムアウトした文自身が待ちに入る前に取ったロックが文ロールバックで解放されるかは、8.4 マニュアルの一文としては未確定。
- `Dynamic No` なので、`innodb_rollback_on_timeout` の変更にはサーバ再起動が必要。実行中の `SET` では切り替えられない。

### Gaps
- 8.4 の `ha_innodb.cc` で `row_rollback_on_timeout` を `thd_mark_transaction_to_rollback` に渡す箇所は、今回 8.4 ファイルとしては未読。一致するコメントは 8.0 ブランチの `ha_innodb.cc` スニペット（`DB_LOCK_WAIT_TIMEOUT` は文ロールバック、`row_rollback_on_timeout` が真ならトランザクション印）として見えている。8.4 の確定根拠はマニュアルと `lock0wait.cc` のコメント。
- 1205 のあと、失敗した文が取ったロックだけが残るか消えるかは、上記のとおりマニュアルが 1205 の箇条に書いていない。

## 何に適用し、何に適用しないか。混同しやすいタイムアウト

### Takeaway
`innodb_lock_wait_timeout` は InnoDB の row lock 待ち。MySQL の table lock 待ちには適用しない。MDL は別変数 `lock_wait_timeout`（デフォルト 31536000 秒、1 年）で、タイムアウト時のエラー番号は同じ 1205。接続のアイドル切断は `wait_timeout` / `interactive_timeout`（どちらもデフォルト 28800 秒）。ソケットの読み待ちは `net_read_timeout`（デフォルト 30 秒）。デッドロック検出は `innodb_deadlock_detect`（デフォルト ON）で、有効時はデッドロックにこのタイムアウトは使わない。

### Cited Findings
- "`innodb_lock_wait_timeout` applies to InnoDB row locks. A MySQL table lock does not happen inside InnoDB and this timeout does not apply to waits for table locks." — [innodb_lock_wait_timeout (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- "The lock wait timeout value does not apply to deadlocks when `innodb_deadlock_detect` is enabled (the default) because InnoDB detects deadlocks immediately and rolls back one of the deadlocked transactions. When `innodb_deadlock_detect` is disabled, InnoDB relies on `innodb_lock_wait_timeout` for transaction rollback when a deadlock occurs." — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- `lock_wait_timeout`: `Scope Global, Session`、`Dynamic Yes`、`SET_VAR Hint Applies Yes`、`Type Integer`、`Default Value 31536000`、`Minimum Value 1`、`Maximum Value 31536000`、`Unit seconds`。"This variable specifies the timeout in seconds for attempts to acquire metadata locks." デフォルトは 31536000。"This timeout applies to all statements that use metadata locks. These include DML and DDL operations on tables, views, stored procedures, and stored functions, as well as LOCK TABLES, FLUSH TABLES WITH READ LOCK, and HANDLER statements." — [Server System Variables (8.4), lock_wait_timeout](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_lock_wait_timeout)
- 同じ変数: "The timeout value applies separately for each metadata lock attempt. A given statement can require more than one lock, so it is possible for the statement to block for longer than the lock_wait_timeout value before reporting a timeout error. When lock timeout occurs, ER_LOCK_WAIT_TIMEOUT is reported." `LOCK INSTANCE FOR BACKUP` の待ち時間もこれ。 — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_lock_wait_timeout)
- `wait_timeout`: `Scope Global, Session`、`Dynamic Yes`、`SET_VAR Hint Applies No`、`Default Value 28800`、`Minimum Value 1`、`Maximum Value (Windows) 2147483`、`Maximum Value (Other) 31536000`、`Unit seconds`。"The number of seconds the server waits for activity on a noninteractive connection before closing it." セッション初期値は、クライアントが `CLIENT_INTERACTIVE` なら global `interactive_timeout`、そうでなければ global `wait_timeout`。 — [同ページ wait_timeout](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_wait_timeout)
- `interactive_timeout`: `Default Value 28800`、`Minimum Value 1`、`Maximum Value 31536000`、`Unit seconds`、`Scope Global, Session`、`Dynamic Yes`、`SET_VAR Hint Applies No`。"The number of seconds the server waits for activity on an interactive connection before closing it." — [同ページ interactive_timeout](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_interactive_timeout)
- `net_read_timeout`: `Default Value 30`、`Minimum Value 1`、`Maximum Value 31536000`、`Unit seconds`、`Scope Global, Session`、`Dynamic Yes`、`SET_VAR Hint Applies No`。"The number of seconds to wait for more data from a connection before aborting the read." 書き込み側は `net_write_timeout`。 — [同ページ net_read_timeout](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_net_read_timeout)
- `net_write_timeout`: `Default Value 60`、同じ最小・最大・単位、`SET_VAR Hint Applies No`。 — [同ページ net_write_timeout](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_net_write_timeout)
- `innodb_deadlock_detect`: `Scope Global`、`Dynamic Yes`、`SET_VAR Hint Applies No`、`Type Boolean`、`Default Value ON`。"This option is used to disable deadlock detection." 高並列で同じロックを多数が待つと検出が遅くなることがあり、そのときは無効化して `innodb_lock_wait_timeout` に任せる、とある。 — [innodb_deadlock_detect (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_deadlock_detect)
- Performance Schema のロック表は四つに分かれる。`data_locks`（保持中と要求中のデータロック）、`data_lock_waits`（所有者と待っている側）、`metadata_locks`（MDL）、`table_handles`（table lock）。 — [Performance Schema Lock Tables (8.4)](https://dev.mysql.com/doc/refman/8.4/en/performance-schema-lock-tables.html)
- MySQL 開発者（Jakub Łopuszański、2024-05-15、bugs.mysql.com）: `lock_wait_timeout` はサーバ層の MDL、`innodb_lock_wait_timeout` は InnoDB 層のデータロック、と切り分けている。 — [Bug #114949](https://bugs.mysql.com/bug.php?id=114949)

### Inferences
- 1205 は InnoDB row lock タイムアウトと MDL タイムアウトの両方で出る。エラー番号と SQLSTATE とメッセージでは区別できない。切り分けは `performance_schema.data_lock_waits` と `metadata_locks`。
- `innodb_lock_wait_timeout` は `SET_VAR` 不可。`lock_wait_timeout` は `SET_VAR` 可。ヒントで片方だけ変えたつもりが、もう片方は変わらない。
- `wait_timeout` と `interactive_timeout` はロック待ちではなく、接続が無活動のまま生きる時間。ロック待ち中の文は「活動」なので、これらが 50 秒の row lock 待ちを切る、とはマニュアルは書いていない。この点はマニュアルの直接の一文ではない。

### Gaps
- InnoDB 内部の table-level lock（intention lock の IS/IX、AUTO-INC lock）が `innodb_lock_wait_timeout` の対象か。マニュアルは対象を "InnoDB row locks" とし、"waits for table locks" には適用しないと言う。後者の "table lock" は MySQL の table lock を指す文脈。`lock0wait.cc` は `lock_table()` が作る wait-for edge にも言及するが、AUTO-INC 待ちがこの秒数で 1205 になるとは 8.4 マニュアルに書かれていない。
- `GET_LOCK()` のタイムアウトは関数引数であり、失敗時はエラー 1205 ではなく戻り値 0、という扱いは今回の変数節の範囲では深追いしていない。ユーザレベルロックのデッドロックは別エラー 3058 `ER_USER_LOCK_DEADLOCK`。

## デッドロック検出、1213 と 1205、wait-for graph の上限

### Takeaway
検出が ON（デフォルト）のとき、InnoDB はデッドロックを直ちに見つけてトランザクションを戻す。これは 1213 であり、`innodb_lock_wait_timeout` は使わない。検出が OFF のとき、および InnoDB から見えない table lock が絡むときは、タイムアウトがバックストップ。wait-for list が 200 トランザクション、または参照ロックが 1,000,000 を超えると、それもデッドロックとして扱う。この上限の記述は 8.4 に残っている。

### Cited Findings
- "When deadlock detection is enabled (the default), InnoDB automatically detects transaction deadlocks and rolls back a transaction or transactions to break the deadlock. InnoDB tries to pick small transactions to roll back, where the size of a transaction is determined by the number of rows inserted, updated, or deleted." — [Deadlock Detection (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlock-detection.html)
- "InnoDB is aware of table locks if `innodb_table_locks = 1` (the default) and `autocommit = 0` ... Otherwise, InnoDB cannot detect deadlocks where a table lock set by a MySQL LOCK TABLES statement or a lock set by a storage engine other than InnoDB is involved. Resolve these situations by setting the value of the innodb_lock_wait_timeout system variable." — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlock-detection.html)
- Monitor の `LATEST DETECTED DEADLOCK` に `TOO DEEP OR LONG SEARCH IN THE LOCK TABLE WAITS-FOR GRAPH, WE WILL ROLL BACK FOLLOWING TRANSACTION` が出るのは、"the number of transactions on the wait-for list has reached a limit of 200"。 "A wait-for list that exceeds 200 transactions is treated as a deadlock and the transaction attempting to check the wait-for list is rolled back. The same error may also occur if the locking thread must look at more than 1,000,000 locks owned by transactions on the wait-for list." — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlock-detection.html)
- 検出無効化: "it may be more efficient to disable deadlock detection and rely on the innodb_lock_wait_timeout setting for transaction rollback when a deadlock occurs." — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlock-detection.html)
- 変数節の対句: 検出 ON なら "detects deadlocks immediately"。検出 OFF ならタイムアウトに頼る。 — [innodb_lock_wait_timeout (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- 8.4 `lock0wait.cc`: `lock_wait_timeout_thread()` が待ちスロットを見てタイムアウトを起こし、同じループで wait-for graph を解析してデッドロックを解く。デッドロック探索は `if (innobase_deadlock_detect)` の中。 — [lock0wait.cc (8.4)](https://raw.githubusercontent.com/mysql/mysql-server/8.4/storage/innobase/lock/lock0wait.cc)

### Inferences
- 通常のデッドロックは待たずに 1213。タイムアウト（1205）は、検出 OFF、または InnoDB がグラフに載せられないロック（`LOCK TABLES`、他エンジン）のときのバックストップ。200 / 1,000,000 のケースはマニュアルが "treated as a deadlock" と言い、待っている側をタイムアウトさせるとは書いていない。
- 200 / 1,000,000 の段落はクライアントに出るエラー番号を書いていない。"treated as a deadlock" と "the same error" から 1213 と読むのは推論。マニュアルはその段落に `ERROR 1213` を載せていない。
- 検出 OFF のときの文言は "transaction rollback" だが、実際に通るのはタイムアウト経路。デフォルトでは 1205 の文ロールバックであり、`innodb_rollback_on_timeout=OFF` のままではトランザクション全体にはならない、と読むのが変数節とエラー処理節を合わせた解釈。検出 OFF が 1213 を出す、とは書かれていない。

### Gaps
- 200 トランザクション / 1,000,000 ロックのときクライアントが受け取る errno が必ず 1213 であることは、8.4 マニュアルのその段落には番号がない。
- 検出 OFF のデッドロックが 1205 になり、かつ `innodb_rollback_on_timeout` の文ロールバック規則に従うことは、文言の組み合わせからの推論。8.4 でその組み合わせを実行した公式テスト結果は未確認。

## 待たせるロックと、待たない読み取り。アイソレーション

### Takeaway
デフォルトの isolation level は `REPEATABLE READ`。このレベルでは検索と index scan に next-key lock を使い、phantom を防ぐ。通常の `SELECT`（consistent read）はロックを取らないので row lock を待たない。`SELECT ... FOR UPDATE` / `FOR SHARE`、および `UPDATE` / `DELETE` はロックを取り、競合すれば待つ。gap lock 同士は競合しない。INSERT の insert intention lock は、既存の gap / next-key と競合して待ち得る。`READ COMMITTED` では検索の gap lock は原則無効。`SERIALIZABLE` で `autocommit` が OFF の平文 `SELECT` は `FOR SHARE` になり、待ち得る。

### Cited Findings
- InnoDB のデフォルト isolation は `REPEATABLE READ`。 — [InnoDB Transaction Isolation Levels (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- `REPEATABLE READ` のロック: "For a unique index with a unique search condition, InnoDB locks only the index record found, not the gap before it." それ以外の検索条件では "InnoDB locks the index range scanned, using gap locks or ... to block insertions by other sessions into the gaps covered by the range."（取得テキストは "using gap locks or to block" と助詞が崩れて見える。意味は gap lock または next-key lock で範囲をロックする、という節の続き。） — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- `READ COMMITTED`: locking read（`FOR UPDATE` / `FOR SHARE`）、`UPDATE`、`DELETE` について "InnoDB locks only index records, not the gaps before them, and thus permits the free insertion of new records next to locked records. Gap locking is only used for foreign-key constraint checking and duplicate-key checking." — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- `SERIALIZABLE`: "This level is like REPEATABLE READ, but InnoDB implicitly converts all plain SELECT statements to SELECT ... FOR SHARE if autocommit is disabled. If autocommit is enabled, the SELECT is its own transaction. It therefore is known to be read only and can be serialized if performed as a consistent (nonlocking) read and need not block for other transactions. (To force a plain SELECT to block if other transactions have modified the selected rows, disable autocommit.)" — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- "Consistent read is the default mode in which InnoDB processes SELECT statements in READ COMMITTED and REPEATABLE READ isolation levels. A consistent read does not set any locks on the tables it accesses, and therefore other sessions are free to modify those tables at the same time a consistent read is being performed on the table." より新しい状態が欲しければ `READ COMMITTED` か locking read（例 `SELECT * FROM t FOR SHARE`）。 — [Consistent Nonlocking Reads (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html)
- トランザクションモデル: "InnoDB performs locking at the row level and runs queries as nonlocking consistent reads by default"。 — [InnoDB Transaction Model (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-model.html)
- Record lock: "A record lock is a lock on an index record." 例 `SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE`。インデックスが無くても隠れ clustered index の record をロックする。status の例は `lock_mode X locks rec but not gap`。 — [InnoDB Locking (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- Gap lock: インデックスレコードの間、先頭の前、末尾の後をロックする。例 `SELECT c1 FROM t WHERE c1 BETWEEN 10 AND 20 FOR UPDATE` は、値が無くても 15 の挿入を妨げる。異なるトランザクションが同じ gap に gap S-lock と gap X-lock を同時に持てる。"Gap locks in InnoDB are “purely inhibitive”, which means that their only purpose is to prevent other transactions from inserting to the gap. Gap locks can co-exist. ... There is no difference between shared and exclusive gap locks. They do not conflict with each other, and they perform the same function." `READ COMMITTED` にすると "gap locking is disabled for searches and index scans"。 — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- Next-key lock: "a next-key lock is an index-record lock plus a gap lock on the gap preceding the index record." "By default, InnoDB operates in REPEATABLE READ transaction isolation level. In this case, InnoDB uses next-key locks for searches and index scans, which prevents phantom rows." — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- Insert intention lock: "An insert intention lock is a type of gap lock set by INSERT operations prior to row insertion." 同じ gap の別位置への挿入同士は待たない。例では、相手が gap を含む排他ロックを持っているとき `INSERT INTO child (id) VALUES (101)` が "takes an insert intention lock while it waits to obtain an exclusive lock." — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- Intention lock: `SELECT ... FOR SHARE` は IS、`SELECT ... FOR UPDATE` は IX。 — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- ロック種別の目次には Record Locks、Gap Locks、Insert Intention Locks、AUTO-INC Locks、Predicate Locks for Spatial Indexes がある。 — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- 待ちの状態: 非互換モードでロックされている行やテーブルを要求するとキューに入り、相手の COMMIT または ROLLBACK を待つ。`INNODB_TRX` の状態は待っていなければ `RUNNING`、待っていれば `LOCK WAIT`。 — [InnoDB Lock and Lock-Wait Information (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-information-schema-understanding-innodb-locking.html)

### Inferences
- 平文 `SELECT` が 1205 にならないのは、RC と RR では consistent read がロックを取らないから。待つのは locking read と、行を変更する DML。`SERIALIZABLE` かつ `autocommit=0` は例外で、平文 `SELECT` も `FOR SHARE` になり得る。
- gap lock 同士は待たない。待ちが起きるのは、insert intention や record / next-key の S/X が、既存の非互換ロックと衝突したとき。`BETWEEN` や非ユニーク検索の locking read が、存在しないキーへの INSERT を長く止めるのはこの組み合わせ。
- ユニーク索引で等値の一意検索なら、RR でも record だけで gap は取らない、とマニュアルは言う。複合ユニークの一部カラムだけ、という条件は gap を取る（ロック節の、取得済みの前文）。

### Gaps
- Predicate lock（空間索引）の待ちが `innodb_lock_wait_timeout` で 1205 になることは、8.4 の変数節に列挙がない。過去のサーバテストが空間索引の INSERT で `ER_LOCK_WAIT_TIMEOUT` を期待している例はあるが、それは 8.4 マニュアルの規定ではない。
- AUTO-INC lock の待ちがこの変数に従うかは、前節と同じくマニュアル未記載。

## タイマーは「待ち時間」か「文の経過時間」か。49 秒と 51 秒

### Takeaway
マニュアルが言うのは「row lock を待つ秒数」であり、文の開始からコミットまでの経過時間ではない。8.4 の実装は、ロック待ちに入るたびにその時点のタイムアウトをコピーし、suspend 時刻からの経過が「設定秒数より大きい」ときだけ 1205 にする。判定スレッドは約 1 秒に 1 回なので、壁時計では設定値をわずかに超え得る。ブロッカーがタイムアウト判定より前に COMMIT すれば待ちは解けて文は続く。判定後では 1205。連続する複数のロック待ちは、それぞれが自分のタイマーを持つ。

### Cited Findings
- マニュアルの範囲は "waits for a row lock" と "waits at most this many seconds for write access to the row"。文全体の実行時間、とは書いていない。ロック要求ごとのリセット、とも書いていない。 — [innodb_lock_wait_timeout (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- 対照: MDL の `lock_wait_timeout` は明示的に "The timeout value applies separately for each metadata lock attempt. A given statement can require more than one lock, so it is possible for the statement to block for longer than the lock_wait_timeout value"。InnoDB 変数節にこの文はない。 — [lock_wait_timeout (8.4)](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_lock_wait_timeout)
- 8.4 `lock_wait_suspend_thread`: 待ちに入るとき `trx_lock_wait_timeout_get(trx)` を読み、`lock_wait_table_reserve_slot` が `slot->suspend_time = now`、`slot->wait_timeout = wait_timeout` をセットする。 — [lock0wait.cc (8.4)](https://raw.githubusercontent.com/mysql/mysql-server/8.4/storage/innobase/lock/lock0wait.cc)
- 判定: `wait_time = now - slot->suspend_time`。タイムアウトは `slot->wait_timeout < seconds{100000000} && wait_time > slot->wait_timeout`（`>` であり `>=` ではない）。真なら `DB_LOCK_WAIT_TIMEOUT` を立て、待っているロック要求を `lock_cancel_waiting_and_release` で外す。 — [同ファイル](https://raw.githubusercontent.com/mysql/mysql-server/8.4/storage/innobase/lock/lock0wait.cc)
- 判定スレッドのコメント: "Calling this more often than once a second isn't needed, as lock timeouts are specified with one second resolution, so probably nobody cares if we wake up after T or T+0.99, when T itself can't be precise." 実装は前回チェックから 1 秒以上経っていれば `lock_wait_check_slots_for_timeouts()` を呼び、待ちがいるときは "we wake up every second (at worst)"。 — [同ファイル `lock_wait_timeout_thread`](https://raw.githubusercontent.com/mysql/mysql-server/8.4/storage/innobase/lock/lock0wait.cc)
- `trx_lock_wait_timeout_get`（8.4）は `return thd_lock_wait_timeout(t->mysql_thd);`。直前のコメントは「`mysql_thd` が無いトランザクションは timeout を 0 にする」と書くが、関数本体に 0 を返す分岐は無い。 — [trx0trx.h (8.4)](https://raw.githubusercontent.com/mysql/mysql-server/8.4/storage/innobase/include/trx0trx.h)
- 同じ `lock0wait.cc` の別コメント: "`trx->mysql_thd == NULL`" のシステムトランザクション（purge、クラッシュリカバリ後の未完了トランザクションのロールバック）は "`innodb_lock_wait_timeout` の global 値" を使う。 — [lock0wait.cc (8.4)](https://raw.githubusercontent.com/mysql/mysql-server/8.4/storage/innobase/lock/lock0wait.cc)
- 8.0 の `thd_lock_wait_timeout` コメント: `thd == NULL` は session 変数の global 値を返す。`return std::chrono::seconds{THDVAR(thd, lock_wait_timeout)};` — [ha_innodb.cc (8.0)](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/handler/ha_innodb.cc)（検索スニペット。8.4 ファイルは未読）
- row lock 待ちの統計は `thr->lock_state == QUE_THR_LOCK_ROW` のときだけ `n_lock_wait_count` を増やし、起床時に `suspend_time` からの差分をマイクロ秒で足す。 — [lock0wait.cc (8.4)](https://raw.githubusercontent.com/mysql/mysql-server/8.4/storage/innobase/lock/lock0wait.cc)

### Inferences
- タイマーが動くのは、そのロック要求のためにスレッドが suspend している間だけ。ロックを待っていない実行時間（CPU、I/O、前の行の処理）は、この変数のカウントに入らない。これはマニュアルの "waits for a row lock" と、ソースが `suspend_time` からの差分を見ることの両方に合う。
- 1 回の `lock_wait_suspend_thread` が 1 つのタイマー。次のロック要求で再び suspend すれば、新しい `suspend_time` と、その時点の session 値でやり直す。したがって、文がロックを順番に複数回待つと、文のブロック時間の合計は 50 秒を超え得る。MDL 側にはこの文があるが、InnoDB 側のマニュアルには無い。InnoDB 側はソースからの帰結。
- デフォルト 50 のとき、ブロッカーが待ち開始から 49 秒で COMMIT すれば、まだ `wait_time > 50s` ではないのでタイムアウト経路には入らない。ロックが付与されれば文は続く。51 秒時点でまだ待っていて、判定スレッドが `wait_time > 50s` を見たら 1205。ただし判定は最大約 1 秒間隔で、比較は厳密な `>` なので、「ちょうど 50.000 秒で必ず切れる」とは実装コメント自身が言っていない。切れるのは T より後、おおよそ T+1 秒以内、という解像度。
- 待ち開始時に slot へコピーするので、待ちの途中で `SET SESSION innodb_lock_wait_timeout` しても、進行中のその待ちの期限は変わらない、と読める。これはソースの読みであり、マニュアルは途中変更に触れていない。

### Gaps
- `trx0trx.h` のコメント（timeout を 0 にする）と、関数本体および `lock0wait.cc` のコメント（NULL なら global）が 8.4 ソース内で食い違う。8.4 の `thd_lock_wait_timeout` 実装は未読。8.0 では NULL が global。システムトランザクションの実効値は、コメント矛盾が残るので「global と読むのが lock0wait.cc と 8.0 の handler」としか言えない。
- 高優先度トランザクションが、ブロッカーが非高優先度のときタイムアウトキャンセルをスキップする分岐が `lock0wait.cc` にある（"An HP trx should not give up if the blocker is not HP."）。何がユーザから見て high priority になるかは今回未確認。マニュアルの変数節にはこの例外が無い。
- 「49 秒で助かり、51 秒では助からない」は上記の実装からの説明であり、マニュアルに 49 / 51 の例は無い。

## Performance Schema と INFORMATION_SCHEMA

### Takeaway
8.4 で現行のロック観測は `performance_schema.data_locks` と `performance_schema.data_lock_waits`、および `information_schema.INNODB_TRX`。`INNODB_LOCKS` と `INNODB_LOCK_WAITS` は 8.0 で後継に置き換わっている。5.7 マニュアルは「MySQL 8.0 で削除」、8.0 日本語マニュアルは「8.0.1 の時点で削除」と書く。8.4 英語のロック節はそれらを現行テーブルとして挙げない。

### Cited Findings
- 8.4: 監視に使うのは `INFORMATION_SCHEMA.INNODB_TRX` 一つと、Performance Schema の `data_locks` と `data_lock_waits` 二つ。`data_locks` は保持ロックと、ブロックされて待っている要求。`INNODB_TRX.TRX_STATE` は `RUNNING`、`LOCK WAIT`、`ROLLING BACK`、`COMMITTING`。`LOCK WAIT` のトランザクションはちょうど一つの blocking lock request にブロックされる、と説明している。 — [InnoDB INFORMATION_SCHEMA Transaction and Locking Information (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-information-schema-transactions.html)
- `data_lock_waits` は、要求側 `REQUESTING_ENGINE_LOCK_ID` と、それを止めている `BLOCKING_ENGINE_LOCK_ID`。 — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/innodb-information-schema-transactions.html)
- これらの表は瞬間の内部状態であり、三表の間で一貫している保証は無い。`SELECT` 実行時点のロックは、クライアントが結果を読むまでに消え得る。 — [Persistence and Consistency (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-information-schema-internal-data.html)
- `sys.innodb_lock_waits` / `x$innodb_lock_waits` は待ちの要約。`wait_started`、`wait_age`、`wait_age_secs`、待っている SQL（`waiting_query`）、ブロック側の SQL（`blocking_query`。発行セッションが idle になると NULL）。 — [sys schema innodb_lock_waits (8.4)](https://dev.mysql.com/doc/refman/8.4/en/sys-innodb-lock-waits.html)
- 8.0 英語: `data_locks` と `data_lock_waits` は "`INFORMATION_SCHEMA` `INNODB_LOCKS` and `INNODB_LOCK_WAITS` tables in MySQL 8.0" を "supersede" する。 — [8.0 locking information](https://dev.mysql.com/doc/refman/8.0/en/innodb-information-schema-transactions.html)
- 5.7 英語: `INNODB_LOCK_WAITS` は "deprecated as of MySQL 5.7.14 and is removed in MySQL 8.0." `INNODB_LOCKS` も同じ注記。 — [5.7 INNODB_LOCK_WAITS](https://dev.mysql.com/doc/refman/5.7/en/information-schema-innodb-lock-waits-table.html)、[5.7 INNODB_LOCKS](https://dev.mysql.com/doc/refman/5.7/en/information-schema-innodb-locks-table.html)
- 8.0 日本語: `INNODB_LOCK_WAITS` は「非推奨で、MySQL 8.0.1 の時点で削除されています。代わりに、パフォーマンススキーマ `data_lock_waits` テーブルを使用してください。」権限も変わる。旧表は global `PROCESS`、`data_lock_waits` は Performance Schema の通常の `SELECT` 権限。 — [8.0 日本語 INNODB_LOCK_WAITS](https://dev.mysql.com/doc/refman/8.0/ja/information-schema-innodb-lock-waits-table.html)

### Inferences
- 8.4 で `information_schema.INNODB_LOCKS` / `INNODB_LOCK_WAITS` を照会する監視は、8.0 で既に後継へ移った前提で動かない。使うのは `data_locks`、`data_lock_waits`、`INNODB_TRX`、必要なら `sys.innodb_lock_waits`。
- スナップショットは一貫していないので、`data_lock_waits` の blocking id が `data_locks` に無い行は、競合条件としてマニュアルが予告している。

### Gaps
- 8.4 英語マニュアルに "INNODB_LOCKS is removed" という一文を、今回の取得範囲では見つけていない。削除の明示は 5.7 英語（removed in 8.0）と 8.0 日本語（8.0.1 で削除）。8.4 は現行手段として PS 表と `INNODB_TRX` だけを書く。
- `data_locks.LOCK_MODE` の 8.4 での全表記（`X,REC_NOT_GAP`、`X,GAP`、`X`（next-key）、`X,INSERT_INTENTION` など）の公式一覧は、今回変数調査の範囲では表定義ページを全文取得していない。

## ステータスカウンタ

### Takeaway
`Innodb_row_lock_waits` は待ち回数。`Innodb_row_lock_current_waits` は今待っている数。時間三つはミリ秒。`Innodb_row_lock_time` は合計、`_avg` は平均、`_max` は最大。いずれも row lock の取得に費やした時間、とマニュアルは言う。

### Cited Findings
- `Innodb_row_lock_current_waits`: "The number of row locks currently waited for by operations on InnoDB tables." — [Server Status Variables (8.4)](https://dev.mysql.com/doc/refman/8.4/en/server-status-variables.html#statvar_Innodb_row_lock_current_waits)
- `Innodb_row_lock_time`: "The total time spent in acquiring row locks for InnoDB tables, in milliseconds." — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/server-status-variables.html#statvar_Innodb_row_lock_time)
- `Innodb_row_lock_time_avg`: "The average time to acquire a row lock for InnoDB tables, in milliseconds." — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/server-status-variables.html#statvar_Innodb_row_lock_time_avg)
- `Innodb_row_lock_time_max`: "The maximum time to acquire a row lock for InnoDB tables, in milliseconds." — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/server-status-variables.html#statvar_Innodb_row_lock_time_max)
- `Innodb_row_lock_waits`: "The number of times operations on InnoDB tables had to wait for a row lock." — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/server-status-variables.html#statvar_Innodb_row_lock_waits)
- オプション一覧ではこれらは Status var で、Scope Global、Dynamic No（設定項目ではない）。 — [InnoDB parameters 一覧 (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html)
- ソースは row lock 待ち（`QUE_THR_LOCK_ROW`）のときだけカウントを増やし、待ち時間をマイクロ秒で積む。ステータス変数の公開単位はマニュアルが milliseconds と指定している。 — [lock0wait.cc (8.4)](https://raw.githubusercontent.com/mysql/mysql-server/8.4/storage/innobase/lock/lock0wait.cc)、[status variables (8.4)](https://dev.mysql.com/doc/refman/8.4/en/server-status-variables.html#statvar_Innodb_row_lock_time)

### Inferences
- 時間カウンタの単位は秒ではない。`innodb_lock_wait_timeout` が秒、カウンタがミリ秒。50 秒待ち 1 回は、おおよそ `Innodb_row_lock_time` に 50000 前後が足さる規模。マイクロ秒積算からミリ秒表示への丸めは、公開経路を 8.4 で追っていないので「マニュアルの単位は milliseconds」までが確定。
- カウンタはグローバル。セッションごとの内訳はこれらの status 変数には無い。セッション単位の待ちは `sys.innodb_lock_waits` の `wait_age_secs` 側。

### Gaps
- `Innodb_row_lock_time` が「成功した取得も含む」のか「待った時間だけ」なのか。マニュアルは "time spent in acquiring" / "time to acquire" であり、待ちだけとは言っていない。ソースは待ちスロットに入った row lock の suspend 差分を足しているので、即時取得（待たなかったロック）は入らない、と読める。これはソース解釈。
- 平均の分母が `Innodb_row_lock_waits` と厳密に一致するかは未確認。

## 8.4 でデフォルトは変わったか

### Takeaway
変わっていない。8.0 も 8.4 もデフォルト 50 秒、最小 1、最大 1073741824。8.4.0 でデフォルトが変わった InnoDB 変数の公式一覧（Table 1.1）に `innodb_lock_wait_timeout` は無い。変わったのは buffer pool、adaptive hash index、I/O capacity、log buffer など別の変数。

### Cited Findings
- 8.4 の Default Value は 50。 — [8.4 innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- 8.0 の Default Value も 50、最小 1、最大 1073741824、単位 seconds。 — [8.0 innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- "The default values for a number of server system variables relating to the InnoDB storage engine were changed in MySQL 8.4.0" として **Table 1.1 InnoDB system variable default values in MySQL 8.4 differing from MySQL 8.0** がある。取得テキストに並ぶ名前は `innodb_buffer_pool_in_core_file`、`innodb_buffer_pool_instances`、`innodb_adaptive_hash_index`（8.4 は OFF、8.0 は ON）、`innodb_doublewrite_files`、`innodb_doublewrite_pages`、`innodb_flush_method`、`innodb_io_capacity`（8.4 は 10000、8.0 は 200）、`innodb_io_capacity_max`、`innodb_log_buffer_size`（67108864、以前 16777216）、`innodb_numa_interleave`、`innodb_page_cleaners`、`innodb_parallel_read_threads`、`innodb_purge_threads`、`innodb_read_io_threads`、`innodb_use_fdatasync`、および `temptable_max_ram` / `temptable_max_mmap` / `temptable_use_mmap`。この差分表の取得範囲に `innodb_lock_wait_timeout` は無い。 — [What Is New in MySQL 8.4 since MySQL 8.0](https://dev.mysql.com/doc/refman/8.4/en/mysql-nutshell.html)
- `--innodb-dedicated-server` について、同表は「この変数自体のデフォルトは OFF のままで 8.0 から変わらない」と注記している。デフォルト変更の表は「変わったもの」を載せ、変わらないものは原則載せない。 — [同ページ](https://dev.mysql.com/doc/refman/8.4/en/mysql-nutshell.html)

### Inferences
- 「8.4 で多くの InnoDB デフォルトが変わった」は Table 1.1 のとおり事実。その集合に `innodb_lock_wait_timeout` は含まれず、8.0 マニュアルと 8.4 マニュアルの Default Value がどちらも 50 なので、この変数は変わっていない。
- `innodb_deadlock_detect` のデフォルト ON、`innodb_rollback_on_timeout` のデフォルト OFF も、8.4 変数節の現行値。これらが 8.0 から変わったとは Table 1.1 に出てこない。8.0 側の両変数の Default Value 行は今回未引用。

### Gaps
- Oracle のブログ記事そのものは今回開いていない。確認に使ったのは 8.4 リファレンスマニュアルの "What Is New" と、8.0 / 8.4 の変数定義。
- 8.4 のマイナーリリース（8.4.1 以降）でこのデフォルトが再び変わった、というリリースノート行は検索していない。8.4 マニュアルの現行 Default Value が 50 であることは 2026-10-08 時点の変数ページで確認した。

## 公式のチューニング文言

### Takeaway
対話的な OLTP では下げ、長いバックエンド処理では上げる、と 8.4 マニュアルが書いている。エラーリファレンスも、他トランザクションを長く待つなら上げ、長いトランザクションが concurrency を落とすなら下げると書く。デッドロックもロック待ちタイムアウトもビジーなサーバでは普通に起きるので、アプリケーションはリトライする前提、とも書いている。

### Cited Findings
- "You might decrease this value for highly interactive applications or OLTP systems, to display user feedback quickly or put the update into a queue for processing later. You might increase this value for long-running back-end operations, such as a transform step in a data warehouse that waits for other large insert or update operations to finish." — [innodb_lock_wait_timeout (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- 1205 の説明: 文が他トランザクションの完了をより長く待つべきなら `innodb_lock_wait_timeout` を増やす。長いトランザクションが locking 問題を起こし busy system の concurrency を下げているなら減らす。 — [Error 1205 (8.4)](https://dev.mysql.com/doc/mysql-errors/8.4/en/server-error-reference.html#error_er_lock_wait_timeout)
- "Both deadlocks and lock wait timeouts are normal on busy servers and it is necessary for applications to be aware that they may happen and handle them by retrying. You can make them less likely by doing as little work as possible between the first change to data during a transaction and the commit, so the locks are held for the shortest possible time and for the smallest possible number of rows." — [InnoDB Error Handling (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-error-handling.html)
- 高並列で同じロックを多数が待つときは、デッドロック検出を切ってタイムアウトに任せる方が効率的なことがある。 — [Deadlock Detection (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlock-detection.html)、[innodb_deadlock_detect (8.4)](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_deadlock_detect)

### Inferences
- マニュアルは「OLTP は短く、ウェアハウスの変換段は長く」という方向だけを示し、OLTP の推奨秒数（1 秒、5 秒など）は書いていない。数値の推奨をマニュアル由来として作らない。
- リトライ単位は設定次第。デフォルトは文のリトライで足りるが、トランザクション内の先行変更が残る。`innodb_rollback_on_timeout=ON` やデッドロック 1213 ではトランザクション全体のやり直し。

### Gaps
- 「何秒が適切か」の公式な数値は無い。50 がデフォルトであることと、上げ下げの方向だけが公式。
- Aurora がこのガイダンスと違うデフォルトや上限を持つかは未調査。
