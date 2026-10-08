# InnoDB `innodb_lock_wait_timeout` の歴史と、50秒という既定が必要になった理由

調査日: 2026-10-08。日付・引用は取得したページに限る。マニュアル本文の「既定は50秒」は、少なくとも MySQL 4.0/4.1 世代のマニュアル文と、2026年時点の MySQL 8.4 リファレンスの両方で確認できる。**「なぜ 10 でも 60 でもなく 50 なのか」を説明した一次資料は見つからなかった。** 初期バイナリ（2001年頃）のソースで別の数値が入っていた、という証拠も見つからなかった。

再話用のたとえは各節の Inferences に置き、歴史上の発言としては扱わない。

## InnoDBの起源と、行ロックがMyISAMの表ロックに対して解いたこと

### Takeaway
InnoDB は Heikki Tuuri が 1995年に興したフィンランドの Innobase のエンジンで、MySQL に組み込まれたのは Wikipedia によれば 2001年3月である。行ロックが解いたのは「表を更新する人が一人しかいられない」問題で、公式マニュアルは行ロックの利点として「1行を長いあいだロックできる」ことも挙げている。その利点が、後述の待ち時間制限を必要にした。

### Cited Findings
- Innobase はヘルシンキの会社で、InnoDB を開発した。2005年以降は Oracle の子会社になり、2013年7月8日をもって事業活動を終えて Oracle に吸収された、と英語版 Wikipedia は書く。 — [Innobase - Wikipedia](https://en.wikipedia.org/wiki/Innobase)
- 同じ Wikipedia の History: 1995年に Heikki Tuuri が InnoDB を開発するために Innobase を設立。2000年9月に MySQL AB との協業が始まり、2001年3月に InnoDB を組み込んだ MySQL がリリースされた。InnoDB は当初クローズドソースで、買い手が見つからず MySQL との協業に入ったあとオープンソースになった。MySQL はその後も Innobase の買収を試みたが、2005年10月に Oracle が Innobase を買収した。Oracle は 2010年1月に、当時 MySQL AB の所有者だった Sun Microsystems も買収した。 — [Innobase - Wikipedia](https://en.wikipedia.org/wiki/Innobase)
- Oracle のプレスリリースは 2005年10月7日 10:38（Redwood Shores）。「Finland-based Innobase OY」を買収したと発表。InnoDB は単体製品ではなく MySQL の一部として配布され、MySQL との契約は翌年更新期限を迎え、Oracle は延長を交渉するつもりだと書いている。金額は非公開。Charles Rozwat のコメントは、InnoDB の開発を続け、オープンソースへの関与を広げる、という内容。 — [Oracle press release, 2005-10-07（Internet Archive）](https://web.archive.org/web/20181115071936/http://www.oracle.com/us/corporate/press/016679_EN)
- InfoWorld も同日（2005年10月7日）に買収を報じ、InnoDB は GPL で配布される MySQL 向けストレージエンジンだと書いている。金額は非公開。 — [InfoWorld, 2005-10-07](https://www.infoworld.com/article/2221276/oracle-buys-finnish-open-source-developer-2.html)
- MySQL 5.7 マニュアル（Oracle が 2026-05-05 時点でも掲出）: MySQL は InnoDB 表に行レベルロックを使い、複数セッションが同時に書き込めるようにする。これは多ユーザ・高競合・OLTP 向きである。行ロックの利点として「別の行なら競合が少ない」「ロールバックで戻す変更が少ない」「1行を長いあいだロックできる」を挙げる。 — [8.11.1 Internal Locking Methods（MySQL 5.7）](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/internal-locking.html)
- 同じ節: MySQL は MyISAM、MEMORY、MERGE に表レベルロックを使う。これらの表を更新できるセッションは一度に一つだけである。読み取り中心・単一ユーザ向き。デッドロックを避けるため、必要なロックをクエリの先頭でまとめて、同じ順で取る。代償は並行性で、表を変更したい他セッションは「今の変更文が終わるまで」待たされる。 — [8.11.1 Internal Locking Methods（MySQL 5.7）](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/internal-locking.html)
- MySQL 5.5 GA の告知（2010-12-15）は、5.5 で InnoDB が MySQL の既定ストレージエンジンになった、と書いている。ACID、参照整合性、クラッシュリカバリを最初から持つ、という説明。Tomas Ulin の発言として、5.5 は Oracle 体制での最初のメジャーリリースであり、2005年に買収された InnoDB チームと MySQL チームが初めて一緒に作ったリリースだ、とも伝えている。 — [Database Trends and Applications, 2010-12-15](https://www.dbta.com/Editorial/News-Flashes/MySQL-55-Now-Generally-Available-72995.aspx)
- 2010年9月20日の Linux.com 記事は、公開されたばかりの MySQL 5.5 リリース候補が、初めて InnoDB を既定バックエンドにした、と書いている。 — [Linux.com, 2010-09-20](https://www.linux.com/news/first-release-candidate-mysql-55-innodb-default/)

### Inferences
- 再話用（史実の比喩であり、Tuuri の逸話ではない）。MyISAM は「ノートのどのページを直すときも、ノート全体を机の上から取り上げる」。別のページを直したい人も、その人が書き終わるまで待てるだけである。マニュアルが言う待ちは「文が終わるまで」であり、トランザクションという長い囲いはない。InnoDB は「直している行だけを押さえる」ので、隣の行は同時に直せる。公式が利点として書く「1行を長くロックできる」は、その行を必要とする次の人にとっては、いつまでも終わらない列に並ぶ理由になる。
- 「5.1 で InnoDB Plugin が既定エンジンになった」は、今回当たった 5.5 の一次・二次告知と合わない。既定エンジンへの切り替えを明示しているのは 5.5 の RC（2010年9月）と GA（2010-12-15）である。5.1 の Plugin が任意同梱だった日付は、この調査ではリリースノートを開いて確認していない。

### Gaps
- Wikipedia の 1995年設立、2000年9月協業開始、2001年3月の組み込みは、この調査では Heikki 本人の告知や MySQL 3.23 のリリースノートでは再確認していない。2001年3月の MySQL の版番号（しばしば 3.23 台と言われる）は未確認のため書かない。
- 2010年1月の Sun 買収は Wikipedia の月単位のみ。完了日は未確認。
- 2013年7月8日の吸収は Wikipedia の記述のみで、フィンランド商業登記は見ていない。
- フィンランド誌 Tivi（2008-02-04）の検索スニペットには、Tuuri が MySQL の買収提案を断り Oracle から 550万ユーロ超を受け取った、とある。Oracle のプレスは金額非公開なので、この金額は採用しない。 — スニペットのみ: [Tivi, 2008-02-04](https://www.tivi.fi/uutiset/a/c0c55230-1833-36f4-8476-467c0abe34a3)

## `innodb_lock_wait_timeout` はいつ文書に現れ、最初から 50 秒か

### Takeaway
確認できた最も古いマニュアル文は MySQL 4.0/4.1 世代で、既定はすでに 50 秒である。4.0.20（マニュアル目次の日付は 2004年5月18日）と 4.1.2 は「LOCK TABLES を InnoDB が見るようになった版」であり、タイムアウト変数が生まれた版ではない。50 という数を選んだ理由の一次資料は無い。別の初期既定だったという証拠も無い。

### Cited Findings
- SCO/Xinuos が配布する `mysql.info` の「15.5 InnoDB Startup Options」は、MySQL 4.0 以降は `--OPT_NAME=VALUE` で指定でき、4.0 より前は `--set-variable` だと前置きしたうえで、次を載せる。`innodb_lock_wait_timeout`: InnoDB トランザクションがロックを待ってロールバックされるまでの秒数。InnoDB は自分のロック表のデッドロックを自動検出してトランザクションをロールバックする。MySQL 4.0.20 と 4.1.2 以降、InnoDB は `LOCK TABLES` のロックに気づく。それより前は、`LOCK TABLES` や InnoDB 以外のトランザクション対応エンジンを同じトランザクションで使うと、InnoDB が見つけられないデッドロックが起き得る。そのときタイムアウトが状況を解く。既定は 50 秒。 — [mysql.info.gz InnoDB start（Xinuos）](http://osr507doc.xinuos.com/cgi-bin/info2html?(mysql.info.gz)InnoDB%20start&lang=en)
- 2004年11月3日、Lukas Bradley が MySQL フォーラムに、当時のマニュアル `http://dev.mysql.com/doc/mysql/en/InnoDB_start.html` からほぼ同じ段落を引用している。「The default is 50 seconds.」まで含む。彼が使っていた版は 4.0.16 で、引用は「4.0.20 と 4.1.2 から LOCK TABLES に気づく」という将来（その時点では直近の過去）の変更説明である。 — [MySQL Forums, 2004-11-03](https://forums.mysql.com/read.php?20,5095,5095)
- MySQL AB の 3.23/4.0/4.1 リファレンス（文書生成日 2006-05-16、対象は 4.1.20 まで）の変更履歴目次に「Changes in MySQL/InnoDB-4.0.20, May 18, 2004」「Changes in MySQL/InnoDB-4.1.2, May 30, 2004」とある。 — [MySQL 3.23, 4.0, 4.1 Reference Manual 目次](http://marcinmazurek.com.pl/download/manual-4.1/index.html)
- MySQL 8.4 リファレンス（この調査で開いた現行本文）: コマンドライン `--innodb-lock-wait-timeout=#`、スコープ Global と Session、Dynamic Yes、整数、Default Value 50、Minimum 1、Maximum 1073741824、単位は秒。InnoDB トランザクションが行ロックを諦めるまでの秒数。既定 50 秒。他の InnoDB トランザクションがロックしている行へ書こうとすると、最大この秒数だけ待ち、`ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction` を出す。タイムアウト時に戻るのは現在の文であり、トランザクション全体ではない。全体を戻すには `--innodb-rollback-on-timeout` で起動する。対話的アプリや OLTP では値を下げ、ユーザへ早く返すか、更新をあとで処理するキューに載せる。データウェアハウスの変換段のように、他の大きな INSERT/UPDATE の完了を待つバックエンドでは値を上げてよい。この変数は InnoDB の行ロックにだけ効く。MySQL の表ロックは InnoDB の外なので、表ロック待ちには効かない。`SET GLOBAL` と `SET SESSION` で実行時に変えられる。GLOBAL はそれ以降の新規接続に効き、SESSION はそのクライアントだけに効く。 — [MySQL 8.4 Reference Manual, innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html)
- 同じ 8.4 本文の断片: `innodb_deadlock_detect` が無効のとき、デッドロック時のトランザクションのロールバックは `innodb_lock_wait_timeout` に頼る。検出が有効なときの直後の文は「デッドロックを直ちに検出し、デッドロックしたトランザクションの一方をロールバックする」と読める（ツール取得は文の途中から）。 — [MySQL 8.4 Reference Manual, innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html)
- Stack Overflow のコメントが MySQL 8.0 マニュアルを引用: 「`innodb_deadlock_detect` が有効（既定）のとき、ロック待ちタイムアウト値はデッドロックには適用されない」。 — [Stack Overflow, When is innodb_lock_wait_timeout ignored?](https://stackoverflow.com/questions/3751987/when-is-innodb-lock-wait-timeout-ignored)
- 2026年9月に公開された測定記事は、設定ファイル無しの `mysqld --no-defaults` で MySQL 8.0.46 と MariaDB 10.11.14 の `innodb_lock_wait_timeout` がどちらも 50 秒だった、と書いている。配布物やホスティングが変えることは同記事も注意している。 — [Copahost, 2026-09-28](https://www.copahost.com/blog/mysql-mariadb-default-settings/)

### Inferences
- 2004年11月のマニュアルは、4.0.20 より前の挙動を説明するためにタイムアウトの存在を前提にしている。したがって変数と「既定 50 秒」は 4.0.20（2004-05-18）より前から文書化されていた、と読んでよい。ただし 2001年3月の最初の同梱ビルドのソース定数までは追っていない。
- 最大 1073741824 秒は 2^30 秒で、365.25 日の年に直すと約 34.0 年。これはマニュアルの文言ではなく計算である。実質「ほぼ無制限」にできる上限で、50 は上限側ではなく短い側の既定である。
- 4.x のマニュアルは Startup Options の節に置いている。8.4 ではセッション単位で動的に変えられる。いつセッション動的になったかは、この調査では版を特定していない。

### Gaps
- 50 を選んだ設計メモ、Heikki のコメント、メーリングリストは見つかっていない。「最初から 50」は「見つかった最古のマニュアルが 50」であり、「2001年のソースを読んで 50 だった」ではない。
- 古い記憶にある「最初は別の既定だった」を裏づける差分は無い。否定も、2001年のソースを見るまでは完了しない。
- MySQL 5.0 のパラメータ頁 URL は、この環境では 9.7 の現行マニュアルへ落ちた。5.0 当時の HTML そのものは再取得していない。

## なぜ待ちのタイムアウトが必要か。デッドロック検出では足りない

### Takeaway
デッドロック検出が切るのは「互いが相手のロックを待つ輪」だけである。誰も待たずにロックを持ち続けるトランザクション（遅い処理、COMMIT しない接続）の後ろに並ぶ列は輪ではないので、検出は動かない。行ロックは文が終わっても残る設計なので、待ち人を無限に座らせない秒数が要る。2005年に Heikki が変えたのは、その秒数が来たとき「トランザクション全部」ではなく「直前の SQL 文だけ」を戻す挙動である。デッドロック（エラーの種類が違う）は今もトランザクション全体を戻す。

### Cited Findings
- 4.0/4.1 マニュアルの自分の言葉: タイムアウトが役立つのは、InnoDB が見つけられないデッドロック（当時は `LOCK TABLES` や他エンジン）である。見つけられるデッドロックはロック表を見て自分でロールバックする。 — [mysql.info.gz InnoDB start](http://osr507doc.xinuos.com/cgi-bin/info2html?(mysql.info.gz)InnoDB%20start&lang=en)
- MySQL 8.4 のデッドロック検出: 検出が有効（既定）なら、トランザクションのデッドロックを自動で見つけ、トランザクションをロールバックして輪を壊す。犠牲にするのは、挿入・更新・削除した行数が小さい方を選ぼうとする。`innodb_table_locks = 1`（既定）かつ `autocommit = 0` なら表ロックも意識する。そうでなければ、`LOCK TABLES` や InnoDB 以外のエンジンのロックが混ざったデッドロックは検出できない。その状況は `innodb_lock_wait_timeout` で解く、とマニュアルは書く。高競合で同じロックを大勢が待つと検出自体が遅くなるので、検出を切り、デッドロック時のロールバックを `innodb_lock_wait_timeout` に任せる選択肢がある。切る変数が `innodb_deadlock_detect`。 — [MySQL 8.4, Deadlock Detection](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlock-detection.html)
- 8.4 の変数説明は、待ちの対象を「他の InnoDB トランザクションがロックしている行」と書き、タイムアウトをエラー 1205 にしている。検出されたデッドロックにはタイムアウトを適用しない、という 8.0 マニュアル文は Stack Overflow が引用している（上節）。 — [8.4 パラメータ](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html); [Stack Overflow](https://stackoverflow.com/questions/3751987/when-is-innodb-lock-wait-timeout-ignored)
- Bug #12308（起票 2005-08-02、Michael Widenius）。タイトルは「InnoDB はロックのタイムアウトでトランザクション全体をロールバックする」。Widenius の主張: ロック待ちタイムアウトやデッドロックで全体を戻すと、他のエラー（直前の文だけ戻る）と違い、アプリケーションが使いにくい。直前の文を再実行する方が、トランザクションを最初からやり直すより易しい。 — [Bug #12308](https://bugs.mysql.com/bug.php?id=12308)
- 同バグ、2005-08-02 の Heikki Tuuri: デッドロックを解くにはトランザクション全体を戻す必要がある。InnoDB はロックをビットマップに置き、どのビットがどの SQL 文で立ったかを覚えない。だから直前の文のロックだけを放すことはできない。直前の文のロックだけ放して同じ文を再試行すると、デッドロックを作り直すことが多い。SQL 標準はデッドロック解決でトランザクション全体のロールバックを許しているように見える。デッドロックでもロック待ちタイムアウトでも、挙動を変える理由があるか。利用者は文句を言っていない、と書いている。 — [Bug #12308](https://bugs.mysql.com/bug.php?id=12308)
- 同バグ、2005-09-04 の Heikki Tuuri: 「Fixed in 5.0.13. Now a lock wait timeout only rolls back the latest SQL statement. Previously, it rolled back the whole transaction.」ドキュメント向けの置換文: デッドロックはトランザクション全体をロールバックする。MySQL 5.0.12 以前はロック待ちタイムアウトも全体を戻した。5.0.13 からは直前の SQL 文だけを戻す。 — [Bug #12308](https://bugs.mysql.com/bug.php?id=12308)
- 2005-09-08、Paul DuBois が 5.0.13 の変更履歴に載せ、エラー処理の節を Heikki の文に直した、とバグに記録している。 — [Bug #12308](https://bugs.mysql.com/bug.php?id=12308)
- Stack Overflow（2010-03-08）が当時の 5.0 マニュアルを引用: ロック待ちタイムアウトでは現在の文は実行されず、現在のトランザクションはロールバックされない。MySQL 5.0.13 まではタイムアウトでトランザクション全体を戻していた。古い挙動に戻す `--innodb_rollback_on_timeout` は MySQL 5.0.32 以降で使える、とその引用は言う。 — [Stack Overflow, innodb lock wait timeout](https://stackoverflow.com/questions/2400744/innodb-lock-wait-timeout)
- 8.4 マニュアルは今も、タイムアウトで戻るのは文であり、全体を戻すには `--innodb-rollback-on-timeout` だと書く。5.0.32 という導入版は 8.4 のその段落には無い。 — [8.4 パラメータ](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html)
- 5.7 のレプリケーション抜粋: レプリカの SQL スレッドが InnoDB のデッドロック、または `innodb_lock_wait_timeout` 超過でトランザクションを実行できないとき、`slave_transaction_retries` 回（既定 10）再試行してからエラー停止する。 — [MySQL 5.7 Replication Retries and Timeouts](https://dev.mysql.com/doc/mysql-replication-excerpt/5.7/en/replication-features-timeout.html)
- 8.4 のロック章はギャップロックを定義する。索引レコードのあいだ、または先頭の前・末尾の後の隙間へのロック。例は `SELECT c1 FROM t WHERE c1 BETWEEN 10 AND 20`。一意索引で一意な行を探す文にはギャップロックは不要、とも書く。競合するギャップロックを別トランザクションが同時に持てる、という注意もある。 — [MySQL 8.4 InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- DBA Stack Exchange で Rick James は、質問（MySQL 5.0.68 でロック待ちの最大時間をどこで決めるか）に対し、「50秒の InnoDB タイムアウトに当たったなら、アプリケーションの設計が悪い。そのタイムアウトは、ほかの方法では捕まえられない行儀の悪いものを捕まえるためにある」と答えている。返信者は `ALTER TABLE ... EXCHANGE PARTITION` を使っている、と書いている（この操作は長い正当な処理の例として出ている）。ページ本文の再取得は失敗し、文言は検索結果に繰り返し出た本文による。 — [DBA Stack Exchange, Set lock wait timeout](https://dba.stackexchange.com/questions/2763/set-lock-wait-timeout-in-mysql)

### Inferences
- 再話用。デッドロックは「A が持っている鉛筆を B が待ち、B が持っている消しゴムを A が待つ」輪である。InnoDB はその輪を見て、小さい方の作業をなかったことにする（マニュアルは行数で小ささを見る、と書く）。タイムアウトが要るのは輪ではないときである。A が鉛筆を持ったまま席を外し、B はただ列に並んでいる。A は誰も待っていないので輪が無く、検出は鳴らない。B を無限に立たせないため、50 秒で B の「今の一文」だけを失敗させる（5.0.13 以降）。A の鉛筆はそのままである。
- 5.0.13 の変更は「待ちが長すぎる」問題と「どこまで取り消すか」問題を分けた。Heikki が 2005-08-02 に書いた制約は今もデッドロック側に残る。ビットマップは文ごとのロックを覚えないので、輪を壊すにはトランザクションごと捨てる。タイムアウト側だけ「直前の文」にした結果、失敗した文より前に取ったロックは残る。アプリケーションが 1205 のあと COMMIT すると、途中までの変更が確定し得る。全体を原子にしたければ、アプリケーションが ROLLBACK するか、`--innodb-rollback-on-timeout` が要る。これは 8.4 マニュアルの「文だけ戻る」と、Heikki の「ビットマップ」説明を合わせた読みである。
- 接続枠が有限であること、OS が無限待ちを嫌うことは、タイムアウトの必要性としてもっともらしいが、InnoDB のマニュアルはそこを 50 秒の理由としては書いていない。マニュアルが明示するのは (1) 見えないデッドロックの尻ぬぐい、(2) 行ロックを待ちすぎないこと、(3) 検出を切ったときの代替、である。
- 古典レプリカの SQL 適用スレッドは 1205 で最大 10 回再試行する、という 5.7 の記述がある。Aurora のライターがロックを取るインスタンスであり、リーダーの適用が同じ変数で待つかは、この調査では Aurora のドキュメントを開いていない。ここでは主張しない。

### Gaps
- 「ロックは COMMIT/ROLLBACK まで保持される」という 8.4 ロック章の一文は、パターン検索ではヒットしなかった。行ロックが文の終了では終わらないことは、5.7 マニュアルの「1行を長くロックできる」、タイムアウトの主体が「トランザクション」、MyISAM 側の待ちが「文が終わるまで」であることから読む。正確な一文の再引用はしていない。
- `wait_timeout` / `interactive_timeout` の既定 28800 秒（8時間）は Stack Exchange の回答と Copahost の 8.0.46 測定にはあるが、公式の変数頁は開いていない。アイドル接続がいつ切れるかの公式秒数は、このノートでは確定扱いにしない。
- `max_connections` 既定 151 も Copahost の測定のみ。待ちが接続枠を食う、は推論にとどめる。
- 5.0.32 で `--innodb_rollback_on_timeout` が追加された、は Stack Overflow が引用した 5.0 マニュアル依存で、変更履歴原文は見ていない。5.0.13 の挙動変更は Heikki のバグコメントが一次。
- 外部キー検査が子表の大量行で共有ロックを長く持つか、は今回の取得範囲に公式の記述が無かった。ギャップロックの存在は 8.4 で確認した。ギャップ待ちも「InnoDB の行ロック待ち」に含まれる、はマニュアルの「row locks」という語からの読みであり、ギャップ専用の別タイマーは 8.4 の変数説明には無い。

## MySQL 5.5.3 のメタデータロックと、1年の `lock_wait_timeout`。5.1 で既定になったのか

### Takeaway
行ロックの 50 秒と、表定義（メタデータ）の待ちは別物である。メタデータロックは MySQL 5.5.3（2010-03-24）で、開いているトランザクションが終わるまで DDL を待たせるために入った。その待ちの `lock_wait_timeout` の既定は 31536000 秒（1年）で、8.4 でも同じである。InnoDB の 50 秒には従わない。1年にした理由として、MDL を実装した Konstantin Osipov は「5.1 にはこのタイムアウトが無く、互換のため」と 2013年にコメントしている。InnoDB がサーバの既定エンジンになったのは 5.1 ではなく 5.5 である。

### Cited Findings
- O'Reilly『MySQL Troubleshooting』（2012）: 表を使っているトランザクションがあるあいだ DDL を止めるため、5.5.3 からメタデータロックを使う。トランザクション開始時に使用表のメタデータロックを取り、終了時に放す。定義を変えようとする他スレッドは、トランザクションが終わるまで待つ。5.5.3 より前の DDL は並行トランザクションを知らず、例として、トランザクションが SELECT したあと別接続が DROP すると、戻って SELECT した側が「表が無い」（ERROR 1146）になり得た、と書く。 — [O'Reilly, Metadata Locking](https://www.oreilly.com/library/view/mysql-troubleshooting/9781449317836/ch02s04.html)
- MySQL 5.5.3 の変更履歴（2010-03-24, Milestone 3）のミラー: 他セッションのアクティブなトランザクションが使っている表に DDL が走ると、バイナリログの順序が狂い得た。例は、トランザクションが使っている表への DROP TABLE。これを、トランザクション内で使った表のメタデータロック解放をトランザクション終了まで遅らせることで防いだ、とある。 — [MySQL Server Version Reference, 5.5.3（2010-03-24）](http://ftp.ncnu.edu.tw/Unix/Database/MySQL/doc/mysqld-version-reference/en/ch05s04s01.html)
- 同じ 5.5.3 変更履歴の Archive 版は、メタデータロックを取ったトランザクション中の RESET がデバッグビルドでアサートした件、LOCK TABLES 中の DROP DATABASE と CREATE DATABASE のデッドロック、LOCK TABLES 中は CREATE TABLE を禁止、など、メタデータロック導入に伴う非互換を列挙している。 — [Changes in MySQL 5.5.3（Internet Archive）](https://web.archive.org/web/20190201033750/https://dev.mysql.com/doc/relnotes/mysql/5.5/en/news-5-5-3.html)
- Konstantin Osipov（Kostja）の 2011-04-15 の講演メモ: 彼は MySQL のサーバエンジニアとして 5.5 の実現に関わり、講演は 5.5 のサーバロックの変更についてだと書く。メタデータロックは、全エンジンと全参加者が使える独立した仕組みとして必要だった。ALTER/DROP/CREATE の表、RENAME DATABASE のスキーマ、SET GLOBAL READ ONLY のようなインスタンス全体を、一つの仕組みで守る、と説明している。 — [kostja_osipov, 2011-04-15](https://kostja-osipov.livejournal.com/35982.html)
- Daniel Nichter（Percona、2013-02-28）: MySQL 5.5 の `lock_wait_timeout` 文書を引用し、メタデータロック取得のタイムアウトで、許容範囲は 1 から 31536000（1年）、既定は 31536000 だと言う。彼の評価は「既定で 1 年メタブロックするのは、自分の意見ではバグ。`innodb_lock_wait_timeout` の 50 秒は妥当だが 31536000 は妥当ではない」。変数名が `metadata_lock_wait_timeout` でないことにも文句を書いている。 — [Percona, 2013-02-28](https://www.percona.com/blog/mysql-5-5-lock_wait_timeout-patience-is-a-virtue-and-a-locked-server/)
- 同記事への Kostja のコメント（ページ上は「13 years ago」、記事公開が 2013-02-28 なので 2013年頃）: ALTER TABLE は 1 年は走らないが 1 週間にはなり得るので、1時間のロック待ちタイムアウトはバグになり得る。`lock_wait_timeout` はすべてのロック、とくにグローバル読み取りロックと MyISAM ロックに効くべきだ。InnoDB が `lock_wait_timeout` に従わないのは InnoDB 側のバグであり、`lock_wait_timeout` のバグではない。既定を何にするかは主観で、1年はタイムアウトの無かった 5.1 との互換のために選んだ。ほかの値にしても批判されただろう、と書く。 — [Percona, 2013-02-28 のコメント](https://www.percona.com/blog/mysql-5-5-lock_wait_timeout-patience-is-a-virtue-and-a-locked-server/)
- MySQL 8.4 マニュアルは今も `lock_wait_timeout` の Default Value を 31536000、最小 1、最大 31536000、単位秒とする。メタデータロックを取る試行のタイムアウトである。値はロック試行ごとに別々に適用され、一つの文が複数ロックを要すると、タイムアウトエラーを出す前に `lock_wait_timeout` より長くブロックし得る。タイムアウト時は `ER_LOCK_WAIT_TIMEOUT`。 — [MySQL 8.4 server system variables](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html)
- 8.4 の `innodb_lock_wait_timeout` は「MySQL の表ロック待ちには適用されない」と明示する。 — [8.4 innodb parameters](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html)
- MySQL Bug #119422（起票 2025-11-18、Hongju Yang、対象 8.0.x / 8.4.x、状態 Verified は 2025-11-19）: 既定 31536000 は事実上タイムアウト無しで、隠れたメタデータロックの停止を起こす、として 1日か 1週間への引き下げを提案。MariaDB は 10.2 で 31536000 から 86400（1日）へ変えた、と同バグは MariaDB の文書を引用して書く。根拠の一つに Nichter の 2013年記事を挙げ、1年は古い版との互換が主目的だった、と要約している。 — [MySQL Bug #119422](https://bugs.mysql.com/bug.php?id=119422)
- 既定エンジンは 5.5。2010-09-20 の RC 報道と 2010-12-15 の GA 告知が「InnoDB が既定」と書いている（第1節）。5.1 で既定になった、という公式告知はこの調査では見つかっていない。

### Inferences
- 再話用。行の鉛筆（InnoDB、50秒）と、目録カード（表の定義、最長1年）は別のカウンターである。5.5.3 以前は、誰かが本を開いたままなのに、別の人が目録からその本のカードを捨てられた（DROP が先行トランザクションを知らない）。5.5.3 は「本を読み終わってからカードを捨てる」にした。その待ちに短いベルを付けると、5.1 で何時間もかかる ALTER が突然失敗するので、Osipov のコメントによればベルを 1年、つまり以前の「ベル無し」に近づけた。Nichter はそれを実務上危険だと批判した。両方とも意見であり、コード上の既定が 1年であることは 8.4 マニュアルが事実として残している。
- したがって「ロック待ちを 50 秒より長くしたい」がメタデータ待ち（ALTER、LOCK TABLES 由来の MDL）だった場合、`innodb_lock_wait_timeout` を上げても効かない。8.4 がそう書いている。

### Gaps
- 「1年は 5.1 互換」は Osipov のブログコメントであり、5.5 の公式変更履歴の文としては見ていない。実装者の同時代説明として強く、リリースノートの文言ではない。
- MariaDB 10.2 の 86400 への変更は Bug #119422 の主張と、そこが指す MariaDB 文書に依存する。MariaDB のアップグレードノート自体は開いていない。
- 5.1 InnoDB Plugin の公開日と「組み込み InnoDB とは別物だった」という公式発表文は未取得。5.5 で初めて既定エンジンになった、という告知とは矛盾しないが、Plugin の履歴は別ノートが要る。

## MySQL 5.6 / 5.7。オンライン DDL と `innodb_deadlock_detect`

### Takeaway
`innodb_deadlock_detect` は MySQL 5.7.15（2016-09-06）で追加された。高競合で検出が遅いとき、検出を切り、デッドロックの尻ぬぐいを 50 秒（または設定した `innodb_lock_wait_timeout`）に戻すためのスイッチである。検出を切ると、本当の輪も 1205 まで待たされる。MySQL 5.6.10 の GA は 2013-02-05。オンライン DDL の章はこの調査では開いていない。

### Cited Findings
- Oracle の MySQL 8.0 FAQ（ページの掲載日メタデータは 2026-05-05）: MySQL 5.6 は 5.6.10 で GA、本番向けリリース日は 2013年2月5日。MySQL 5.7 は 5.7.9 で GA、2015年10月21日。5.7 の開発終了は 5.7.44、2023年10月25日。 — [A.1 MySQL 8.0 FAQ: General](https://docs.oracle.com/cd/E17952_01/mysql-8.0-en/faqs-general.html)
- MySQL 5.7.15 リリースノート（2016-09-06, General Availability）: 新しい動的オプション `innodb_deadlock_detect` でデッドロック検出を無効にできる。高競合で多数のスレッドが同じロックを待つと検出が遅くなる。検出を切り、デッドロック時のロールバックを `innodb_lock_wait_timeout` に任せる方が効率的なことがある。(Bug #23477773, WL #9383)。 — [Changes in MySQL 5.7.15](https://docs.oracle.com/cd/E17952_01/mysql-5.7-relnotes-en/news-5-7-15.html)
- 8.4 マニュアルは同じ説明を維持し、無効化に `innodb_deadlock_detect` を使うと書く。 — [MySQL 8.4 Deadlock Detection](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlock-detection.html)
- アリババ雲の開発者記事（2016-09-25）は、5.7.15 と MySQL 8.0.0 でこのスイッチが入った、と書く。5.7.15 は上記リリースノートと一致する。8.0.0 という版は、この記事以外では突き合わせていない。 — [阿里云开发者社区, 2016-09-25](https://developer.aliyun.com/article/60941)

### Inferences
- 再話用。輪の検査は、並んでいる人が増えると検査そのものが列を遅くする。5.7.15 は「検査をやめて、50秒のベルだけにする」スイッチを公式にした。検査をやめると、本当に輪になっている二人も、ベルまで立ち続ける。マニュアルはそれを性能のための選択として書き、いつでも使えとは書いていない。
- オンライン DDL は 5.6 の代表機能として広く知られるが、この調査の取得範囲では「5.6 で入った」という公式の文を引用できない。5.6.10 の日付だけが FAQ で確認できた、と分ける。

### Gaps
- MySQL 5.6 のオンライン DDL の章、ALGORITHM=INPLACE の導入版、開始時・終了時にメタデータロックを短く取る、という説明は未取得。
- `innodb_deadlock_detect` の既定が最初から ON だったかは、5.7.15 ノートが「無効にできる」と書いていることから ON が通常側だと読める。8.4 は「検出が有効（既定）」と書いている。5.7.15 当日の既定値の行はノート本文の抜粋には無かった。

## MySQL 8.0 と 8.4 LTS。既定 50 は残ったか。2026-10-08 時点のパッチ

### Takeaway
8.0 の GA は 8.0.11、2018年4月19日。`INFORMATION_SCHEMA.INNODB_LOCKS` はマニュアル上 8.0.1 で削除され、代わりが `performance_schema.data_locks` である。8.4.0 LTS は 2024年4月30日。8.4 で変わった既定値の公式一覧に `innodb_lock_wait_timeout` は無く、8.4 マニュアルの既定は 50 のままである。リリースノート（文書生成 2026-09-24）では、サーバとしての新しい版は 8.4.11（2026-07-28）。8.4.12（2026-08-18）は Docker イメージだけの重要セキュリティ修正、とノート自身が断っている。利用者環境の 8.4.7 は 2025-10-21 の版で、それより新しい 8.4.8 以降がノートに存在する。8.4.7 のノートを変数名で検索しても、このタイムアウトの変更はヒットしなかった。

### Cited Findings
- FAQ: MySQL 8.0 は 8.0.11 で GA、本番向けは 2018年4月19日。8.0.34 から、Innovation と LTS の導入に伴いバグ修正シリーズになった。MySQL 8 Innovation は 8.1.0（2023年7月18日）に始まり、8.3.0（2024年1月16日）で開発終了。MySQL 8.4 LTS は 8.4.0、2024年4月30日に始まる。MySQL 9 Innovation は 9.0.0、2024年7月1日。FAQ は本番利用の現行として 9.6、8.4、8.0 を挙げる（この FAQ 頁の「Published」は 2026-05-05）。 — [A.1 MySQL 8.0 FAQ: General](https://docs.oracle.com/cd/E17952_01/mysql-8.0-en/faqs-general.html)
- Mike Frank（Oracle MySQL Product Management、ブログは 2024-05-07 付け）: 2024年4月30日時点の GA として MySQL Server 8.4.0 LTS と 8.0.37 を公開。2023年夏に LTS と Innovation へ移る計画を発表し、約2年ごとにメジャー LTS、それ以降 8.0.34+ はバグとセキュリティのみ、Innovation は 8.1、8.2、8.3、その準備のうえ最初の LTS が 8.4.0、と書く。8.4.0 の一般項目に「サーバ既定の洗練」とある。InnoDB 項はバルクロードで、`innodb_lock_wait_timeout` には触れていない（取得した範囲）。 — [MySQL April 2024 GA Releases, 2024-05-07](https://blogs.oracle.com/mysql/mysql-april-2024-ga-releases-now-available)
- Oracle MySQL ブログ（2024-11-28）: 8.4 は 8.0 と比べて組み込み設定の既定を変えた。何も削除しておらず、8.0 の値を設定ファイルに書けば使える。変更表の行として取得できたのは `innodb_adaptive_hash_index`（ON から OFF）、`innodb_buffer_pool_in_core_file`、`innodb_buffer_pool_instances`、`innodb_page_cleaners` など。`innodb_buffer_pool_size` 128MB と `innodb_redo_log_capacity` 1GB は 8.0 と 8.4 で維持、とも書く。取得した表と本文に `innodb_lock_wait_timeout` は無い。 — [Auto Adapting Configuration Parameters in MySQL, 2024-11-28](https://blogs.oracle.com/mysql/auto-adapting-configuration-parameters-in-mysql)
- 8.4 リファレンスは `innodb_lock_wait_timeout` の Default Value を 50 とする（第2節の引用）。`lock_wait_timeout` は 31536000 のまま（第4節）。
- MySQL 8.0 の `performance_schema.data_locks` の説明: MySQL 8.0.1 より前は似た情報が `INFORMATION_SCHEMA.INNODB_LOCKS` にあった。こちらは、要求したがまだ持っていないロックと、他トランザクションをブロックしている保持ロックだけを示す。`INNODB_LOCKS` は非推奨で、8.0.1 で削除。代わりに `data_locks`。違い: 誰も待っていなくても `data_locks` は保持ロックを出す。`LOCK_SPACE` / `LOCK_PAGE` / `LOCK_REC` に相当する列は無い。 — [MySQL 8.0, The data_locks Table](https://docs.oracle.com/cd/E17952_01/mysql-8.0-en/performance-schema-data-locks-table.html)
- 8.4 リリースノート PDF（文書生成 2026-09-24、revision 31620）の目次: 8.4.12（2026-08-18）、8.4.11（2026-07-28）、8.4.10（2026-06-16）、8.4.9（2026-04-21）、8.4.8（2026-01-20）、8.4.7（2025-10-21）、8.4.6（2025-07-22）、8.4.5（2025-04-15）、8.4.4（2025-01-21）、8.4.3（2024-10-15）、8.4.2（2024-07-23）、8.4.1（2024-07-01）。本文: 8.4.12 は「MySQL Server docker image だけの Critical Security Patch Update」で、2026年8月の Oracle CPU 勧告を見よ、と注記する。8.4.11（2026-07-28）は Audit Log や Compilation の節を持つ通常の変更節に見える。8.4.10（2026-06-16）は Critical Security Patch Update の注記。 — [MySQL 8.4 Release Notes PDF](https://downloads.mysql.com/docs/mysql-8.4-relnotes-en.pdf)
- 8.4.7 の HTML リリースノートはタイトルが「Changes in MySQL 8.4.7 (2025-10-21)」。この頁を `innodb_lock_wait_timeout` で検索してもヒットは無く、ヒットした “Default” は `PARTITION BY ... (DEFAULT (column))` のバグ修正だった。 — [Changes in MySQL 8.4.7](https://dev.mysql.com/doc/relnotes/mysql/8.4/en/news-8-4-7.html)
- endoflife.ai は 8.4 の最新パッチを 8.4.11、公開を 2026-06-30、EOL を 2032-04-30 とする。公式ノートの 8.4.11 は 2026-07-28 で、8.4.12（Docker 限定、2026-08-18）も存在する。日付と「最新」の定義が公式ノートと食い違う。EOL 日は公式ライフサイクル頁では再確認していない。 — [endoflife.ai MySQL 8.4](https://endoflife.ai/mysql/8.4) （公式ノートと矛盾するため、版の事実は PDF を優先）
- AWS は 2026-05-21 に Amazon Aurora MySQL 8.4 の一般提供を告知し、コミュニティ MySQL 8.4.7 互換だと書いている。バージョン表記 `8.4.7` は Aurora ではメジャー `8.4`、マイナー `7`、という説明。 — [AWS Database Blog, 2026-05-21](https://aws.amazon.com/blogs/database/amazon-aurora-mysql-8-4-is-now-generally-available/)

### Inferences
- 8.0 で変わったのは「ロックを見る窓」であり、50 秒というベルではない。古い `INNODB_LOCKS` は「誰かが待っているロック」しか出さない。`data_locks` は持っているロックも出す。待ちの原因を見る場所が移った、という物語は子どもにも話せるが、タイムアウトの長さの変更ではない。
- 8.4 の既定見直し記事がこの変数を表に出さず、リファレンスが 50 と書いているので、8.4.0 の「既定の洗練」で 50 秒は対象外だった、と判断してよい。8.4.8 から 8.4.11 の全文を 1 件ずつ「既定値変更」で通読してはいない。
- 2026-05-21 時点の Aurora MySQL 8.4 GA はコミュニティ 8.4.7 互換、という AWS の文はある。Aurora が `innodb_lock_wait_timeout` の既定を 50 以外にしているかは未確認。

### Gaps
- 8.0 のトランザクショナルデータディクショナリの導入を、8.0 の “What’s New” から引用してはいない。ロック可視性の変更（`INNODB_LOCKS` 削除）は引用した。
- 8.0.1 のリリース日は未確認。削除が「8.0.1 から」であることは 8.0 マニュアルが書き、GA の 8.0.11（2018-04-19）より前の開発版での削除である。
- 8.4 の Premier/Extended Support の終了日は公式ライフサイクル文書を開いていない。endoflife.ai の 2032-04-30 は、版日付の矛盾があるサイトなので、ここでは採用しない。
- Aurora 固有のパラメータグループ既定は未調査。

## 現場は 50 秒を下げるのか、上げるのか

### Takeaway
公式マニュアルは二つの方向を両方書く。OLTP や対話処理では下げ、ウェアハウスの長い変換では上げる。Rick James は「50秒に当たること自体がアプリケーションの欠陥」と言う。二次資料は OLTP で 5〜10 秒、バッチはセッションだけ上げる、と勧める。これは調査された業界統計ではなく、マニュアルと著名な DBA の意見と、2026年の解説記事である。グローバルに上げて長いトランザクションを隠すのは、マニュアルが勧める「バックエンドだけ上げる」とは別の使い方である。

### Cited Findings
- 8.4 マニュアル: OLTP や対話型では下げ、フィードバックを早くするか、更新をキューに回す。ウェアハウスの変換段のように、他の大きな INSERT/UPDATE を待つ処理では上げる。 — [8.4 innodb parameters](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html)
- Rick James（DBA Stack Exchange）: 50秒に当たるなら設計が悪い。このタイムアウトは、ほかでは捕まえられない行儀の悪いものを捕まえるためにある。同じスレッドの返信は、パーティション交換 `ALTER TABLE ... EXCHANGE PARTITION` のあとパーティションを外す、という長い操作に触れている。 — [DBA Stack Exchange](https://dba.stackexchange.com/questions/2763/set-lock-wait-timeout-in-mysql)
- neverblink の 2026-09-22 の解説（二次、運用ガイド）: 既定 50 秒はほとんどの OLTP には長すぎ、5〜10 秒はフェイルファストで問題が早く見える。バッチはグローバル既定を上げず、セッション値を上げるべきだ、と書く。1205 は「誰かがロックを長く持ちすぎた」で、1213 はデッドロック、とも区別している。`performance_schema.data_locks` と `data_lock_waits` が 8.0 以降の正準な観測表で、古い `INNODB_LOCKS` / `INNODB_LOCK_WAITS` は 8.0 で削除、とも書く（削除自体は Oracle マニュアルと一致）。 — [neverblink, 2026-09-22](https://neverblink.ai/kb/mysql-innodb-lock-wait-timeout-exceeded)
- Copahost（2026-09-28、8.0.46 を実測した記事）: 1205 は、他トランザクションがロックした行を 50秒より長く待った、という意味だ。`innodb_lock_wait_timeout` を上げてもたいてい助からない。本体はロックを長く持つトランザクション（遅い更新、トランザクションを開いたまま待つプロセス）だ、と書く。 — [Copahost](https://www.copahost.com/blog/mysql-mariadb-default-settings/)
- 公式が「上げてよい」とする例は、ウェアハウスの変換が他の大きな書き込みを待つ場合、に限って引用できる。パーティション交換は Stack Exchange の利用者が長い操作として挙げているが、Oracle がそれを 50 秒超過の正当例として保証しているわけではない。

### Inferences
- 緊張は実在する。公式は「短くする人」と「長くする人」の両方を想定している。OLTP でグローバルを 50 より大きくするのは、マニュアルの OLTP 文（下げよ）と逆で、James や 2026年の解説が「設計問題を隠す」と言う側に近い。バッチや移行やパーティション交換を、そのセッションだけ長くするのは、マニュアルの「バックエンドでは上げよ」と同じ方向である。
- 5〜10 秒は neverblink の推奨値であり、Oracle の既定でも、測定された中央値でもない。再話で「みんなが 5 秒にしている」とは言えない。「短い仕事の店はベルを早め、倉庫の夜間作業はベルを遅らせる。店のベルを倉庫に合わせると、待たされている客が 50秒より長く立たされる」が、マニュアルの二文に忠実である。
- 誤診の型: 1205 が出るたびにグローバルを上げる。ロックを持ったまま止まっている側は、待ち人のタイマーを伸ばしても COMMIT しない。5.0.13 以降、1205 は待ち人の直前の文だけを失敗させ、持ち手のトランザクションはそのままなので、タイマー延長は持ち手を救わない。

### Gaps
- 本番の OLTP が実際に何秒にしているかの調査（Percona のアンケート等）は、この調査では取っていない。
- 外部キーの親削除、大きな範囲更新のギャップロック、移行、を「50秒を超えてよい」と公式が列挙しているわけではない。公式の上げる例はウェアハウスの変換段である。
- Aurora ライターでレプリカ適用がこのタイマーに当たるかは未確認。古典 MySQL の SQL スレッドが 1205 で再試行することは第3節のとおり。

## Oracle Database との比較（ソースがある範囲だけ）

### Takeaway
Oracle の通常の行ロック（DML）には、InnoDB の 50 秒に相当するサーバ既定タイムアウトは、当たった Ask Tom の回答には無い。待ちを切るのは `SELECT ... FOR UPDATE WAIT n` のように SQL が頼んだときである。`distributed_lock_timeout` はデータベースリンク越しの分散トランザクション用で、Ask Tom は「普通のロック待ちにはおそらく違う」と答えている。ある 11gR1 の例ではそのパラメータの表示値は 60 秒で、既定のまま（ISDEFAULT TRUE）だった。プロファイルの IDLE_TIME は未調査。

### Cited Findings
- Ask Tom（質問はロック待ちにタイムアウトを付けたい、`distributed_lock_timeout` を聞いている。ページの検索メタデータは 2018-10-06）: Chris の回答は、`distributed_lock_timeout` はデータベースリンク越しの文に関係するので、たぶん欲しいものではない、と言う。ブロックする DDL は `ddl_lock_timeout`（秒）で待てる。例は `ALTER SESSION SET ddl_lock_timeout = 10` のあと `ALTER TABLE ... DROP` が `ORA-00054: resource busy and acquire with NOWAIT specified or timeout expired`。非ブロックの DDL（列追加など）は、相手の COMMIT/ROLLBACK まで待つ、とも書く。DML については、UPDATE/DELETE の前に `SELECT ... FOR UPDATE WAIT n` を使い、n 秒でロックを取れなければ諦める。例のエラーは `ORA-30006: resource busy; acquire with WAIT timeout expired`。 — [Ask Tom, lock wait timeout](https://asktom.oracle.com/ords/asktom.search?tag=lock-wait-timeout)
- Ask Tom（2013-03-27、Tom Kyte）: `ORA-02049: timeout: distributed transaction waiting for lock` について。彼の 11gR1 セッションで `v$parameter` の `distributed_lock_timeout` は VALUE 60、ISDEFAULT TRUE、変更はオンライン不可。説明文は「number of seconds a distributed transaction waits for a lock」。大きくするには init.ora か spfile を変えて再起動する。エラーを出さないには、待ちたい最長より大きくする、と答える。 — [Ask Tom, distributed_lock_timeout](https://asktom.oracle.com/ords/f?p=100:11:0::::P11_QUESTION_ID:1308033500346496795)

### Inferences
- 再話用の対比は、ソースの範囲ではこう言える。InnoDB は「行を待つ人」にサーバ既定の 50 秒ベルを付ける。Ask Tom が説明する Oracle の DML は、SQL が `WAIT n` と言わない限り、持ち手が終わるまで待つ。分散トランザクションだけ別ベルがあり、示された 11gR1 の例では 60 秒だった。50 と 60 を「兄弟の設計」と結ぶ資料は無い。近い数字だが、対象が違う（InnoDB は普通の行ロック、Oracle の 60 秒例は分散トランザクション）。
- Oracle の `ddl_lock_timeout` は MySQL の `lock_wait_timeout`（メタデータ、既定1年）に役割が近い。ただし MySQL の既定は非常に長く、Ask Tom の例ではセッションが 10 秒を自分で置いている。既定値の対応表は作らない。`ddl_lock_timeout` の Oracle 既定は、この調査では確認していない。

### Gaps
- プロファイルの IDLE_TIME、リソースマネージャの待機制限は見ていない。比較に使わない。
- `distributed_lock_timeout` の 60 秒は、Tom Kyte がそのとき見た 11gR1 インスタンスの `v$parameter` であり、全バージョンの既定だとマニュアルで確認してはいない。
- enqueue 待ち（enq: TX - row lock contention）がブロッカ終了まで待つ、という待機イベント文書は開いていない。Ask Tom の「DML は FOR UPDATE WAIT を使え」が、通常 DML に短い既定タイムアウトが無いことの根拠である。

## 時系列（確認できた日付だけ）

### Takeaway
ベルは「行ロックを発明したあと、輪でない待ちを有限にするため」にあり、文書上の既定は見つかる限り 50 秒のままである。途中で変わったのは、鳴ったときに戻す範囲（5.0.13）、見えない表ロックへの対応（4.0.20 / 4.1.2）、メタデータという別ベル（5.5.3、1年）、検出を切るスイッチ（5.7.15）、ロックの見え方（8.0.1 で古いビュー削除）である。8.4 の既定見直しは 50 秒を動かした証拠が無い。

### Cited Findings
- 1995年: Heikki Tuuri が Innobase を設立（Wikipedia。一次の設立文書は未確認）。 — [Wikipedia](https://en.wikipedia.org/wiki/Innobase)
- 2000年9月: MySQL AB と協業開始（Wikipedia）。 — [Wikipedia](https://en.wikipedia.org/wiki/Innobase)
- 2001年3月: InnoDB 入りの MySQL をリリース（Wikipedia。版番号は未確認）。 — [Wikipedia](https://en.wikipedia.org/wiki/Innobase)
- 4.0/4.1 マニュアル文: `innodb_lock_wait_timeout` の既定 50 秒。用途は、InnoDB が見えないデッドロック。4.0.20 と 4.1.2 から `LOCK TABLES` に気づく、という書き方なので、変数自体はその版の新機能としては書かれていない。 — [mysql.info InnoDB start](http://osr507doc.xinuos.com/cgi-bin/info2html?(mysql.info.gz)InnoDB%20start&lang=en)
- 2004-05-18: MySQL/InnoDB-4.0.20（マニュアル目次の見出し日付）。2004-05-30: InnoDB-4.1.2。 — [4.1 マニュアル目次](http://marcinmazurek.com.pl/download/manual-4.1/index.html)
- 2004-11-03: フォーラム利用者が現行マニュアルの「default is 50 seconds」を引用。 — [MySQL Forums](https://forums.mysql.com/read.php?20,5095,5095)
- 2005-08-02: Widenius が Bug #12308 を起票。Heikki はビットマップのため文単位ではロックを戻せない、と返す。 — [Bug #12308](https://bugs.mysql.com/bug.php?id=12308)
- 2005-09-04: Heikki が「5.0.13 で修正。ロック待ちタイムアウトは直前の SQL 文だけを戻す。以前はトランザクション全体」と書く。デッドロックは全体のまま。 — [Bug #12308](https://bugs.mysql.com/bug.php?id=12308)
- 2005-10-07: Oracle が Innobase 買収を発表。 — [Oracle press release](https://web.archive.org/web/20181115071936/http://www.oracle.com/us/corporate/press/016679_EN)
- 2010年1月: Oracle が Sun を買収（Wikipedia、日は未確認）。 — [Wikipedia](https://en.wikipedia.org/wiki/Innobase)
- 2010-03-24: MySQL 5.5.3。トランザクション終了までメタデータロックを保持し、DDL が先行トランザクションを追い越さないようにする。 — [5.5.3 version reference](http://ftp.ncnu.edu.tw/Unix/Database/MySQL/doc/mysqld-version-reference/en/ch05s04s01.html)
- 2010-09 頃: MySQL 5.5 RC が InnoDB を既定に（Linux.com、2010-09-20）。2010-12-15: 5.5 GA 告知で既定エンジンが InnoDB。 — [Linux.com](https://www.linux.com/news/first-release-candidate-mysql-55-innodb-default/); [DBTA](https://www.dbta.com/Editorial/News-Flashes/MySQL-55-Now-Generally-Available-72995.aspx)
- 2013-02-05: MySQL 5.6.10 GA。 — [MySQL 8.0 FAQ](https://docs.oracle.com/cd/E17952_01/mysql-8.0-en/faqs-general.html)
- 2013-02-28: Nichter が `lock_wait_timeout` 既定 1年と `innodb_lock_wait_timeout` 50秒を対比。Kostja が 1年は 5.1 にタイムアウトが無かったための互換、とコメント。 — [Percona](https://www.percona.com/blog/mysql-5-5-lock_wait_timeout-patience-is-a-virtue-and-a-locked-server/)
- 2013-07-08: Innobase の事業終了（Wikipedia）。 — [Wikipedia](https://en.wikipedia.org/wiki/Innobase)
- 2015-10-21: MySQL 5.7.9 GA。 — [MySQL 8.0 FAQ](https://docs.oracle.com/cd/E17952_01/mysql-8.0-en/faqs-general.html)
- 2016-09-06: MySQL 5.7.15。`innodb_deadlock_detect` 追加。 — [5.7.15 release notes](https://docs.oracle.com/cd/E17952_01/mysql-5.7-relnotes-en/news-5-7-15.html)
- 8.0.1: `INFORMATION_SCHEMA.INNODB_LOCKS` を削除（マニュアル。リリース日は未確認）。2018-04-19: 8.0.11 GA。 — [data_locks](https://docs.oracle.com/cd/E17952_01/mysql-8.0-en/performance-schema-data-locks-table.html); [FAQ](https://docs.oracle.com/cd/E17952_01/mysql-8.0-en/faqs-general.html)
- 2023-07-18: 8.1.0。2024-01-16: 8.3.0 で 8 Innovation の開発終了。2024-04-30: 8.4.0 LTS。 — [FAQ](https://docs.oracle.com/cd/E17952_01/mysql-8.0-en/faqs-general.html); [Mike Frank, 2024-05-07](https://blogs.oracle.com/mysql/mysql-april-2024-ga-releases-now-available)
- 2024-11-28: 8.4 の既定変更の公式説明。この変数は変更表に無い。 — [Oracle MySQL blog](https://blogs.oracle.com/mysql/auto-adapting-configuration-parameters-in-mysql)
- 2025-10-21: MySQL 8.4.7。 — [8.4.7 release notes](https://dev.mysql.com/doc/relnotes/mysql/8.4/en/news-8-4-7.html)
- 2026-05-21: Aurora MySQL 8.4 GA、コミュニティ 8.4.7 互換、と AWS。 — [AWS Database Blog](https://aws.amazon.com/blogs/database/amazon-aurora-mysql-8-4-is-now-generally-available/)
- 2026-07-28: MySQL 8.4.11。2026-08-18: 8.4.12 は Docker イメージのみの CPU。ノート PDF の生成日は 2026-09-24。調査日 2026-10-08 時点で、8.4 マニュアルの行ロック待ち既定は 50 秒、メタデータ待ち既定は 31536000 秒。 — [8.4 Release Notes PDF](https://downloads.mysql.com/docs/mysql-8.4-relnotes-en.pdf); [8.4 parameters](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html); [8.4 system variables](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html)

### Inferences
- 子ども向けの一本の話にできる骨格は次のとおり。昔の MyISAM はノート全体を取り上げたので、別のページを書く人は「その文が終わるまで」待った。InnoDB は行だけを押さえ、トランザクションが終わるまで離さない。輪になっていれば検出がすぐ一人を戻す。輪でなく、持ち手が席を外しているだけなら、2004年のマニュアルがすでに書いていた 50 秒で、待っている側の今の文を失敗させる。2005年9月に Heikki は、その失敗でノート全部を消すのをやめ、直前の文だけ消すようにした。輪のときだけ全部消す。2010年、表の定義を変える待ちは別ベルになり、古い「ベル無し」に合わせるため 1年になった。2016年、輪の検査が混雑で遅くなるサーバは、検査をやめて 50 秒ベルに戻せるようになった。2018年以降、ロックの一覧の見方が変わったが、50 秒は 8.4 のマニュアルに残っている。50 という数字を選んだ手紙は見つかっていない。

### Gaps
- 2001年のソースツリーで定数が 50 だったかは未確認。
- 50 の設計根拠は未発見。無ければ「不明」が正確である。
