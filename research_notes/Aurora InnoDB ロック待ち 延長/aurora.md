# Aurora MySQL における innodb_lock_wait_timeout 延長（50→120）と Serverless v2

調査日: 2026-10-08。一次情報は AWS ドキュメントと MySQL 8.4 リファレンス。バージョン番号は取得したページに書かれているものだけを使う。8.4.7 を名指ししないページは、近くのどの版を確認したかを書く。

コミュニティ MySQL と Aurora の両方に当てはまる事実と、Aurora 固有の事実は各項目で分けている。

## Aurora MySQL のどの版が MySQL 8.4 か。8.4.7 と、その次のパッチは何か

### Takeaway
MySQL 8.4 互換は Aurora MySQL version 3 ではない。メジャー系統の名前が Aurora MySQL version 8.4 で、最初のマイナーが 8.4.7（2026-05-21、Community MySQL 8.4.7）、確認できた最新パッチは 8.4.8（2026-09-03、Community MySQL 8.4.8）である。8.4.8 で `aurora_transaction_timeout` が追加された。

### Cited Findings
- Aurora MySQL version 2 は MySQL 5.7 互換、version 3 は MySQL 8.0 互換、version 8.4 は MySQL 8.4 互換。バージョン番号 `3.04.1` の `3` がメジャー、`8.4.7` では `8.4` がメジャー。 — [Upgrading the major version of an Amazon Aurora MySQL DB cluster](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.MajorVersionUpgrade.html)
- Aurora MySQL version 8.4 は Aurora MySQL-Compatible Edition の最新メジャーで、MySQL 8.4 Community Edition 互換。version 3 の最新リリースと同じ Aurora 機能をサポートし、例外として Fast insert は 8.4 では使えない。バージョン形式は `major.minor`。例として `8.4.7` が 8.4 ファミリーの最初のマイナー。 — [Aurora MySQL version 8.4 compatible with MySQL 8.4](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL84.html)
- `version()` はコミュニティ版、`aurora_version()` / `@@aurora_version` は Aurora 版。version 2 と 3 は `major.minor.patch`（例 `3.05.2`）。version 8.4 以降は `major.minor` で、例の出力は `8.4.7`。 — [Amazon Aurora versioning](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.VersionPolicy.Versioning.html)
- リリースノートの 8.4 系統は次の 2 本だけが目次に載っている: Aurora MySQL 8.4.8（2026-09-03）、Aurora MySQL 8.4.7（2026-05-21）。 — [Database engine updates for Amazon Aurora MySQL version 8.4](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraMySQLReleaseNotes/AuroraMySQL.Updates.84Updates.html)
- 8.4.7 は MySQL 8.4.7 互換。リリース日 2026-05-21、標準サポート終了 2027-11-30。8.4.8 は MySQL 8.4.8 互換。リリース日 2026-09-03、標準サポート終了 2028-03-31。メジャー対応表では Community MySQL 8.4 = Aurora MySQL version 8.4、コミュニティ EOL と Aurora 標準サポート終了はどちらも 2032 年 4 月。MySQL 8.0 = Aurora MySQL version 3。 — [Release calendars for Amazon Aurora MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraMySQLReleaseNotes/AuroraMySQL.release-calendars.html)
- 8.4.7 の新機能として、クラスターパラメータグループで設定するパスワード管理、`aurora_enable_validate_password_component` による validate_password、`aurora_enable_memory_management`（既定 ON）による自動メモリ管理が書かれている。このページの取得スニペットに `aurora_transaction_timeout` はない。 — [Aurora MySQL 8.4.7, May 21, 2026](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraMySQLReleaseNotes/AuroraMySQL.Updates.847.html)
- 8.4.8 は MySQL 8.4.8 ベース。新機能は (1) マルチソース binlog レプリケーション、(2) 遅延 binlog レプリケーション、(3) `aurora_transaction_timeout`、(4) TLS 1.3 のポスト量子ハイブリッド鍵交換（X25519MLKEM768、SecP256r1MLKEM768）。改良は 8.4.7 との差分。version 3 のサポート対象クラスターから 8.4.8 へインプレースメジャーアップグレード、スナップショットリストア、Blue/Green が可能。 — [Aurora MySQL 8.4.8, September 3, 2026](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraMySQLReleaseNotes/AuroraMySQL.Updates.848.html)
- ドキュメント履歴も 8.4.8 を 2026-09-03、8.4.7 を 2026-05-21 としている。8.4.9 以降のリリースノート項目はこの履歴の取得範囲に無い。 — [Document history for the Aurora MySQL Release Notes](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraMySQLReleaseNotes/doc-history.html)
- 8.4.7 以上では、ヒストグラムの AUTO UPDATE を指定しても警告を出して MANUAL UPDATE として扱う。 — [Aurora MySQL version 8.4 compatible with MySQL 8.4](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL84.html)

### Inferences
- 「Aurora MySQL 3 の上に 8.4.7 互換がある」という整理は誤り。8.4 互換のエンジン文字列は `8.4.7` / `8.4.8` であり、`3.x` ではない。
- 2026-10-08 時点でリリースノート目次とカレンダーが示す 8.4 の最新パッチは 8.4.8。8.4.7 は一つ前のマイナーで、今も標準サポート期限内（2027-11-30 まで）。

### Gaps
- 「Checking Aurora MySQL version numbers」の取得スニペット（更新表示 2026-01-27）は、エンジン文字列を `8.0.mysql_aurora.3.04.0` のような version 2/3 形式だけ説明しており、8.4 の `major.minor` に更新されているかは未確認。8.4 の番号付けは Versioning ページと MySQL 8.4 ページを正とする。 — [Checking Aurora MySQL version numbers](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.Versions.html)
- 8.4.7 リリースノート本文は全文を取得していない。新機能 3 点は検索スニペット、トランザクションタイムアウト非搭載の決定的な根拠は次節のバージョンゲートである。
- 8.4.9 / 8.4.10 のリリースノートは見つかっていない。性能ページの検索スニペットに「8.4.10 で max_connections 上限 50,000」と出た箇所は、英語本文を開いた行と一致せず、採用しない。

## aurora_transaction_timeout は 8.4.7 にあるか。ロック待ち 120 秒とどう違うか

### Takeaway
`aurora_transaction_timeout` は Aurora 専用で、Aurora MySQL 8.4.8 以上にだけある。既定 0（無効）。有効にすると、待ち側の `innodb_lock_wait_timeout` とは別に、長時間トランザクション（実行中もアイドルも）を切って InnoDB purge を守る。8.4.7 ではこのパラメータはドキュメント上まだ存在しない。アイドルなブロッカーを止めるレバーであり、待ち側を長く待たせるレバーではない。

### Cited Findings
- パラメータは Aurora MySQL version 8.4.8 以上で利用できる。長いトランザクション（active または idle）が InnoDB purge をブロックするのを防ぐ。 — [Transaction timeout in Amazon Aurora MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.TransactionTimeout.html)
- 8.4.8 リリースノートが「Introduced the aurora_transaction_timeout parameter」と書き、用途は長いトランザクションが purge を塞ぐことの防止。 — [Aurora MySQL 8.4.8, September 3, 2026](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraMySQLReleaseNotes/AuroraMySQL.Updates.848.html)
- 属性: 名前 `aurora_transaction_timeout`。Scope は Session と Global。既定 0（無効）。単位は秒。Dynamic は Yes だが、新しい InnoDB トランザクションにだけ効く。クラスター、インスタンス、セッションのいずれかで設定できる。 — [Transaction timeout in Amazon Aurora MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.TransactionTimeout.html)
- 対象は DML を実行する InnoDB トランザクションで、読み取り専用も含む。暗黙コミット文は対象外。例外は `CREATE TABLE .. AS SELECT` と `LOAD DATA`。タイマーは最初の InnoDB 文の実行時に開始し、そのトランザクションの寿命のあいだ値は固定。`BEGIN` だけでは開始しない。`autocommit=0` のあとの最初の InnoDB 文で開始する。 — 同ページ
- 期限切れ時: 実行中クエリがあればそのクエリを中断してトランザクションをロールバックし、接続は使える。クエリが無くアイドルなら接続を切る。 — 同ページ
- 実行中に切れたクライアントエラーは `ERROR 63952 (40001): Transaction exceeded maximum allowed duration of <N> seconds and was rolled back.` アイドル時の次のクエリは MySQL の "server has gone away" と同じ。状態変数 `Aurora_transaction_timeouts`。performance_schema の `ER_AURORA_TRANSACTION_TIMEOUT_ERROR` は実行中タイムアウトだけが増やし、アイドル切断は増やさない。 — 同ページ
- 他のタイムアウトより優先して切る、という相互作用が明記されている。「設定した `aurora_transaction_timeout` より長く開いたトランザクションは、他のタイムアウト設定に関係なく終了する。」 — 同ページ
- XA PREPARED は対象外。write forwarding を含む文やトランザクションも対象外。転送トランザクションのアイドル制御は `aurora_fwd_writer_idle_timeout`。長いロールバックは元の変更より何倍もかかることがあり、プロセスを殺しても起動時にロールバックが再開する。 — 同ページ
- コミュニティ MySQL 8.4 の `innodb_lock_wait_timeout` は、行ロックを待つ秒数。既定 50。待ちが尽きた文はロールバックされるが、トランザクション全体はロールバックされない（`innodb_rollback_on_timeout` を起動時に付けた場合だけ全体）。テーブルロック待ちには適用されない。`innodb_deadlock_detect` が ON（既定）のときデッドロックには適用されず、InnoDB は即座に一方をロールバックする。エラーは `ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction`。Scope は Global と Session、Dynamic Yes、最小 1、最大 1073741824、単位秒。 — [MySQL 8.4 InnoDB Startup Options and System Variables](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html)

### Inferences
- 8.4.7 にこのパラメータは無い。バージョンゲートが「8.4.8 and higher」であり、導入が 8.4.8 の新機能だからである。8.4.7 でアイドルブロッカーを同じ仕組みでは切れない。
- `innodb_lock_wait_timeout=120` は待ち側の文が 120 秒で ERROR 1205 になるまでの時間を伸ばすだけである。ロックを持ったままアイドルなブロッカーの寿命は伸ばしも縮めもしない。
- 8.4.8 以上で `aurora_transaction_timeout` を 0 以外にすると、ブロッカー側（実行中もアイドルも）をトランザクション年齢で切る。アイドルなら接続ごと切れる。これは待ち時間を 120 秒に延ばす変更とは別レバーで、有効値が 120 未満なら、待ち側が 120 秒待つ前にブロッカーが消えることがある。既定 0 のままならこの介入は起きない。
- 両方とも「タイムアウト」だが、切る対象が逆である。lock wait は waiter の現在文、transaction timeout は blocker を含む開いた InnoDB トランザクション。

### Gaps
- `innodb_rollback_on_timeout` が Aurora のパラメータグループにあるかは未確認。コミュニティでは文だけ戻すのが既定で、全体ロールバックは別オプション。
- `aurora_transaction_timeout` と `innodb_lock_wait_timeout` を同時に変えたときの優先順位を、ロック待ち中の waiter について実験した記述はページに無い。ページが言うのは「開いたトランザクションは transaction timeout で他のタイムアウトに関係なく終了する」まで。
- 8.4.7 のパラメータグループに同名パラメータが残っていて無効、ということはドキュメントからは分からない。利用可能バージョンの文面は「8.4.8 and higher」だけである。

## innodb_lock_wait_timeout は Aurora で変更できるか。パラメータグループ、動的変更、SET、既定値

### Takeaway
AWS のパラメータ一覧ではインスタンスレベルの DB パラメータグループにあり、Modifiable は Yes、注記は None。クラスターパラメータグループの表には出てこない。ただしその一覧の前文は「特記なきパラメータは Aurora MySQL version 2 と 3 に有効」としており、8.4.7 向けの明示行は無い。既定 50 はコミュニティ MySQL 8.4 の値であり、Aurora がその既定を変えたという文は見つかっていない。永続化の正規手段はパラメータグループ。`SET PERSIST` を Aurora が拒否すると書かれた Aurora ユーザーガイドの文は見つかっていない。

### Cited Findings
- クラスター全体のパラメータは DB クラスターパラメータグループ、インスタンスだけのパラメータは DB パラメータグループ。クラスターレベルはインスタンスの DB パラメータグループには無い。 — [Aurora MySQL configuration parameters](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.ParameterGroups.html)
- 同ページ前文: 「Unless otherwise noted, parameters listed in the following tables are valid for Aurora MySQL versions 2 and 3.」同じページは 8.4 専用パラメータ（例: `aurora_enable_memory_management`、`authentication_policy`、`validate_password.*`、`aurora_enable_validate_password_component` は 8.4.7 以上）を、利用可能バージョンを注記して載せている。 — 同ページ
- インスタンスレベル節は同ページの Instance-level parameters（取得時に見出しがこの節の開始）。その表に `| innodb_lock_wait_timeout | Yes | None |`。クラスター表側の走査ではこの名前は一度だけ、インスタンス節の中でヒットした。 — 同ページ
- 別パラメータ `lock_wait_timeout` もインスタンス表に `| lock_wait_timeout | Yes | None |`。これはメタデータロック待ち用で、InnoDB 行ロック用の `innodb_lock_wait_timeout` とは別。コミュニティマニュアルも、行ロック用タイムアウトはテーブルロック待ちに適用されないと書く。 — [Aurora MySQL configuration parameters](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.ParameterGroups.html)、[MySQL 8.4 innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html)
- Aurora が適用しない MySQL パラメータの例（取得した一覧）: `innodb_adaptive_flushing`、`innodb_doublewrite`、`innodb_flush_method`、`innodb_io_capacity`、`innodb_log_file_size`、`innodb_redo_log_capacity`、`innodb_undo_log_truncate`、`innodb_undo_tablespaces` など。この抜粋に `innodb_lock_wait_timeout` は無い。一覧は網羅ではないとページが断っている。 — [Aurora MySQL 設定パラメータ（適用されないパラメータ節）](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.ParameterGroups.html)
- `wait_timeout` と `interactive_timeout` は Aurora では小さい方を、対話・非対話の両方のアイドルセッションを切るタイムアウトに使う。これはパラメータ表の注記。 — [Aurora MySQL configuration parameters](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.ParameterGroups.html)
- Serverless の一般則: 一覧に無いパラメータはプロビジョンドと同じ。既定はクラスターパラメータグループから継承。カスタムのクラスターグループでクラスター既定を、カスタムの DB パラメータグループで特定インスタンスを変えられる。Dynamic は即時、static は再起動後。 — [Performance and scaling for Aurora serverless](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html)
- Serverless がカスタム値を無視して自分で変える Aurora MySQL パラメータは `innodb_buffer_pool_size`、`innodb_purge_threads`、`table_definition_cache`、`table_open_cache`。この 4 つの中に `innodb_lock_wait_timeout` は無い。 — 同ページ
- コミュニティ MySQL 8.4: Default 50、Dynamic Yes、Scope Global と Session、最小 1、最大 1073741824 秒。 — [MySQL 8.4 innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html)
- RDS for MySQL（Aurora ではない）の非サポート機能一覧に「Persisted system variables」がある。 — [MySQL feature support on Amazon RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/MySQL.Concepts.FeatureSupport.html)

### Inferences
- 変更単位はクラスターパラメータグループではなく、インスタンスの DB パラメータグループ。ライターとリーダーで別値にできる形。行ロック待ちが問題になるのはライターなので、ライターの DB パラメータグループを変えるのが該当箇所になる（リーダー側を変えてもライターの DML ロック待ちには効かない。次々節）。
- 表の Modifiable=Yes と、Serverless が上書きする 4 パラメータに入っていないことから、120 への変更はパラメータとして受け付けられる種類である。Apply type が dynamic か static かは AWS のこの行には書かれていない。コミュニティでは Dynamic Yes なので、Aurora でも `SET GLOBAL` / `SET SESSION` とパラメータグループの即時適用が通る可能性が高いが、それは推論であり AWS の ApplyMethod 記載ではない。
- 再起動をまたいで残す正規の場所はパラメータグループ。`SET GLOBAL` はコミュニティ仕様上そのプロセスのグローバル値を変えるが、パラメータグループに書かない限り再起動で戻る、という RDS/Aurora 一般の運用と整合する。ただしこの文面そのものを Aurora の `innodb_lock_wait_timeout` 行では確認していない。
- 既定が 50 のまま、と Aurora が明言したページは無い。パラメータ表は既定値欄を持たず、注記 None は「Aurora 固有の上書き説明が無い」という意味に読める。コミュニティ既定 50 をそのまま使う、が最も薄い仮定。

### Gaps
- 8.4.7 / 8.4.8 のパラメータグループで `innodb_lock_wait_timeout` の Source、ApplyType、許可レンジ、Engine default を `describe-db-parameters` で確認していない。一覧の有効バージョン文が「2 and 3」のままなので、8.4 で同名パラメータがインスタンスグループにあることは強く示唆されるが、8.4 専用の一文は無い。
- Aurora ユーザーガイドに `SET PERSIST` / `SET PERSIST_ONLY` を禁止する文は見つからなかった。RDS for MySQL の「Persisted system variables 非サポート」を Aurora に転用しない。
- `USER_WorkingWithParamGroups` 本文は、dynamic / `SET` / reboot のパターン検索で該当行を得られなかった。dynamic と static の即時・再起動の区別は Serverless のパラメータ節で確認した。
- Aurora 上の `wait_timeout` / `interactive_timeout` の既定秒数は未確認。Aurora は両者の最小値で全アイドルセッションを切るが、コミュニティ既定が 28800 秒であることの Aurora での確認はしていない。120 秒の行ロック待ちよりこちらのアイドル切断が先に来る、とは現時点の資料だけでは言えない。

## 待機イベントと、AWS が示す見方

### Takeaway
行ロック待ちの説明ページはエンジン版の注記が古い。`synch/cond/innodb/row_lock_wait` は「Aurora MySQL version 3」のみ、`row_lock_wait_cond` と `aurora_lock_thread_slot_futex` は「version 2」のみと書いている。8.4 をサポート対象に含める文は無い。一方 8.4.8 のリリースノートは `performance_schema.data_lock_waits` と `data_locks` に言及しており、ロック待ちの情報源自体は 8.4.8 に存在する。トラブルシュートの型は Performance Insights、`innodb_lock_wait_timeout` の確認、`SHOW ENGINE INNODB STATUS`、ブロッカーが idle か active か、アイドルならセッション終了、である。

### Cited Findings
- `synch/cond/innodb/row_lock_wait`: あるセッションが更新用に行をロックし、別セッションが同じ行を更新しようとすると起きる。サポート対象エンジンは「Aurora MySQL version 3」とだけ書かれている。増加原因は複数 DML が同じ行に同時アクセス。対処は Performance Insights で SQL を特定、楽観ロック、複数行文、パーティション、`innodb_lock_wait_timeout` の確認（タイムアウトエラーまでの待ち時間）。ブロッカーが idle か active か、アプリ由来か利用者由来かを見る。特定は `SHOW ENGINE INNODB STATUS`。`performance_schema.data_lock_waits` と `data_locks`、`information_schema.innodb_trx`、`processlist` の結合例がある。サンプルの blocker_state は `User sleep`。アイドルならセッション終了を検討し、長いロールバックがあり得ると書く。 — [synch/cond/innodb/row_lock_wait](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/ams-waits.row-lock-wait.html)
- `synch/cond/innodb/row_lock_wait_cond`: 説明は同様。サポート対象は「Aurora MySQL version 2」。参照先 InnoDB locking は MySQL 5.7 の URL。同じく `innodb_lock_wait_timeout` と、idle/active の切り分け、アイドルならセッション終了。 — [synch/cond/innodb/row_lock_wait_cond](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/ams-waits.row-lock-wait-cond.html)
- 待機イベント一覧は `synch/mutex/innodb/aurora_lock_thread_slot_futex` を「複数 DML が同じ行に同時アクセス」と説明する。 — [Aurora MySQL wait events](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.Waitevents.html)
- 日本語版の futex 詳細ページは、サポート対象エンジンを「Aurora MySQL バージョン 2」と書く。対処に `innodb_lock_wait_timeout` と `SHOW ENGINE INNODB STATUS` がある。 — [synch/mutex/innodb/aurora_lock_thread_slot_futex（日本語）](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/ams-waits.waitsynch.html)
- デッドロック検出が ON（既定）のとき InnoDB は即座に検出し、挿入・更新・削除行数が小さいトランザクションをロールバックしようとする。監視は `SHOW ENGINE INNODB STATUS`（再起動以降の直近 1 件）、`innodb_print_all_deadlocks`（DB クラスターパラメータ。ON でエラーログへ全件）、CloudWatch `Deadlocks`、Performance Insights の `innodb_deadlocks` と `innodb_lock_wait_timeout`。 — [Minimizing and troubleshooting Aurora MySQL deadlocks](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.BestPractices.FeatureRecommendations.html)
- 8.4.8 は、`performance_schema.data_lock_waits` または `data_locks` へのクエリが、`ALGORITHM=INSTANT` で列追加されたテーブルへの `ALTER TABLE ... REORGANIZE PARTITION` と同時だとインスタンス再起動を起こし得た不具合を修正している。この記述は、8.4.8 にこれらの performance_schema 表があることを示す。 — [Aurora MySQL 8.4.8, September 3, 2026](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraMySQLReleaseNotes/AuroraMySQL.Updates.848.html)
- セッション終了手順は待機イベントページから「Ending a session or query」へリンクされている。 — [synch/cond/innodb/row_lock_wait](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/ams-waits.row-lock-wait.html)

### Inferences
- 8.4.7 のロック待ちを Performance Insights で見るとき、イベント名 `synch/cond/innodb/row_lock_wait` が出ても、AWS の解説ページは対象を version 3 としか書いていない。手順（PI、タイムアウト値、INNODB STATUS、idle ブロッカーの終了、`data_lock_waits`）は 8.0 系の InnoDB と同じ型だが、ページのサポート対象行を 8.4 に読み替える公式更新は確認していない。
- `row_lock_wait_cond` と `aurora_lock_thread_slot_futex` の詳細ページを 8.4 の根拠に使わない。版の注記が version 2 のままである。
- Performance Insights のカウンタ名 `innodb_lock_wait_timeout` は「パラメータ値」ではなく、ロック待ちタイムアウトが起きた回数側のメトリクスとしてデッドロック節に並んでいる。パラメータを 120 にすると、このカウンタの増え方は遅くなる方向であり、ブロッカーの保持時間は測れない。

### Gaps
- 英語版 `ams-waits.waitsynch.html` 本文は未取得。futex の「version 2 のみ」は日本語版ページによる。
- 8.4.7 の Performance Insights に `synch/cond/innodb/row_lock_wait` が出ることは、待機イベントページが保証していない。8.4.8 が `data_lock_waits` に言及しているのは 8.4.8 であり、8.4.7 での同表の存在はリリースノート上は未確認（コミュニティ 8.4 の機能としては 8.0 からある表である）。
- `innodb_print_all_deadlocks` が 8.4 のクラスターパラメータとして残るかは、デッドロック節がそう呼んでいるだけで、8.4 パラメータ表の行は見ていない。

## Serverless v2 のスケールと、ロック待ちセッション

### Takeaway
スケール判定にドキュメントが名指しする負荷は CPU、メモリ、ネットワークである。接続数そのものはスケール条件として書かれていない。各 ACU は約 2 GiB のメモリと対応する CPU・ネットワーク。スケールは接続が開いたまま、トランザクション実行中、テーブルがロックされたままでも起き、進行中の操作を切らない。ロック待ちセッションは条件変数で寝ているので CPU は低い、と AWS は書いていない。接続は 1 本ずつメモリを使い、`max_connections` は最大 ACU から計算した値に固定で、スケールダウンでは接続を切らない。120 秒への延長は待ちセッションの生存を延ばし、接続数とメモリには効きうるが、CPU スケールの主因にはなりにくい、という推論になる。

### Cited Findings
- 容量の単位は ACU。各 ACU は約 2 GiB のメモリと、対応する CPU とネットワーキング。容量は 0〜256 ACU、増分 0.5。最小 0 はワークロードが無いとき 0 まで縮む。ストレージ容量とコンピュートは別で、スケールするのはコンピュート。CPU とメモリが低くてもデータはテラバイト級であり得る。 — [How Aurora serverless works](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html)
- ライターまたはリーダーごとに CPU、メモリ、ネットワークの利用率を継続的に見る。これらをまとめて load と呼ぶ。load にはアプリの DB 操作に加え、サーバのバックグラウンドと Aurora の管理処理が含まれる。これらのいずれかで容量が足りなければスケールアップする。性能問題を解消できるときもスケールアップする。 — 同ページ
- ライターが DDL と DML（`INSERT`、`UPDATE` など）を処理する。リーダーは `SELECT` のような読み取り専用を処理する。 — 同ページ
- スケールは 0.5 ACU 刻みから。現在容量が大きいほど増分も大きく、速く増える。コンソールに離散イベントは出さず、`ServerlessDatabaseCapacity` と `ACUUtilization` で見る。 — 同ページ
- 昇格ティア 0 と 1 のリーダーは最小容量が現在のライター容量に縛られ、最大はクラスターの最大 ACU。フェイルオーバーでライター負荷を受けられる大きさにする。ティア 2〜15 は独立にスケールする。 — 同ページ
- 「Aurora serverless scaling can happen while database connections are open, while SQL transactions are in process, while tables are locked, and while temporary tables are in use.」静かな点を待たない。進行中の DB 操作はスケールで中断されない。 — 同ページ
- 容量表（エンジン）: 0.5–128 ACU は Aurora MySQL 3.02.0 以上、0.5–256 は 3.06.0 以上、0–256 は 3.08.0 以上。表のセルに 8.4 という文字は無い。 — 同ページ
- スケールアップの多くはメモリ使用と CPU。`TempStorageIOPS` と `TempStorageThroughput` は、ローカルストレージとの転送が稀に容量増の原因になるときの切り分け。`ACUUtilization` が 100 に近いと最大 ACU に張り付いている。Serverless の CPU メトリクスは、使用 CPU を最大 ACU で使える CPU で割った割合で、高い割合が続くとスケールアップする。 — [Performance and scaling for Aurora serverless](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html)
- 監視すべきインスタンスメトリクスとして `ServerlessDatabaseCapacity`、`ACUUtilization` に加え、標準の `DatabaseConnections` と `Queries` も見られる、と書いている。`DatabaseConnections` をスケールの入力とは書いていない。 — 同ページ
- アイドル接続でもメモリはゼロではない。Aurora MySQL は同じ接続数でも Community よりメモリが少ないことが多い、と AWS の規範ガイダンスが書く。Performance Schema は `max_connections` に応じてメモリを確保し、Performance Insights を有効にすると Performance Schema が有効になる。 — [Managing connections（Prescriptive Guidance）](https://docs.aws.amazon.com/prescriptive-guidance/latest/hyperscale-aurora-mysql/manage-connections.html)
- ほとんどがアイドルな接続プールの性能影響は限定的。同時にアクティブな接続が増えると、接続あたりスループットが下がりレイテンシが上がる。ピークの同時アクティブ接続数で容量を見積もる。 — [Managing performance and scaling for Amazon Aurora MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Performance.html)
- DBA ハンドブックは、Community MySQL が接続ごとに OS スレッドを割り当てるのに対し、Aurora はスレッドプールでワーカーを多重化すると書く。アイドル、ユーザ入力待ち、I/O 待ちのあいだワーカーは別接続へ移れる。スレッドプールは自動で増減する。ハンドブックはこの説明に 8.4 や 8.0 の版ゲートを書いていない。 — [Connection handling in Aurora MySQL and MySQL](https://docs.aws.amazon.com/whitepapers/latest/amazon-aurora-mysql-db-admin-handbook/connection-handling-in-aurora-mysql-and-mysql.html)
- Serverless v2 のメモリ逼迫では、まず ACU を上げる。スケール中も逼迫が続けば既存接続を切ることがある。メモリ確保しようとする接続の切断は、最大 ACU に達してそれ以上スケールできないときだけ。`aurora_enable_memory_management` は Aurora MySQL 8.4 のみ。ON（既定）だと Aurora が回復動作を管理し、`aurora_oom_response` は無視される。 — [Troubleshooting out-of-memory issues for Aurora MySQL databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQLOOM.html)
- 8.4.8 は、Serverless のスケール中に重要データ構造のリサイズが遅れると RDS ヘルスチェックがインスタンスを再起動し得た問題を修正している。 — [Aurora MySQL 8.4.8, September 3, 2026](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraMySQLReleaseNotes/AuroraMySQL.Updates.848.html)
- パラメータグループ族の表は Aurora MySQL version 3 = `aurora-mysql8.0` / `default.aurora-mysql8.0` だけで、「Those are the Aurora MySQL and Aurora PostgreSQL versions that you use with Aurora serverless.」と書く。8.4 の族名は無い。 — [Performance and scaling for Aurora serverless](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html)
- 別ページは、8.4 は version 3 最新と同じ Aurora 機能を持ち、例外は Fast insert のみ、と書く。Serverless を例外に含めていない。 — [Aurora MySQL version 8.4 compatible with MySQL 8.4](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.MySQL84.html)

### Inferences
- ロック待ちは `synch/cond/innodb/row_lock_wait` という条件待ちであり、その間クエリは実行していない。AWS は「ロック待ちは CPU を使わない」とは書いていない。ハンドブックが I/O 待ちやアイドルでワーカーを手放すと書くことと合わせると、待ちセッションの CPU は低く、スレッドプールなら待ちの本数だけ OS スレッドが増える Community よりさらに CPU は出にくい、が推論である。ハンドブックが 8.4.7 のスレッドプールを保証しているわけではない。
- 接続数はスケール入力として列挙されていない。ただし各接続はメモリを使い、メモリはスケール入力である。待ちが 50 秒から 120 秒になると、同じ到着率なら同時に滞留する待ち接続は最大で 120/50 = 2.4 倍になりうる。それがメモリを押し上げればスケールアップ、またはスケールインを遅らせることはあり得る。CPU だけでは起きない。これは計算上の推論で、AWS の測定値ではない。
- ブロッカーが重い DML なら、その CPU とメモリでライターはスケールアップしうる。待ち側を 120 秒にしてもブロッカーの重さは変わらない。
- スケールはロック中でも起き、待ちを中断しない。したがって 120 秒待ちの途中で ACU が変わっても、そのことだけでは ERROR 1205 にならない。逆に、待ちを伸ばしたからスケールが止まる、とも書かれていない。
- ドキュメント同士の食い違い: 機能同等（例外は Fast insert）なら 8.4 でも Serverless v2 がある、と読める。一方 Serverless の容量表とパラメータ族は version 3 と `aurora-mysql8.0` しか書いていない。8.4.7 の Serverless v2 をどちらか一方だけで断定しない。8.4.8 のリリースノートが「Aurora serverless scaling operations」の不具合修正を書いているので、8.4.8 時点では Serverless スケール処理がエンジン内にある。

### Gaps
- 8.4.7 の Serverless v2 対応、最小 0 ACU（auto-pause）の可否、パラメータグループ族の名前（`aurora-mysql8.4` か否か）は、取得した Serverless ページが version 3 としか書いていないため未確認。
- ロック待ち 1 セッションあたりのメモリ（スレッドスタック、Performance Schema、ネットバッファ）の数値は AWS ドキュメントから取っていない。
- 接続数がスケールインを遅らせる、という直接の文は無い。メモリ経由の推論にとどめる。

## 共有ストレージ。行ロックはライターの問題か

### Takeaway
クラスターボリュームはライターもリーダーも同じである。データ変更はプライマリ（ライター）だけが行う。リーダーは同じボリュームに接続するが読み取り専用。Serverless の説明でも DML はライター、読み取り SELECT はリーダー。行ロックを取る更新系の待ちはライター上で起き、リーダーを足してもその行ロック競合は減らない。

### Cited Findings
- DB クラスターは 1 つ以上の DB インスタンスと、それらのデータを管理するクラスターボリュームからなる。クラスターボリュームは複数 AZ にまたがる仮想ストレージで、各 AZ にデータのコピーがある。プライマリ（ライター）は読み書きを行い、クラスターボリュームへのデータ変更をすべて行う。クラスターにプライマリは 1 つ。Aurora Replica（リーダー）はプライマリと同じストレージボリュームに接続するが、読み取りだけ。レプリカはプライマリに加えて最大 15。この説明はプロビジョンド、Global Database、Aurora Serverless、Aurora MySQL、Aurora PostgreSQL のすべてに当てはまる、と注記がある。 — [Amazon Aurora DB clusters](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.html)
- Serverless: ライターがすべての DDL とすべての DML を処理する。リーダーは読み取り専用文（`SELECT`）を処理する。読み取り容量が足りなければリーダーを追加し、リーダーエンドポイントで読み取りセッションを分散してライターの負荷を下げる。 — [How Aurora serverless works](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html)
- リーダーは常に読み取り専用。Aurora MySQL はすべてのリーダーで `innodb_read_only=1` にする、というパラメータ注記（`read_only` の説明、取得箇所は version 2 の文脈を含む）。 — [Aurora MySQL configuration parameters](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.ParameterGroups.html)
- コミュニティ InnoDB の一貫性読み取りは undo で古い行を再構成し、ロックを避ける。REPEATABLE READ のスナップショットは最初の読み取り時点。これは移行プレイブックが Aurora MySQL のトランザクションモデルとして書いている。 — [Oracle Transaction Model and MySQL Transactions](https://docs.aws.amazon.com/dms/latest/oracle-to-aurora-mysql-migration-playbook/chap-oracle-aurora-mysql.sql.transactions.html)
- write forwarding を含むトランザクションは `aurora_transaction_timeout` の対象外。 — [Transaction timeout in Amazon Aurora MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.TransactionTimeout.html)
- 8.4.8 は、転送クエリがキャンセルまたはタイムアウトしたときにリーダーの write forwarding が止まりリーダー再起動が必要だった問題、フェイルオーバー後に転送のスループットとレイテンシメトリクスが 0 のままだった問題を修正している。 — [Aurora MySQL 8.4.8, September 3, 2026](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraMySQLReleaseNotes/AuroraMySQL.Updates.848.html)

### Inferences
- プレーンな一貫性読み取り（ロックを取らない `SELECT`）はリーダーへ逃がせる。`UPDATE` / `DELETE` / `SELECT ... FOR UPDATE` のような行ロックはライターで取られる。リーダー追加は行ロック待ちの解消にならない。
- `innodb_lock_wait_timeout` をリーダーのパラメータグループだけ変えても、ライターで待っている DML には効かない。
- リーダーからの書き込み転送を使っている場合、ロックを取るのは転送先のライターである。8.4.8 以上でも、転送を含むトランザクションは `aurora_transaction_timeout` では切れない。

### Gaps
- リーダー上の `SELECT ... FOR UPDATE` が即エラーになるか、write forwarding でライターへ送られるかは、今回取得したページには手順として無い。`innodb_read_only=1` の注記は version 2 の `read_only` 説明に付いており、8.4.7 での文言確認はしていない。
- リーダーの一貫性読み取りがライターの行ロックと待合しない、という文を Aurora が「row lock」という言葉で書いたページは取得していない。根拠は「変更はライターだけ」「リーダーは読み取り専用」「InnoDB の一貫性読み取りは undo」の組み合わせである。

## フェイルオーバーと 120 秒の待ち

### Takeaway
レプリカがあるときの復旧は通常 60 秒未満、しばしば 30 秒未満。そのあいだ読み書きは例外で失敗する。120 秒のロック待ちは、このフェイルオーバー窓より長い。待ちの途中でフェイルオーバーが起きれば、そのセッションの読み書きは失敗する。Aurora MySQL はフェイルオーバー時、ライターと昇格先リーダーだけを再起動し、他のリーダーはリーダーエンドポイントで読み取りを続けられる。

### Cited Findings
- プライマリ障害時、レプリカがあればそれを昇格する。無い場合は新しいプライマリを作る。レプリカ昇格の失敗イベントでは短い中断があり、読み書きは例外で失敗する。サービスは通常 60 秒未満、しばしば 30 秒未満で復旧する。レプリカが無く新しいプライマリを作る場合、復旧は通常 10 分未満。 — [High availability for Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)
- Aurora MySQL では、クラスターにリーダーが複数あるとフェイルオーバー中の可用性が上がる。再起動するのはライターと、フェイルオーバー先のリーダーだけ。他のリーダーはリーダーエンドポイント経由のクエリを続けられる。 — 同ページ
- DBA ハンドブック: プライマリ障害を検知して新しいプライマリへフェイルオーバーするのは通常 30 秒以内。アプリは手動なしで変更を認識する設計にする。DNS ゾーンの TTL は 5 秒。クライアントや JVM が DNS を長くキャッシュすると、フェイルオーバー後も古いインスタンスへ書き続ける。 — [Designing for fault tolerance and quick recovery](https://docs.aws.amazon.com/whitepapers/latest/amazon-aurora-mysql-db-admin-handbook/designing-for-fault-tolerance-and-quick-recovery.html)、[DNS caching](https://docs.aws.amazon.com/whitepapers/latest/amazon-aurora-mysql-db-admin-handbook/dns-caching.html)
- メンテナンス更新はクラスター内の全インスタンスを同時に再起動し、20〜30 秒のダウンタイムのあと再開できる、とリリースノートの案内が書く。 — [Release notes for Amazon Aurora MySQL-Compatible Edition](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraMySQLReleaseNotes/Welcome.html)
- バイナリログを有効にすると計画外再起動のリカバリが延びる。Aurora クラスター内の複製と PITR にバイナリログは不要。外部複製が不要なら `binlog_format=OFF` を推奨。 — [Amazon Aurora reliability](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Reliability.html)
- Serverless のホスト移動は稀で、そのとき接続は Aurora が管理する。通常のスケールは同じホスト上。 — [How Aurora serverless works](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html)

### Inferences
- 50 秒待ちでもフェイルオーバー（しばしば 30 秒未満、通常 60 秒未満）と重なり得る。120 秒にすると重なる時間の上限が伸び、待ちの途中で接続が例外失敗する機会が増える。失敗した待ちは ERROR 1205 ではなく接続エラーとしてアプリに見えるので、リトライが必要になる。これは時間の長さの比較からの推論で、「120 秒にするとフェイルオーバーが失敗する」ではない。
- フェイルオーバーで再起動されない他リーダーの読み取りは続く。行ロック待ちはライター側なので、昇格が終わるまで待ちセッションは新しいライターへ自動では続かない（接続が例外で失敗する、という HA ページの記述に従う）。
- DNS TTL 5 秒よりアプリの DNS キャッシュが長いと、復旧時間よりアプリ停止が長くなる、というハンドブックの注意は 120 秒待ちとは独立だが、リトライ設計の一部である。

### Gaps
- フェイルオーバー瞬間に `innodb_lock_wait_timeout` のカウントがリセットされるか、待ちが新しいライターで再開されるかは未記載。
- 「tens of seconds」という英語の定訳ではなく、ページの数値は「often less than 30 seconds」「typically less than 60 seconds」である。レプリカが無い場合は「typically less than 10 minutes」なので、120 秒待ちはこちらの復旧より短いこともある。
- 8.4.7 のフェイルオーバー時間が version 3 と違う、という記載は無い。上記は Aurora 全般の HA ページである。

## 長いトランザクション、purge、履歴リスト。Backtrack、zero-ETL、Blue/Green、Global Database

### Takeaway
`innodb_lock_wait_timeout` は待ち側の上限であり、ブロッカーがロックと undo を持ち続ける時間を縮めない。待ちを 120 秒にすると、その背後に滞留するセッションが増え、それらのセッションが自分でもロックや読み取りビューを持っていれば purge の遅れに間接的に寄与しうる。purge を直接守る Aurora 専用の仕組みは 8.4.8 以上の `aurora_transaction_timeout` である。履歴リストはクラスター共有ストレージ上でクラスター全体の問題になる。Backtrack、zero-ETL、Blue/Green、Global Database を `innodb_lock_wait_timeout` に結びつけた文は見つかっていない。

### Cited Findings
- InnoDB 履歴リストはコミット済みトランザクションの undo ログのグローバルリスト。履歴リスト長は、変更を含む undo ログの本数。長くなりすぎると古い行バージョンが多く、クエリと DB シャットダウンが遅くなる。長い履歴リストはシャットダウンを遅くするため、メジャーバージョンアップグレードのようにシャットダウンを伴う操作の前に短くする。 — [The InnoDB history list length increased significantly](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/proactive-insights.history-list.html)
- 長いトランザクションは `information_schema.innodb_trx` と `processlist` を結合し、開始からの秒数が 10 を超えるものを見る。注記: リードレプリカ上の長いトランザクションも見ること。 — 同ページ
- CloudWatch `RollbackSegmentHistoryListLength` は、削除マーク付きレコードを持つコミット済みトランザクションの undo を測る。`trx_rseg_history_len` と同じ値。purge 予定のレコードである。 — 同ページ
- ハンドブック: 開いたトランザクションはガベージコレクションを塞ぐ。極端だとストレージ増、CPU 増、クエリ遅延。`INNODB_TRX.TRX_STARTED` で年齢を見る。数分以上は調査対象。`trx_rseg_history_len` が数万だと GC がやや遅れ、数百万から数千万は危険。Aurora では全 DB インスタンスが同じストレージボリュームを使うので、ガベージコレクションはインスタンス単位ではなくクラスター全体。この推奨は Aurora 固有ではなく InnoDB 全般にも当てはまる、とハンドブックが断っている。autocommit 無効の常時トランザクションは非推奨。 — [Transaction management and autocommit](https://docs.aws.amazon.com/whitepapers/latest/amazon-aurora-mysql-db-admin-handbook/transaction-management-and-autocommit.html)
- `aurora_transaction_timeout` の存在理由は、active または idle の長時間トランザクションが InnoDB purge をブロックすることの防止。8.4.8 以上、既定 0。 — [Transaction timeout in Amazon Aurora MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.TransactionTimeout.html)
- コミュニティの lock wait timeout は待ち側の行ロック待ちであり、ロックを持っている側の寿命ではない。タイムアウト時に戻るのは現在の文。 — [MySQL 8.4 innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html)
- Global Database の計画スイッチオーバー: 対象エンジン（Aurora MySQL 3.09 / MySQL 8.0.40 以上、または記載の PostgreSQL）では読み書きの回復が通常 30 秒未満。8.4 はこの文に無い。 — [Using switchover or failover in Amazon Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html)
- 8.4.8 は Global Database スイッチオーバー中、一時テーブル掃除でライターが再起動しスイッチオーバー完了が延び得た問題を修正している。したがって 8.4.8 に Global Database スイッチオーバー処理はある。ロック待ちタイムアウトとは結びつけていない。 — [Aurora MySQL 8.4.8, September 3, 2026](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraMySQLReleaseNotes/AuroraMySQL.Updates.848.html)
- 8.4 へのアップグレード手段として Blue/Green が書かれている。ロック待ちとは結びつけていない。 — [Upgrading to Aurora MySQL version 8.4](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.mysql84-upgrade-procedure.html)、[Aurora MySQL 8.4.8, September 3, 2026](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraMySQLReleaseNotes/AuroraMySQL.Updates.848.html)
- Backtrack はクラスターを指定時刻へ巻き戻す。ウィンドウ上限 72 時間。クラスター全体が対象で、単一テーブルや単一更新だけの巻き戻しはできない。binlog レプリケーションとは併用できない。ロック待ちタイムアウトとの関係はプレイブックに無い。 — [Amazon Aurora Backtrack](https://docs.aws.amazon.com/dms/latest/oracle-to-aurora-mysql-migration-playbook/chap-oracle-aurora-mysql.tools.aurorabacktrack.html)

### Inferences
- 待ちタイムアウトを 50 から 120 に上げても、アイドルブロッカーが持つ undo の寿命は縮まない。ブロッカーが開いているあいだ履歴リストは伸びうる。これはコミュニティ InnoDB でも Aurora でも同じ。
- Aurora 固有なのは、undo / 履歴がクラスターボリューム上で共有され、リーダーの長い読み取りトランザクションも同じ purge を止めうること。待ち側をライターに滞留させると、そのセッションが既に行を変更している、または読み取りビューを持っているなら、ブロッカーに加えて待ち側も purge 境界を押しうる。まだ何も変更していない純粋な待ちだけなら、そのセッション自身の undo は小さい。ドキュメントはこの切り分けを `innodb_lock_wait_timeout` の言葉では書いていない。
- 8.4.7 では purge 保護の `aurora_transaction_timeout` が無い。長いブロッカー対策は、トランザクションを短くする、アイドルを `wait_timeout` 系やアプリで切る、8.4.8 へ上げてから transaction timeout を使う、のいずれかになる。`wait_timeout` の既定秒数は未確認。
- 120 秒は履歴リストの「数分以上で調査」より短い。単発の 120 秒待ちそのものが履歴リストを危険域（数百万）にする、とは言えない。問題はブロッカーが何分も開いたままであることと、その後ろに鎖状のロック持ちセッションが増えることである。

### Gaps
- zero-ETL とロック待ち、または長いトランザクションを結びつけた AWS ページは見つかっていない。
- Backtrack の実ウィンドウが長いトランザクションや undo で縮む、という記述は今回の取得範囲に無い。プレイブックは「極端に重いワークロードだと実際のウィンドウが目標より短くなる」とだけ書く。
- Blue/Green スイッチオーバー所要時間と、進行中の行ロック待ちの関係は未記載。
- `PurgeBoundary` の対象バージョンは英語ページの取得範囲では確認していない。中国語版は Aurora MySQL 2.11 以上と 3.08 以上、ドイツ語版は別の版番号を書いており、矛盾するため英語で再確認するまで採用しない。
- 8.4 の履歴リストメトリクス名が version 3 と同じかは未確認。

## max_connections と ACU

### Takeaway
Serverless の `max_connections` 既定は、現在の ACU ではなく最大 ACU から換算したメモリサイズに公式を適用した値で、スケールダウンしても接続が切れないよう一定に保つ。最大 ACU を変えたあとその値を更新するには再起動が要る、と本文は書き、static だとしている。閉じた数式はページに印刷されていない。log 底 2 の `DBInstanceClassMemory`（バイト）で、整数に切り捨て、という説明と、最大 ACU ごとの既定表がある。1 ACU で Aurora MySQL は 90、8 ACU で 1,000、256 ACU で 6,000。

### Cited Findings
- Serverless は `max_connections` を一定に保ち、スケールダウンで接続を切らない。既定は DB インスタンスのメモリサイズに基づく公式。公式とプロビジョンドのクラス別既定は「Maximum connections to an Aurora MySQL DB instance」を見よ、とリンクしている。 — [Performance and scaling for Aurora serverless](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html)
- 公式を評価するとき使うメモリは、現在 ACU ではなく最大 ACU に基づくメモリサイズ。既定を変えるなら定数ではなく公式の変形を推奨し、最大容量に応じた値を Aurora が使えるようにする。最大容量を変えたら Serverless インスタンスの再起動が必要。理由は Serverless では `max_connections` が static だから。 — 同ページ
- 最大 ACU と Aurora MySQL の既定最大接続（表）: 1→90、4→135、8→1,000、16→2,000、32→3,000、64→4,000、128→5,000、192→6,000、256→6,000。PostgreSQL で最小 0 または 0.5 ACU のとき 2,000 キャップ、という注記は PostgreSQL 向けで、MySQL の行には同じキャップは書かれていない。 — 同ページ
- 別の例では、メモリが倍になると `max_connections` 公式が 1,000 増える、と書く。カスタム DB パラメータグループで `max_connections` を直接設定した場合は再起動不要、という例の注記もある。本文の「最大容量変更後は再起動」と並記されている。 — 同ページ
- プロビジョンド側: 計算は log 底 2（自然対数ではない）と、インスタンスクラスの `DBInstanceClassMemory`（バイト）。パラメータは整数のみで小数は切り捨て。大きい R3/R4/R5 ではメモリ倍増ごとに接続 +1000、T2/T3 のメモリ違いでは +45。例: db.r6g.large は公式が 1069.2 でも実装は 1000。Aurora MySQL と RDS for MySQL はメモリオーバーヘッドが違い、同じインスタンスクラスでも `max_connections` は違いうる。表の値は Aurora MySQL のみ。T2/T3 の低い上限は開発・テスト向け。 — [Managing performance and scaling for Amazon Aurora MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Performance.html)
- クラス例: db.t3.medium と db.t4g.medium は 90、db.t3.large と db.t4g.large は 135、db.r6g.large / db.r8g.large は 1000、db.r5.xlarge などは 2000、db.r8g.48xlarge は 8000、db.x2g.large は 2000。 — 同ページ
- 各 ACU は約 2 GiB メモリ。 — [How Aurora serverless works](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html)
- 接続上限超過は Error 1040 Too many connections。`max_connections` はクラスターおよびインスタンスのパラメータグループにある、と接続トラブルシュートが書く。 — [Troubleshooting connection issues for Aurora MySQL databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/mysql-troubleshooting-dbconn.html)
- バッファプールなど他の大メモリ消費者を既定から変えたら、接続上限も見直す。 — [Managing performance and scaling for Amazon Aurora MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Performance.html)

### Inferences
- 閉じた式 `max_connections = f(memory)` の代数表現は、取得した英語ページには無い。あるのは log2(`DBInstanceClassMemory`) 、小数切り捨て、R 系はメモリ倍で +1000、T 系は +45、という説明と実装が丸める例（1069.2→1000）である。Serverless 表はこれと整合的に読める（約 2 GiB×1 ACU で 90、×4 で 135、×8 で 1000、以後メモリ倍で +1000、256 ACU で 6000 に頭打ち）が、AWS がその対応を説明しているわけではない。表の数値だけが根拠である。
- ロック待ちを 120 秒にしても `max_connections` の枠は増えない。滞留接続が増えて 1040 に近づく方向である。枠は最大 ACU で決まるので、現在 ACU が小さくても接続上限は最大 ACU 側のままである。
- 「カスタムグループに直接書けば再起動不要」と「static なので最大 ACU 変更後は再起動」は対象が違う可能性がある（ユーザが値を固定した場合と、Aurora が最大 ACU から再計算する場合）。両方引用し、8.4.7 でどちらが勝つかは未検証とする。

### Gaps
- log2 公式の完全な式（係数、定数項、`DBInstanceClassMemory` から何を引くか）は未取得。`USER_ParamValuesRef` は開いていない。
- 絶対上限（16,000 や 50,000 など）は、英語の性能ページを開いた範囲には無かった。検索スニペットの「8.4.10 で 50,000、それ以外は 16,000」は本文の行と一致しなかったので不採用。中国語ミラーの「最大 16,000」も英語で再確認していない。
- 8.4.7 Serverless の `max_connections` 表が上の ACU 表と同一かは、表がエンジン版で分かれていないため未確認。表は「Aurora MySQL」とだけ書いている。

## トラブルシュートとして AWS が推奨していること

### Takeaway
行ロック待ちは Performance Insights で SQL と待機を見る。パラメータ `innodb_lock_wait_timeout` を確認する。ブロッカーは `SHOW ENGINE INNODB STATUS` と、version 3 向けページにある `performance_schema.data_lock_waits` クエリで特定し、idle か active かを分ける。アイドルならセッション終了を検討するが、長いロールバックがあり得る。デッドロックは検出 ON が既定で、PI の `innodb_deadlocks` と `innodb_lock_wait_timeout`、CloudWatch `Deadlocks`、必要なら `innodb_print_all_deadlocks`。長いトランザクションは `information_schema.innodb_trx`。8.4.8 以上なら purge 対策として `aurora_transaction_timeout` がある。8.4.7 には無い。

### Cited Findings
- 行ロック待ちページの手順: Performance Insights、楽観ロック、複数行文、パーティション、`innodb_lock_wait_timeout`、ブロッカーの idle/active、`SHOW ENGINE INNODB STATUS`、`data_lock_waits` 結合、アイドルブロッカーの終了は長いロールバックがあり得る。対象エンジンの記載は version 3 のみ。 — [synch/cond/innodb/row_lock_wait](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/ams-waits.row-lock-wait.html)
- デッドロック節: 即時コミット、大きなトランザクションを小さくする、ロック順を揃える、索引でロック範囲を減らす、ギャップロックなら READ COMMITTED、アプリはリトライ。監視は `SHOW ENGINE INNODB STATUS`、`innodb_print_all_deadlocks`、CloudWatch `Deadlocks`、Performance Insights の `innodb_deadlocks` と `innodb_lock_wait_timeout`。 — [Minimizing and troubleshooting Aurora MySQL deadlocks](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.BestPractices.FeatureRecommendations.html)
- 履歴リスト: `innodb_trx` と `processlist`、レプリカも見る、`RollbackSegmentHistoryListLength`。 — [The InnoDB history list length increased significantly](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/proactive-insights.history-list.html)
- 8.4.8 以上の `aurora_transaction_timeout` と `Aurora_transaction_timeouts`。8.4.7 は対象外。 — [Transaction timeout in Amazon Aurora MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.TransactionTimeout.html)
- 接続枯渇は 1040。アイドル接続もメモリを使う。RDS Proxy は接続の頻繁な開閉や多数の長寿命接続向けとして性能ページが推奨。 — [Troubleshooting connection issues for Aurora MySQL databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/mysql-troubleshooting-dbconn.html)、[Managing performance and scaling for Amazon Aurora MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Performance.html)

### Inferences
- 50→120 の変更は、待ちエラーを遅らせる対症である。AWS の手順が先に置いているのは、ブロッカーの特定と、アイドルならブロッカーを終わらせることである。タイムアウト延長はブロッカー調査の代わりにならない。
- 8.4.7 で使える公式手順は、version 3 向けと書かれた PI / INNODB STATUS / `data_lock_waits` / デッドロックメトリクスを、版注記の限界を承知で使うことになる。`aurora_transaction_timeout` は手順に入れられない。

### Gaps
- `mysql.rds_kill` と `mysql.rds_kill_query` の 8.4 での構文は「Ending a session or query」ページを開いていない。
- DevOps Guru のロック待ち洞察が 8.4.7 にあるかは未確認。
