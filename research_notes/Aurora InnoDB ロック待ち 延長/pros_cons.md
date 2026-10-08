# Aurora MySQL Serverless v2 で innodb_lock_wait_timeout を 50 秒から 120 秒へ上げる利害（調査日 2026-10-08）

対象は OLTP とバッチが同じ Aurora MySQL Serverless v2 に載る構成。比較する値は既定の 50 秒と 120 秒。50 対 120 の公開ベンチマークは見つかっていない。数値で「何秒待つか」を語る箇所は、測定ではなく天井の算術か、たとえ話上の計算である。

このパラメータが変えるのは、InnoDB の行ロックを待っている側がエラー 1205 で諦めるまでの上限だけである。ロックを握っている処理は速くならない。

## 120 秒は 50 秒に対して、機構として何を変えるか

### Takeaway
120 秒は「待たされている文が諦める天井」を 50 秒から 2.4 倍（70 秒長い）にするだけで、相手のトランザクションを速くしない。デッドロック検出が有効（既定）なら、デッドロックはタイムアウトを待たずに切られる。実際の待ち時間は「相手がロックを放す時刻」と「天井」の早い方であり、全クエリが 120 秒になるわけではない。

### Cited Findings
- `innodb_lock_wait_timeout` は、InnoDB トランザクションが行ロックを待つ秒数。既定 50、最小 1、最大 1073741824、単位は秒。スコープは Global と Session、動的変更可。SET_VAR ヒントは不可。 — [MySQL 8.4 Reference Manual: innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- 天井を超えると文は `ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction` で失敗する。既定ではその文だけがロールバックされ、トランザクション全体は戻らない。全体を戻すにはサーバ起動オプション `--innodb-rollback-on-timeout` が必要。 — [同マニュアル](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- このタイムアウトが対象にするのは InnoDB の行ロックだけである。MySQL のテーブルロック待ちには効かない。 — [同マニュアル](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- `innodb_deadlock_detect` の既定は ON。有効なとき、ロック待ちタイムアウトはデッドロックに適用されない。InnoDB はデッドロックを直ちに検出し、デッドロックしたトランザクションの一方をロールバックする。検出を無効にしたときだけ、デッドロック解消を `innodb_lock_wait_timeout` に頼る。 — [同マニュアル](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout); [Deadlock Detection](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlock-detection.html)
- デッドロックはトランザクション全体がロールバックされる。ロック待ちタイムアウトは、既定では待っていた文だけがロールバックされる。 — [InnoDB Error Handling](https://dev.mysql.com/doc/refman/8.4/en/innodb-error-handling.html)
- 待ちグラフが深すぎる、または長すぎる場合（待ちリスト上のトランザクションが 200 に達したとき）、InnoDB Monitor は `TOO DEEP OR LONG SEARCH IN THE LOCK TABLE WAITS-FOR GRAPH, WE WILL ROLL BACK FOLLOWING TRANSACTION` と出してトランザクションを戻す。これはタイムアウトとは別の上限である。 — [Deadlock Detection](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlock-detection.html)
- ALTER のようなメタデータロック（MDL）待ちは別変数 `lock_wait_timeout` の管轄であり、`innodb_lock_wait_timeout` はレコードとテーブルに対する InnoDB 側のロックに効く、と MySQL 開発者 Jakub Lopuszanski が 2024-05-15 に 8.4 マニュアルを指して説明している。 — [MySQL Bug #114949](https://bugs.mysql.com/bug.php?id=114949)
- Percona（2013）は、当時のマニュアルを引用し、`lock_wait_timeout` はメタデータロック取得のタイムアウトで許容範囲 1〜31536000 秒、既定 31536000（1 年）だと書いている。対比として `innodb_lock_wait_timeout` の既定は 50 秒だと述べている。 — [Percona: lock_wait_timeout](https://www.percona.com/blog/mysql-5-5-lock_wait_timeout-patience-is-a-virtue-and-a-locked-server/)
- `SET GLOBAL` は、その後に接続するクライアントの初期値だけを変える。既存接続は変わらない。`SET SESSION` はそのクライアントだけを変える。どのクライアントでも SESSION は変更できる。 — [MySQL 8.4 Reference Manual: innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)

### Inferences
- 天井の比は 120/50 = 2.4 で、失敗するまでの上限が 70 秒延びる。これは算術であり、測定された遅延倍率ではない。
- 経験する待ち時間は min(相手がロックを保持する時間, 天井) である。相手が 30 秒で終われば 50 でも 120 でも結果は同じで、待ちは 30 秒である。差が出るのは相手の保持が 50 秒を超えるときだけである。
- 50 &lt; 保持時間 ≤ 120 なら、50 では 1205、120 では相手の完了時に成功する。保持時間が 120 秒を超えるなら両方失敗し、120 の方が接続を最大 70 秒長く占有する。
- デッドロック検出 ON（既定）のままなら、120 にしてもデッドロックが 120 秒生き残るわけではない。マニュアルは「直ちに」と書くだけで、検出遅延のミリ秒 SLA は示していない。
- 検出を OFF にしているクラスタだけは、デッドロック解消がこの天井まで遅れる。120 への変更は、その設定と組み合わせるとデッドロックの被害時間も延ばす。
- パーティション交換や ALTER が待っているものが MDL なら、この値を 120 にしてもその待ちは短くも長くもならない。行ロック競合とは別に `lock_wait_timeout` を見る必要がある。31536000 という既定秒数は 2013 年の Percona 引用であり、8.4 現行ページの数値は今回抜いていない（下記 Gaps）。

### Gaps
- 50 秒と 120 秒を同じ負荷で比べた公開測定は見つからなかった。
- 「直ちに検出」の実測遅延（ミリ秒）はマニュアルにない。
- `lock_wait_timeout` の 8.4 現行ページから既定値 31536000 を今回再抜粋していない。役割の分離は 2024 年の開発者コメントと 8.4 へのリンクで確認した。
- クライアントが切断したとき、InnoDB のロック待ちが即座に中断される、という文はマニュアル抜粋になかった。呼び出し側が先に諦めてもサーバ側の待ちが天井まで残る、は層が独立であることから導く推論である。

## 120 秒に上げる利益

### Takeaway
正当な処理がロックを 50〜120 秒握っていて、そのあと必ず放すなら、待たされている側の 1205 は減る。マニュアル自身が、対話的 OLTP では下げ、長いバックエンドでは上げてよいと書いている。効くのは行ロック待ちであり、セッション単位で上げる方がグローバルより安全である。

### Cited Findings
- マニュアルは、対話的アプリや OLTP では値を下げ、利用者へ早く返すか、更新をキューに回すことを勧めている。データウェアハウスの変換ステップのように、他の大きな INSERT や UPDATE の完了を待つ長いバックエンドでは、値を上げてよいと書いている。 — [MySQL 8.4 Reference Manual: innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- スコープが Session なので、バッチ接続だけ `SET SESSION innodb_lock_wait_timeout = 120` とでき、その後に来る接続へグローバル値は染み出さない。グローバル変更は、変更後に新規接続するクライアントへ効く。 — [同マニュアル](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- 行ロック待ちのイベント `synch/cond/innodb/row_lock_wait` について、AWS は複数の DML が同じ行へ同時に入ることが原因だとし、対処の一つとしてこのパラメータ（タイムアウト前にどれだけ待つか）を確認するよう書いている。同じページの例は `SELECT ... FOR UPDATE` が、`User sleep` 状態のトランザクション（ACTIVE 82 sec）にブロックされている。 — [synch/cond/innodb/row_lock_wait](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/ams-waits.row-lock-wait.html)
- アイドルなブロック側セッションは切ってよいが、長いロールバックを起こし得る、と AWS は書いている。 — [同ページ](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/ams-waits.row-lock-wait.html)
- 失敗を少なくするには、トランザクション内で最初のデータ変更から COMMIT までの仕事を小さくし、ロックを持つ時間と行数を短くする、とマニュアルは書いている。仕事をトランザクション分割することも有用だとしている。 — [InnoDB Error Handling](https://dev.mysql.com/doc/refman/8.4/en/innodb-error-handling.html)

### Inferences
- 利益が出る具体像は「ブロック側が 50〜120 秒で終わる正当な行ロック」である。大きな DELETE、`SELECT FOR UPDATE` を使うレポートやキューワーカー、外部キー検査やカスケードが行ロックをその帯に収めて放す、などが概念上は該当する。外部キーが実際にどのロックを取るかは今回、ロックの節を開いて確認していない。
- AWS の例（ブロック側が 82 秒で `User sleep`）に当てはめると、天井 50 では待ち側は 1205、天井 120 では 82 秒目に成功し得る。これは例の数字を使った機構の当てはめであり、AWS が 50 と 120 を測った結果ではない。相手が 82 秒で終わるなら、オペレータが気づいて切る窓も 50 秒より広い。相手が眠ったままなら、その窓は被害の延長でもある。
- パーティション交換を「このパラメータを上げる理由」に数えない。MDL 待ちは別変数である（前節）。
- グローバル 120 より、バッチセッションだけの 120 の方がマニュアルの「OLTP は下げ、バックエンドは上げる」と両立する。

### Gaps
- 「バッチ更新の何パーセントが 50〜120 秒の正当なロック保持か」という測定は、この調査の範囲に存在しない。利益の大きさはワークロード測定なしには分からない。
- 外部キーカスケードが行ロックを何秒握るかの一次記述は今回未取得。

## 120 秒に上げる不利益

### Takeaway
相手がロックを放さないとき、利用者は最大 120 秒待ってから失敗する。その間、プールの接続、Lambda の実行時間、呼び出し側がすでに切ったあとのサーバ側セッションが塞がる。待ち側が先に取ったロックも、その余分な時間のあいだ他者を止める。1205 の件数だけを見ていると、悪化がアラートから消える。

### Cited Findings
- 既定ではロック待ちタイムアウトで戻るのは現在の文だけである。トランザクションは開いたまま残る。 — [MySQL 8.4: innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout); [Error Handling](https://dev.mysql.com/doc/refman/8.4/en/innodb-error-handling.html)
- `innodb_rollback_on_timeout` の既定は OFF、スコープは Global、Dynamic は No（実行中の SET では変わらない。変更には再起動が要る）。OFF ではタイムアウト時に最後の文だけを戻す。ON ではトランザクション全体を中断して戻す。 — [MySQL 8.4: innodb_rollback_on_timeout](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_rollback_on_timeout)
- Percona の実験では、OFF のまま先に成功した UPDATE はタイムアウト後も残り、その後の COMMIT で表に書かれた。ON ではその UPDATE も残らなかった。 — [Percona: innodb_rollback_on_timeout](https://www.percona.com/blog/the-transaction-behavior-impact-of-innodb_rollback_on_timeout-in-mysql/)
- エラー処理のページは、MySQL 層が検出する「その他のエラー」について、対応する SQL 文をロールバックするが、単一文のロールバックではロックは解放されない、と書いている。この文は 1205 の段落そのものではなく、その後の「その他のエラー」に付いている。 — [InnoDB Error Handling](https://dev.mysql.com/doc/refman/8.4/en/innodb-error-handling.html)
- REPEATABLE READ でファントムを防ぐため、InnoDB はインデックス行ロックとギャップロックを組み合わせたネクストキーロックを使う。ギャップ（例: id 90 と 102 のあいだ）をロックしないと、他セッションが id 101 を挿入できる。 — [MySQL 8.4: Phantom Rows](https://dev.mysql.com/doc/refman/8.4/en/innodb-next-key-locking.html)
- `Innodb_row_lock_time`、`Innodb_row_lock_time_avg`、`Innodb_row_lock_time_max` の単位はミリ秒。`Innodb_row_lock_time_max` は行ロック取得にかかった最大時間。`Innodb_row_lock_current_waits` は今待たれている行ロック数。`Innodb_row_lock_waits` は待ちが発生した回数。 — [MySQL 8.4: Server Status Variables](https://dev.mysql.com/doc/refman/8.4/en/server-status-variables.html#statvar_Innodb_row_lock_time)
- Aurora Serverless v2 は、各ライター／リーダーについて CPU、メモリ、ネットワークの利用を負荷として追跡し、容量が負荷に不足なら ACU を上げ、過剰なら下げる。 — [How Aurora Serverless v2 works](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html)（ページ日付 2026-10-08）
- 1 ACU はおおよそ 2 GiB のメモリと、それに応じた CPU とネットワークである。 — [AWS What's New: scale to zero (2024-11-20)](https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-aurora-serverless-v2-scaling-zero-capacity/); [How it works](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html)
- `max_connections` は最大 ACU から導いたメモリ量に基づき、スケールダウンしても接続が切れないよう一定に保たれる。現在の ACU ではなく最大 ACU のメモリで式を評価する。 — [Aurora Serverless v2 capacity](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.setting-capacity.html)（ページ日付 2026-10-08）
- 共有メモリに効くパラメータを既定より大きくすると、最小容量までスケールダウンできなくなる、と容量設定の案内にある。例として `max_connections` が挙がっている。 — [同ページの GitHub 版案内](https://github.com/awsdocs/amazon-aurora-user-guide/blob/main/doc_source/aurora-serverless-v2.setting-capacity.md)
- 0 ACU（自動ポーズ）は、利用者の接続がない状態が続いたあとに休止する。Aurora MySQL 3.08 以降が対象。RDS Proxy のように接続を開き続けるプロキシがあると、0 まで落ちない場面がある。 — [What's New (2024-11-20)](https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-aurora-serverless-v2-scaling-zero-capacity/); [InfoQ によるドキュメント要約 (2024-12)](https://www.infoq.com/news/2024/12/aurora-serverless-zero-capacity/)
- 履歴リストが伸びる典型原因は、読み書きどちらの長時間トランザクションと、重い書き込みである。長くなるとクエリとシャットダウンが遅くなる。CloudWatch / Performance Insights の `RollbackSegmentHistoryListLength` と、`trx_rseg_history_len`（INFORMATION_SCHEMA と Performance Insights で見える）が関連メトリクスである。 — [Aurora proactive insight: history list length](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/proactive-insights.history-list.html)（ページ日付 2026-09-30）
- 二次情報として、`trx_rseg_history_len` は `RollbackSegmentHistoryListLength` と同じ値だ、と AWS が述べている旨を検証記事が引用している。 — [Zenn: Aurora MySQL パージメトリクス](https://zenn.dev/nttdata_tech/articles/7d0b6dac026b78)
- Percona は、変更を含むトランザクションを長く開いたままにすると他トランザクションがロック待ちで失敗すること、InnoDB は短く小さいトランザクション向けであること、ループのバッチはイテレーションごとに COMMIT すべきこと、アイドル・イン・トランザクションは殺す対象であること、を書いている。 — [Percona: long-running transactions (2011)](https://www.percona.com/blog/how-to-debug-long-running-transactions-in-mysql/)
- 行ロック待ちは Performance Insights では `synch/cond/innodb/row_lock_wait` と `synch/cond/innodb/row_lock_wait_cond` として出る。メタデータロック待ちは別イベント `synch/cond/sql/MDL_context::COND_wait_status` である。 — [Aurora MySQL wait events](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.Waitevents.html)

### Inferences
- テイル遅延: ブロックが解けないリクエストの上限が 50 秒から 120 秒になる。成功するリクエストの待ちは相手の保持時間のままなので、平均が必ず 2.4 倍になるわけではない。悪くなるのは「50 秒を超えてもロックが残る」裾である。
- ロック連鎖: 待ち側が同じトランザクションで既に更新した行のロックは、待ちのあいだ解放されない（トランザクションが開いたまま、先の変更が残る、という Percona の結果とマニュアルの「文だけ戻す」から導く）。天井を 70 秒延ばすと、その先行ロックが第三者を止める時間も最大 70 秒延びる。1205 のあと OFF のままだと、危険な「半分だけ戻った開きっぱなし」は 70 秒遅く始まる。そのあいだに待ち行列は長くなり得る。
- ギャップ／ネクストキー: ブロックされるのは 1 行とは限らない。挿入を拒むギャップが、余分な待ち時間のあいだ閉じたままになる。影響範囲は REPEATABLE READ の範囲スキャンで特に広い。隔離レベルの既定が REPEATABLE READ であることは今回、変数ページを再抜粋していない。
- コンボイと再試行: 同じロックに N 本がぶら下がると、全員がほぼ同時に天井へ達する。アプリが待たずに再試行すると、同じロックへ再び N 本が入る。120 秒は 1 回あたりの占有が長く、50 秒は失敗と再試行の回数が多い。どちらも、ブロック側を放置したままでは悪化する。ロック解放の瞬間に待ち側が一斉に走り出す（thundering herd）のは機構上あり得るが、測定は見つかっていない。
- プール: ロック待ち中のセッションは接続を返さない。枠が尽きると、新しいリクエストは SQL の 1205 ではなく、プールから接続を借りる側のタイムアウトで失敗する。RDS Proxy の借り出し上限は別節。
- Aurora の ACU: ロック待ちスレッドは条件変数で眠る（イベント名が `synch/cond/...`）。CPU を燃やし続ける負荷ではない。したがって「待ちが増えると CPU 理由で必ずスケールアップする」とは言えない。開いた接続であること自体は、0 ACU へのポーズを妨げる（接続があるかぎり休止しない、という仕様）。0 より上のスケールインは CPU・メモリ・ネットワークの負荷で判断される。眠っている待ちがメモリをどれだけ食ってスケールインを止めるかは、今回の資料に測定がない。`max_connections` を最大 ACU 基準で固定しているので、待ち接続はスケールダウンで切られない。
- 履歴リスト: 待ちのあいだトランザクションが開いたままだと、その読み取りビューがパージを止める。AWS は長時間トランザクションを履歴リスト肥大の原因に挙げている。70 秒の延長は、その窓を広げる。何件増えるかは書き込み量次第で、測定はない。
- 監視の歪み: 天井を 120 秒にすると、50 秒で切れていた待ちが 1205 に数えられなくなる。`Innodb_row_lock_time_max` はミリ秒の高水位なので、理論上 50000 までだった山が 120000 まで許される。1205 の件数だけをアラートにすると、待ちが長くなっているのに「改善した」ように見える。高水位がいつリセットされるかは、今回のステータス変数の文には書かれていない。
- バグの発見が遅れる: コミット忘れや、トランザクションの途中でユーザ入力を待つ実装は、50 秒ならその時点で待ち側が例外を上げる。120 秒では同じバグが 70 秒長く本番の接続を握る。Baron Schwartz が問題視したのは、この「開いたまま何もしない」と「COMMIT しないバッチループ」である。

### Gaps
- ロック待ちスレッド 1 本あたりのメモリ量、およびそれが ACU スケールインを止めた実測は見つからなかった。「ACU が下がらなくなる」は接続がポーズを妨げる点までは一次情報、CPU 以外のメモリ圧力は推論である。
- `Innodb_row_lock_time_max` が起動以降の高水位か、`FLUSH STATUS` で戻るかは、抜粋した定義文には無い。
- CloudWatch メトリクス名 `Deadlocks` や `BlockedTransactions` は、今回開いたページの抜粋には出てこない。履歴リストと PI の待ちイベントは確認した。
- クライアント切断がロック待ちを即時アボートするかは未確認（前節 Gaps）。

## 専門家の勧め: グローバルに上げて待ちを隠さない

### Takeaway
MySQL 公式は「OLTP は下げる、長いバックエンドは上げる」と分けている。DBA の実務寄りの意見は、それより厳しく、50 秒に達する状況はアプリの設計問題であり、グローバルに上げて 1205 を消すのはハングへ取り替えるだけだ、と述べている。一致点は、短いトランザクション、バッチはセッションだけ上げる、対話はむしろ下げる、である。

### Cited Findings
- 公式の増減ガイダンスは前掲のとおり。OLTP では下げ、ウェアハウス型の長いバックエンドでは上げてよい。 — [MySQL 8.4](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- 公式は、最初の変更から COMMIT までを短くし、ロックする行数を減らすことを、デッドロックとロック待ちの両方を減らす方法として書いている。 — [Error Handling](https://dev.mysql.com/doc/refman/8.4/en/innodb-error-handling.html)
- Rick James（DBA Stack Exchange）: 「50 秒の InnoDB タイムアウトに当たるなら、アプリの設計が悪い。このタイムアウトは、他の方法では捕まえられない行儀の悪いものを捕まえるためにある。」 — [DBA Stack Exchange: Set lock wait timeout](https://dba.stackexchange.com/questions/2763/set-lock-wait-timeout-in-mysql)
- Baron Schwartz（Percona, 2011）: 短い小さいトランザクションが前提。バッチはループの途中で COMMIT する。アイドル・イン・トランザクションは開発側が直すまで殺す。この二種がシステム全体をロック待ち停止へ落とした例を何度も見ている。 — [Percona](https://www.percona.com/blog/how-to-debug-long-running-transactions-in-mysql/)
- Percona の Sri Sakthivel（2020）: DBA としては `innodb_rollback_on_timeout` を有効にするか、アプリが失敗トランザクションを扱うべきだ。ドライバが 1205 を受けたらトランザクション全体を ROLLBACK してやり直すことは、このパラメータの値にかかわらず原子性のために必要だ、と結論している。変更には再起動が必要だとも書いている。 — [Percona: rollback_on_timeout](https://www.percona.com/blog/the-transaction-behavior-impact-of-innodb_rollback_on_timeout-in-mysql/)
- Percona Server の OLTP 向け設定例（スレッドプール記事、2026-10-05）は `innodb_lock_wait_timeout` を 50 のままにし、`innodb_rollback_on_timeout` を ON にしている。これは「50 を維持した公開設定例」であり、120 を禁止する実験結果ではない。 — [Percona: Thread Pool (Part 1)](https://www.percona.com/blog/thread-pool-in-percona-server-and-mysql-part-1/)
- 二次情報（Data Ops Pulse / neverblink, 2026-09-22）: 多くの OLTP では既定 50 秒は長すぎ、5〜10 秒で早く失敗させた方が問題が見える。バッチはグローバルではなくセッションで高くする。最大値まで上げるとクラッシュがハングに変わる。索引と短いトランザクションが本筋で、タイムアウトは症状のレバーにすぎない。 — [neverblink: MySQL Error 1205](https://neverblink.ai/kb/mysql-innodb-lock-wait-timeout-exceeded)

### Inferences
- 「上げるな」を MySQL 公式の全文として引用するのは不正確である。公式はバックエンドでの引き上げを明示的に許している。禁止に近いのは「グローバル値を上げて OLTP の 1205 を見えなくする」ことについてであり、こちらは Rick James、Baron Schwartz、2026 年の二次記事、Percona の OLTP 設定例が同じ方向を向く。
- 説明用の一文にするとこうなる。データベースはロックの順番待ちの「諦める時刻」しか変えない。行列の原因（長いトランザクション、コミット忘れ、同じ行への集中）は残る。対話画面は早く諦めて見せ、夜間バッチの接続だけ長く待つ、が公式と実務の重なる点である。

### Gaps
- 「主要クラウドの Aurora パラメータ推奨値が 50 のままである」という AWS の一文は見つかっていない。AWS の行ロック待ちページは値の確認を勧めるだけで、120 を推奨も否定もしていない。

## アプリケーション契約: 1205 のあと何をやり直すか

### Takeaway
マニュアルは、既定なら「文をやり直す」、`innodb_rollback_on_timeout` が ON なら「トランザクション全体をやり直す」と分けている。プールを使うアプリでは、Percona が言うとおり 1205 を受けたらトランザクション全体を ROLLBACK し、仕事の単位ごと、間を空けてやり直す方が安全である。先に成功した文が残っているのに、その文だけ別接続でやり直すのは危険である。

### Cited Findings
- デッドロック: トランザクション全体が戻る。全体をやり直す。 — [Error Handling](https://dev.mysql.com/doc/refman/8.4/en/innodb-error-handling.html)
- ロック待ちタイムアウト: 既定では現在の文だけが戻る。`--innodb-rollback-on-timeout` を有効にしていれば全体が戻る。マニュアルの指示は「既定なら文を再試行、オプション有効ならトランザクション全体を再試行」。 — [Error Handling](https://dev.mysql.com/doc/refman/8.4/en/innodb-error-handling.html)
- Percona の実験は、OFF のとき先の UPDATE がタイムアウトを生き延び、あとから COMMIT すると表に残ることを示している。結論として、1205 ではトランザクション全体を ROLLBACK してトランザクションをやり直すことを、パラメータ値にかかわらず勧めている。 — [Percona](https://www.percona.com/blog/the-transaction-behavior-impact-of-innodb_rollback_on_timeout-in-mysql/)
- マニュアルは、ロック待ちもデッドロックも忙しいサーバでは普通に起き、アプリは再試行で扱う必要がある、とも書いている。 — [Error Handling](https://dev.mysql.com/doc/refman/8.4/en/innodb-error-handling.html)

### Inferences
- マニュアルの「文を再試行」は、同じ開いたトランザクションの中で、戻った文だけをやり直す、という読める。先の文の変更は生きているので、業務的には「全部なかったこと」ではない。
- 接続プールへ戻す前に ROLLBACK しないと、次の借り主が開いたトランザクションと、そのロックを引き継ぐ。これは「トランザクションが残る」という仕様からの推論であり、特定製品の不具合報告を今回引用したわけではない。
- autocommit=1 なら文がトランザクションそのものなので、文のロールバックで単位が閉じる。複数文を 1 トランザクションにまとめている場合（autocommit=0 や明示的な BEGIN）ほど、全体 ROLLBACK が重要になる。
- 再試行は仕事の単位全体にし、待ち時間をばらけさせる（ジッタ）。同時に失敗した N 本が待たずに同じロックへ戻ると、負荷が掛け算になる。ジッタをマニュアルが義務付けているわけではない。
- デッドロック経路はすでに全体が戻っているので、文だけのやり直しでは足りない。1205 とデッドロックを同じ「単位ごとやり直し」で扱うと分岐が減る。デッドロックのエラー番号は今回の抜粋に無い。

### Gaps
- MySQL Connector/J が 1205 のあと接続をプールへ返す実装詳細は未確認。
- ジッタの具体的な秒数を裏づける測定はない。

## 呼び出し側タイムアウトとのずれ（ALB、API Gateway、RDS Proxy）

### Takeaway
データベースが 120 秒待つ設定でも、その前に ALB や API Gateway が利用者へ失敗を返すことがある。既定のままなら、ALB の無通信打ち切りは 60 秒、API Gateway の統合タイムアウトは 29 秒である。利用者はもういないのに、MySQL は天井まで待ち、接続を占有し得る。これが 120 秒化の大きい欠点である。

### Cited Findings
- Application Load Balancer の接続アイドルタイムアウトは、クライアント側またはターゲット側の既存接続で、データが送受信されない時間である。既定は 60 秒。有効範囲は 1〜4000 秒。長い処理では、アイドル時間が来る前に少なくとも 1 バイト送り、必要ならアイドル時間を延ばすよう AWS は書いている。アプリ側のアイドル時間はロードバランサより長くするようも勧めている。 — [ALB 属性（中国リージョン向けミラーの説明。既定 60 秒）](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/application/edit-load-balancer-attributes.html); [API: idle_timeout.timeout_seconds の既定 60 秒、範囲 1〜4000。ページ日付 2026-09-15](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/APIReference/API_LoadBalancerAttribute.html)
- 同じ API リファレンスで、HTTP クライアントキープアライブ `client_keep_alive.seconds` の既定は 3600 秒、範囲 60〜604800。アイドルタイムアウトとは別である。 — [LoadBalancerAttribute](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/APIReference/API_LoadBalancerAttribute.html)
- Classic Load Balancer の文書（2026-09-02）もアイドルタイムアウト既定 60 秒と書いている。Network Load Balancer の TCP アイドル既定は 350 秒で、ALB とは違う。 — [Classic LB idle timeout](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/config-idle-timeout.html); [NLB TCP idle timeout](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/update-idle-timeout.html)
- API Gateway の統合タイムアウト既定は 29 秒（29000 ミリ秒）。超えると HTTP 504。エッジ最適化 API は 50 ミリ秒〜29 秒で、クォータ引き上げ不可。Regional とプライベート REST は 29 秒を超えられるが、アカウントのリージョン別スロットルクォータの引き下げが必要になることがある。クォータ表の更新として見える日付は 2026-10-06。 — [API Gateway REST quotas](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-execution-service-limits-table.html); [Integration.timeoutInMillis](https://docs.aws.amazon.com/apigateway/latest/api/API_Integration.html)
- 29 秒超を許す変更は 2024-06-04 に発表されている。対象は Regional REST とプライベート REST。 — [AWS What's New (2024-06-04)](https://aws.amazon.com/about-aws/whats-new/2024/06/amazon-api-gateway-integration-timeout-limit-29-seconds/)
- 2026-05-14 の AWS re:Post 記事は、Regional とプライベート REST について、よくある引き上げは 90 秒までで、それ以上はサポートケース、と書いている。公式クォータ表は「29 秒超が可能」と書くが、90 秒という上限は表の本文には無い。 — [re:Post (2026-05-14)](https://repost.aws/articles/AROecfpxHaT-S4ADBXRG8DBg/resolve-endpoint-request-timed-out-504-errors-when-api-gateway-invokes-a-lambda-function-for-long-running-operations)
- RDS Proxy の `ConnectionBorrowTimeout` は、プールの接続が最大まで開いていて全部が使用中のとき、接続が空くまでプロキシが待つ秒数。既定 120 秒。API の制約は 0〜300 秒。ベストプラクティス頁は最小 0、最大 5 分、既定 2 分と書いている。フェイルオーバー中など、データベースが接続を受けられないときにも適用される。0 にすると、空きが無く新規接続を開く待ちもできないため、最大未満でもエラーになり得る。 — [RDS Proxy connections](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy-connections.html)（2026-09-04）; [ConnectionPoolConfiguration](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_ConnectionPoolConfiguration.html)（2026-10-07）; [Proxy configuration guidelines](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy-best-practices.configuration.html)（2026-10-08）
- RDS Proxy は、セッション変数を `SET` すると、その状態を後続リクエストが前提にするため、原則ピン留めする。ピン留め中はセッションが終わるまで同じ DB 接続を使い、他クライアントはその接続を再利用できない。追跡対象の変数はピン留めせず、同じ値のセッション同士でだけ再利用する。追跡例として `TIME_ZONE`、`TRANSACTION_ISOLATION`、`TRANSACTION_READ_ONLY`、`WAIT_TIMEOUT` が抜粋に見えた。全クライアントで同じ初期化なら、プロキシの initialization query へ移すとピン留めを避けられる、と AWS は書いている。 — [Avoiding pinning an RDS Proxy](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy-pinning.html)

### Inferences
- ALB の 60 秒は「リクエスト全体の制限時間」ではなく「無通信の上限」である。ロック待ちのあいだアプリが 1 バイトも返さないと、既定では約 60 秒でロードバランサが接続を閉じ得る。DB 天井を 120 秒にすると、利用者側が切れたあとも MySQL が最大あと約 60 秒待ち続ける、という段差が既定値どうしで生まれる。閉じたときの HTTP ステータスが必ず 504 である、とは今回の属性ページは書いていない。
- API Gateway を既定 29 秒のままにすると、DB が 120 秒待てても呼び出し元は約 29 秒で 504 になる。エッジ最適化は 2026-10 のクォータ表でも 29 秒を超えられない。Regional / private は延ばせるが、スロットルクォータとの引き換えがあり、延ばしたとしても DB の 120 秒より長くしておかないと段差は残る。
- RDS Proxy の借り出し既定が 120 秒なので、バックエンドがロック待ちで全部ふさがっていると、新しいクライアントは最大 120 秒待ってから借り出しエラーになる。借りたあとにその SQL 自身がさらに最大 120 秒ロック待ちすると、利用者から見た時間は足し算に近づく。これは段差の機構であり、実測ではない。
- バッチだけ `SET SESSION innodb_lock_wait_timeout=120` するのは MySQL 的には正しいが、RDS Proxy 経由だとセッション変数の SET がピン留め条件に当たり得る。`innodb_lock_wait_timeout` が追跡リストに含まれるかは抜粋では確認できなかった。含まれなければ、バッチの SET は多重化を壊し、その DB 接続はセッション終了まで専有される。initialization query に入れると全バックエンド接続が同じ値になり、OLTP とバッチで値を分ける目的と矛盾する。
- JDBC の `Statement.setQueryTimeout` やプールの借り出しタイムアウトが 120 より短いと、同様に「クライアントは死に、サーバは待つ」が起きる。秒数の一次確認は Gaps。

### Gaps
- ALB が無通信で接続を閉じたときのクライアント向けステータスコード（504 かどうか）は、今回開いた属性ページには無かった。
- API Gateway HTTP API（v2）のタイムアウト上限は今回確認していない。上記は REST API の統合タイムアウト。
- CloudFront のオリジン応答タイムアウトと nginx の `proxy_read_timeout` の 2026 年の既定値は未確認。同じ「手前が先に切れる」型として扱うべきだが、秒数は引用しない。
- HikariCP の `connectionTimeout` 既定 30000 ミリ秒は、今回 GitHub の文書を開いていない。機構として、チェックアウト中の接続はロック待ちのあいだ枠を占有する、ということだけを推論する。
- `innodb_lock_wait_timeout` が RDS Proxy の追跡変数リストに入っているかは未確認。
- クライアント切断や `KILL QUERY` がロック待ちをすぐに終わらせるかは未確認。

## Lambda との相互作用

### Takeaway
Lambda がデータベースのロック待ちで止まっている時間も、関数の実行時間である。既定タイムアウトは 3 秒、通常の同期呼び出しの上限は 900 秒（15 分）なので、120 秒の DB 待ちは既定の Lambda では成立しない。タイムアウトを 120 秒以上にすると、待っているあいだ同時実行数が積み上がり、課金対象の時間も伸びる。

### Cited Findings
- Lambda の Timeout は、関数がイベントを処理できる最大秒数。既定 3 秒。1 秒刻みで最大 900 秒（15 分）。タイムアウトに達すると Lambda は呼び出しを止める。文書の更新として見える日付は 2026-10-08。 — [Configure Lambda function timeout](https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html)
- 2026-09-09 以降、Lambda Managed Instances の非同期呼び出しと、一部を除くイベントソースマッピングは最大 5400 秒（90 分）まで延ばせる。同期呼び出しの上限は 15 分のまま、と同ブログは書いている。 — [AWS Compute Blog (2026-09-09)](https://aws.amazon.com/blogs/compute/announcing-90-minute-function-timeout-on-aws-lambda-managed-instances/); [Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)
- トラブルシューティング文書は、タイムアウトの一般原因として「関数が他サービスへ要求し、その応答が長い」ことを挙げている。 — [Troubleshoot Lambda configuration](https://docs.aws.amazon.com/lambda/latest/dg/troubleshooting-configuration.html)
- API Gateway が Lambda プロキシを待つ時間の例として、知識センター系の記述は最大 29 秒に触れている（REST の既定と一致）。 — [Lambda invocation timeouts knowledge center の検索抜粋](https://aws.amazon.com/premiumsupport/knowledge-center/lambda-troubleshoot-invocation-timeouts)

### Inferences
- OLTP の同期 API が Lambda なら、DB 天井 120 秒は Lambda 既定 3 秒でも、API Gateway 既定 29 秒でも、関数タイムアウトを 60 秒にした程度でも、呼び出し側が先に死ぬ。死んだあとの DB セッションがいつ解放されるかは未確認（前節）。
- たとえ話: 毎秒 R 件がロック待ちで 120 秒滞留すると、同時実行はおよそ R×120 だけその待ちに占有される。50 秒天井なら占有はおよそ R×50。これはリトルの法則による説明用の計算であり、Lambda の実測ではない。アカウントの同時実行クォータの 2026 年の既定値は、今回の公式抜粋には数値が無かった。
- バッチを Lambda で 120 秒以上待つなら、関数タイムアウトは 120 より長く、API Gateway や ALB を通さない経路（非同期、キュー）にする必要がある。Managed Instances の 90 分は同期 API の逃げ道にはならない。

### Gaps
- Lambda タイムアウト時に、RDS Proxy 越しのサーバ側クエリが即時中断されるかは未確認。
- 非同期呼び出しがタイムアウトを自動再試行する回数は、今回の一次抜粋では確認していない。再試行があるとロック待ち負荷はさらに増える、は可能性としてだけ置く。
- 同時実行の既定クォータ（しばしば 1000 と言われる値）は、今回開いたクォータ抜粋に数値が無かった。

## たとえ話上の計算（測定ではない）

### Takeaway
200 本のプールが同じロックに並んだとき、相手が 80 秒で終わるなら 50 秒設定は失敗し 120 秒設定は成功する。相手が 10 分止まったままなら、どちらも失敗し、120 秒設定は接続を 1 本あたり 70 秒、200 本合計で 14000 接続秒長く塞ぐ。この数字は説明用の算術である。

### Cited Findings
- 待ちの上限が 50 秒または 120 秒であることはマニュアルの定義から来る。実待ちは相手が放すまで、ただし上限を超えない。 — [MySQL 8.4](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- 50 対 120 のベンチマークは見つかっていない（このノート全体の前提）。

### Inferences
前提（すべて仮定）: プール 200、待ち側は開始時点で一斉にブロックされる、アプリは 1205 のあと接続を返す、ブロック側は 1 本。

成功する話。ブロック側が 80 秒で COMMIT する。

- 天井 50: 待ち側は 50 秒で 1205。相手はその 30 秒後に終わるが、待っていた仕事は失敗している。接続は約 50 秒でプールへ戻せる。
- 天井 120: 待ち側は 80 秒目にロックを得て成功する。占有は 120 秒ではなく 80 秒。失敗は減るが、200 本が 80 秒塞がる。

止まったままの話。ブロック側がアイドルのまま 10 分（600 秒）ロックを持つ。

- 天井 50: 200 本は 50 秒でエラーになり、アプリが戻せば枠は空く。占有は 50×200 = 10000 接続秒。
- 天井 120: 200 本は 120 秒まで塞ぐ。占有は 120×200 = 24000 接続秒。余分は 70×200 = 14000 接続秒。この 70 秒のあいだ、プールが 200 で頭打ちなら新しい OLTP は接続を借りられない。
- どちらも最終的には 1205 になる。120 は「成功」を増やさず、空きを 70 秒遅らせるだけである。

すぐ再試行する話（まだ測定ではない）。止まったロックに対し、200 本がエラーの直後に同じ仕事をやり直す。

- 天井 50: 約 50 秒ごとに 200 本が失敗し、再び 200 本が並ぶ。エラー件数は多い。1 分あたりの再試行回数は 120 秒天井より多い。
- 天井 120: 再試行の間隔は長く、件数は少ないが、各試行が枠を 120 秒使う。プール枯渇の時間は長い。
- ジッタが無いと、解放の瞬間または次の天井で 200 本が再び同時に動く。

80 秒の話を非エンジニア向けに言い換えると、「80 秒で終わる正当なバッチなら、50 秒設定は利用者にエラーを見せ、120 秒設定は待たせたうえで成功する。10 分止まるバグなら、120 秒設定はエラーを 70 秒遅らせ、そのあいだ窓口の席を全部埋め続ける。」

### Gaps
- 上記の 200、80 秒、10 分、14000 接続秒は実システムの観測ではない。
- プール実装が 1205 で自動ロールバックして接続を返すとは限らない。返さなければ占有はさらに伸びる。

## 変更の前と後に見ること

### Takeaway
見るべきは 1205 の件数だけではない。今誰が誰を止めているか、ブロック側が何秒生きているか、行ロック待ち時間の合計と最大、Performance Insights の `synch/cond/innodb/row_lock_wait`、履歴リストの長さ、である。変更後に 1205 が減っても、待ち時間最大と履歴リストとプール待ちが増えていれば悪化である。

### Cited Findings
- AWS は行ロック待ちの切り分けに `performance_schema.data_lock_waits`、`performance_schema.data_locks`、`information_schema.innodb_trx`、`information_schema.processlist` の結合を例示している。例では待ちクエリ、ブロック側のロックモード、ブロック側の状態（`User sleep` など）、スレッド ID を出している。 — [synch/cond/innodb/row_lock_wait](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/ams-waits.row-lock-wait.html)
- ステータス変数 `Innodb_row_lock_current_waits`、`Innodb_row_lock_time`（ミリ秒の合計）、`Innodb_row_lock_time_avg`、`Innodb_row_lock_time_max`、`Innodb_row_lock_waits` が定義されている。 — [Server Status Variables](https://dev.mysql.com/doc/refman/8.4/en/server-status-variables.html#statvar_Innodb_row_lock_time)
- Performance Insights の待ちは `synch/cond/innodb/row_lock_wait`（および `row_lock_wait_cond`）。MDL は別イベント。 — [Aurora MySQL wait events](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.Waitevents.html)
- 履歴リストは `RollbackSegmentHistoryListLength` と `trx_rseg_history_len`。長時間トランザクションを特定して終わらせることが AWS の対処に含まれる。 — [history list insight](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/proactive-insights.history-list.html)
- Aurora Serverless v2 の容量は `ServerlessDatabaseCapacity` と `ACUUtilization` で追う、と AWS ブログが書いている。CPU・メモリ・ネットワークがスケールの入力である。 — [AWS Database Blog: ACU range (2024-11-25)](https://aws.amazon.com/blogs/database/understanding-how-acu-minimum-and-maximum-range-impacts-scaling-in-amazon-aurora-serverless-v2/); [How it works](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html)
- アイドルなブロックセッションを切ると長いロールバックがあり得る。 — [row lock wait actions](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/ams-waits.row-lock-wait.html)

### Inferences
変更前にベースラインを取り、変更後は少なくとも次を並べる。1205 件数、`Innodb_row_lock_time` の増分、`Innodb_row_lock_time_max`、`Innodb_row_lock_current_waits`、PI の `synch/cond/innodb/row_lock_wait` の DB ロード、履歴リスト長、ACU、データベース接続数、アプリのプール借り出し失敗、ALB / API Gateway の 5xx。1205 だけ減ってプール失敗や 504 が増えるなら、タイムアウトをデータベースの奥へずらしただけである。

8.4 で使える確認 SQL（AWS の結合例を短くしたもの。本番で重いなら LIMIT と短い間隔に限る）:

```sql
-- 今、誰が誰を待っているか
SELECT r.trx_mysql_thread_id AS waiting_thread,
       r.trx_started AS waiting_started,
       r.trx_query AS waiting_query,
       b.trx_mysql_thread_id AS blocking_thread,
       b.trx_started AS blocking_started,
       TIMESTAMPDIFF(SECOND, b.trx_started, NOW()) AS blocking_age_sec,
       b.trx_state AS blocking_state,
       b.trx_rows_locked,
       b.trx_rows_modified,
       b.trx_query AS blocking_query
FROM performance_schema.data_lock_waits w
JOIN information_schema.innodb_trx r
  ON r.trx_id = w.REQUESTING_ENGINE_TRANSACTION_ID
JOIN information_schema.innodb_trx b
  ON b.trx_id = w.BLOCKING_ENGINE_TRANSACTION_ID;

-- 長く開いているトランザクション
SELECT trx_mysql_thread_id, trx_state, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_sec,
       trx_rows_locked, trx_rows_modified, trx_query
FROM information_schema.innodb_trx
ORDER BY trx_started;

-- 行ロック待ちカウンタ（時間の単位はミリ秒）
SHOW GLOBAL STATUS WHERE Variable_name IN (
  'Innodb_row_lock_current_waits',
  'Innodb_row_lock_time',
  'Innodb_row_lock_time_avg',
  'Innodb_row_lock_time_max',
  'Innodb_row_lock_waits'
);
```

履歴リストは Aurora では Performance Insights / CloudWatch の `RollbackSegmentHistoryListLength` を優先する。INFORMATION_SCHEMA 側は公式が「見える」とだけ述べており、カウンタが無効なら空であり得る。

ブロック側が `Sleep` や `User sleep` で age が数十秒を超えるなら、天井を上げる前にそのトランザクションを切る、またはアプリの COMMIT 漏れを直す方が先である。切断は長いロールバックを起こし得るので、ピーク中の無差別 KILL は別被害になる。

### Gaps
- `information_schema.innodb_metrics` の `trx_rseg_history_len` を Aurora MySQL 8.4 互換で必ず読めるかは、公式が INFORMATION_SCHEMA と書いたところまでで、列名の再確認はしていない。
- `Innodb_deadlocks` ステータス変数は今回の抜粋範囲に含まれていない。デッドロック件数は PI または別変数で補う必要がある。
- Performance Schema のロック計器が Aurora で既定有効かは未確認。AWS の例は `data_lock_waits` を直接参照している。

## 判断ガイド: 120 が妥当なときと、害になるとき

### Takeaway
120 秒が筋が通るのは、測った正当なロック保持が 50 秒を超え 120 秒未満で、待つ主体がバッチセッションだけで、利用者側のタイムアウトとプール枠がその待ちより長いときである。対話 API、呼び出し側が 60 秒以下、プールが上限付近、コミット忘れが常態、のいずれかに当てはまれば、グローバル 120 は害である。その測定はこの調査には無い。

### Cited Findings
- OLTP は下げ、長いバックエンドは上げてよい、という公式の分け方。 — [MySQL 8.4](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- セッションだけ変えられ、グローバルは後続の新規接続に効く。 — [同マニュアル](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- ALB アイドル既定 60 秒。API Gateway 統合の既定 29 秒（エッジは 29 秒のまま超えられない。Regional / private は条件付きで延長可）。 — [ALB 属性 API](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/APIReference/API_LoadBalancerAttribute.html); [API Gateway quotas（2026-10-06）](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-execution-service-limits-table.html)
- RDS Proxy の接続借り出し既定 120 秒（0〜300 秒）。 — [ConnectionPoolConfiguration](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_ConnectionPoolConfiguration.html)
- Lambda 既定 3 秒、同期の上限 900 秒。 — [Lambda timeout](https://docs.aws.amazon.com/lambda/latest/dg/configuration-timeout.html)
- 50 秒に達する状況を設計問題と見る立場（Rick James）と、バッチは途中 COMMIT、アイドルトランザクションは放置しない（Baron Schwartz）。 — [DBA Stack Exchange](https://dba.stackexchange.com/questions/2763/set-lock-wait-timeout-in-mysql); [Percona 2011](https://www.percona.com/blog/how-to-debug-long-running-transactions-in-mysql/)

### Inferences
120 秒が妥当な条件（すべて満たすのが望ましい）:

- 正当なロック保持時間を測っており、その裾が 50 秒より長く、120 秒より短い。この調査はその測定を持っていない。持つまで「妥当」とは決められない。
- 上げるのはバッチまたはバックエンドのセッションだけで、OLTP の接続は 50 以下のまま（公式は OLTP で下げることさえ勧めている）。
- そのバッチを呼ぶ経路のタイムアウトがすべて 120 秒より長い。ALB を通すなら無通信 60 秒の既定を延ばすか、待っているあいだバイトを返す。API Gateway 既定 29 秒の同期 API の奥では 120 秒は成立しない。
- プールに空きがある。最悪、待ち本数 × 最大 120 秒でも借り出しが枯れない。RDS Proxy を使うなら借り出しタイムアウトとピン留めをセットで見る。
- アプリは 1205 でトランザクション全体を ROLLBACK し、単位ごと、ジッタ付きでやり直す。

害になる条件（一つでも強い）:

- 対話的な API のグローバル値を上げて 1205 を減らしたいだけである。
- 呼び出し側が 60 秒以下（ALB 既定、API Gateway 既定、Lambda を短く設定）である。利用者は失敗し、データベースだけが待ち続ける。
- プールが上限付近である。70 秒の追加占有で新規リクエストが借りられなくなる（たとえ話の「止まったまま」）。
- アイドル・イン・トランザクションや COMMIT 漏れが既に見えている。天井を上げると発見と解放が遅れる。
- `innodb_deadlock_detect` を OFF にしている。デッドロック解消まで 120 秒になり得る。

非エンジニア向けの決め方: 「並んでいる人に、あと 70 秒長く列にいてもらう変更である。80 秒で確実に終わる作業なら、エラーの代わりに成功が増える。10 分止まる障害なら、エラーが遅れて届き、そのあいだ窓口が埋まる。画面向けの列を長くする理由にはならない。夜間の一括処理の列だけ長くするのは、公式の説明と矛盾しない。」

### Gaps
- このクラスタで正当なロック保持の p99 が 50 と 120 のあいだにある、という測定は存在しない（調査側では未発見）。判断ガイドの「測ってから」は条件であり、結果ではない。
- Aurora のパラメータグループで `innodb_lock_wait_timeout` が即時適用か再起動かは未確認。MySQL としては Dynamic Yes。`innodb_rollback_on_timeout` は Dynamic No で再起動が必要、とマニュアルと Percona が一致している。
