---
external: false
title: "Aurora MySQL から MySQL Community (RDS for MySQL) に移行した"
date: 2024-01-15
---

# この記事はなに？

勤務先のシステムのデータベースを Aurora から RDS for MySQL に移行したので、簡単なメモ書を残しておく。

# 背景

勤務先のシステムで採用していたのは Amazon Aurora MySQL v2 (MySQL Community 5.7 互換) 。MySQL 5.7 は 2020年にプレミアサポートが終了している古いバージョンで、2023年10月には延長サポートも終了。[1] 派生版の Aurora も追従してサポート終了の予定が発表されていることから、Aurora MySQL v3 (MySQL 8.0 互換) への移行を検討した。[2]

# 移行先の選定

移行先として考えられる選択肢は3つ。それぞれ検討したメモ。

## [△] Aurora v3 (MySQL 8.0 互換)

順当な移行先だが、コスト面から不採用とした。Aurora v2 で利用していた db.t3.small インスタンスクラスの継続使用ができず、より高額なインスタンスの選択が必須となるため。


Aurora v2 の最安インスタンスクラスは db.t3.small ($0.063) であるのに対して、v3 では db.t4g.medium ($0.113) が最安となる。勤務先システムの負荷は small インスタンスが過剰と考えられる程度にとどまることもあり、80% の費用増は許容できなかった。

## [○] MySQL Community 8.0

最小インスタンスクラスが db.t4g.micro ($0.05 - マルチAZ配置) であるので、パフォーマンスの低下を許容すれば費用の削減が可能。Aurora から MySQL Community への移行は RDS でサポートされていないことからコンソール外での手動操作が必要になってしまうが、煩雑な作業ではなかったので負担を許容して採用。

## [×] Aurora v2 の延長サポートを利用する

vCPU あたり $0.12 ドルの料金を負担すれば Amazon RDS による延長サポートの利用が可能。5.7 から 8.0 の非互換性の解決に時間を要する、かつ費用負担を許容できる場合には活用できそうだが、今回は非互換性の影響を受けないことをあらかじめ確認していたため検討せず。

# 移行作業

スキーマとデータは別々に移行。

## スキーマを移行
```shell
$ mysqldump --host <database>.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
		--user <user> --password \
		--all-databases --ignore-database mysql --ignore-database sys \
		--no-data --routines --events > dump-defs.sql

$ mysql --host <database>.xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
		--user <user> --password < dump-defs.sql
```

## データを移行
```shell
$ mysqldump --host <database>.cluster-xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
		--user <user> --password \
		--all-databases --ignore-database mysql --ignore-database sys \
		--no-create-info --routines --events > dump-data.sql

$ mysql --host <database>.xxxxxxxxxxxx.ap-northeast-1.rds.amazonaws.com \
		--user <user> --password < dump-data.sql
```

# 雑感

パフォーマンスと運用の柔軟性に優れる Aurora だが、ごく小規模のシステムだと費用負担が過大になることがある。立ち上げ期のスタートアップは RDS for MySQL の採用を優先して検討した方がいいように思う。MySQL から Aurora への移行は RDS のコンソール操作で可能なので負債になることもない。 

[1]: https://www.oracle.com/us/support/library/lsp-tech-chart-069290.pdf
[2]: https://aws.amazon.com/blogs/database/introducing-amazon-rds-extended-support-for-mysql-databases-on-amazon-aurora-and-amazon-rds/