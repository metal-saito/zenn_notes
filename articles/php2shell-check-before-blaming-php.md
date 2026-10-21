---
title: "php2shellで「PHP叩き」より先に確認する7つのこと（CVE-2026-109618 / 109619）"
emoji: "🐘"
type: "tech"
topics: ["php", "security", "phpfpm", "apache", "脆弱性"]
published: false
---

:::message
2026年10月22日時点の情報です。緩和策の細部は二次情報で揺れているため、判断の根拠は末尾のGitHub Security Advisory原文にしてください。
:::

## 結論

- 2026年10月21日 23:00（JST）、PHPのCritical脆弱性2件（通称 **php2shell**）の修正版が公開されました。**8.5.12 / 8.4.27 / 8.3.36 / 8.2.35** です。
- どちらも**認証なしRCE**、CVSS v4.0 **9.2**。アプリのコードに欠陥がなくてもPHPエンジン側で刺さります。**mod_phpが最高リスク**です。
- **PHP 8.1以前には公式修正がありません**。ディストリ／ベンダーのバックポートを確認するか、サポート対象へ移行してください。
- 設定だけで完全に回避する方法は**ありません**（公式アドバイザリ明記）。WAFの仮想パッチも「緩和」です。
- 「PHPだから」ではなく「ランタイムを上げない運用だから」が本質です。

## 前提

| 項目 | 内容 |
|---|---|
| 対象SAPI | PHP-FPM / Apache mod_php / php-cgi / FrankenPHP / LiteSpeed など、Webリクエストを処理する全環境 |
| 影響バージョン | 8.2.35 / 8.3.36 / 8.4.27 / 8.5.12 より前。起源はPHP 5.4系までさかのぼる（Wiz Research） |
| CVE-2026-109618 | OOM処理の不完全な初期化を、クリーンアップが有効データとして扱う（GHSA-w8hw-w9fx-j79w） |
| CVE-2026-109619 | リクエストが残した内部状態を、同一ワーカーの次リクエストが初期化前に参照する（GHSA-6wmf-4rv5-p3rp） |
| 発見者 | Wiz Research。8月24日報告、PHP側の修正着手9月21日、10月21日公開 |
| 悪用状況（10/21時点） | IPA「未確認」、Cloudflare・Palo Alto Unit 42「観測した」で見解が割れている |

## 1. Webサーバーが実際に使っているPHPを確認する

`php -v` はCLIのバイナリを見るだけで、Webサーバーが使うPHPと別物のことがよくあります。

```bash
# PHP-FPM
php-fpm -v
systemctl status php-fpm      # or php8.3-fpm など

# Apache mod_php
apachectl -M | grep -i php

# Debian/Ubuntu 系
dpkg -l | grep -E '^ii\s+(php|libapache2-mod-php)'

# RHEL 系
rpm -qa | grep -E '^php'
```

一番確実なのは、Web経由で返ってくる値を見ることです（確認後は必ず削除）。

```php
<?php
// 確認後に削除すること
header('Content-Type: text/plain');
echo PHP_VERSION, "\n", PHP_SAPI, "\n";
echo 'upload_progress=', ini_get('session.upload_progress.enabled'), "\n";
```

`PHP_SAPI` が `apache2handler` ならmod_phpです。更新の優先順位を最上位にしてください。

## 2. 修正版へ更新する（唯一の完全な対策）

```bash
# Debian/Ubuntu
apt update && apt list --upgradable 2>/dev/null | grep php
apt upgrade -y 'php8.*'
systemctl restart php8.3-fpm   # or apache2

# RHEL/AlmaLinux/Rocky
dnf check-update php\*
dnf update -y php\*
systemctl restart php-fpm      # or httpd
```

```diff
- FROM php:8.3-fpm
+ FROM php:8.3.36-fpm
```

更新後は必ずワーカーを再起動し、手順1の確認ページでバージョンが上がったことを見てから削除します。

:::message
公式アナウンスによると、Linuxディストリビューションや商用LTSベンダーには事前にパッチが渡っています。`rpms.remirepo.net`（ボランティア提供）は10/21時点で未更新と案内されていたので、依存している場合はディストリのパッケージかソースビルドへの切り替えを検討してください。
:::

## 3. PHP 8.1以前なら、バックポートの有無を確認する

PHPプロジェクトからの修正は**8.2以降のみ**です。

| 状況 | 取るべき行動 |
|---|---|
| ディストリのLTS（Ubuntu 22.04のphp8.1、RHEL 9のphp8.0など） | ベンダーのセキュリティアドバイザリを確認し、バックポートが出ていれば適用 |
| ソースビルドの7.4 / 8.0 / 8.1 | 公式が「最小限のバックポート」を提供と案内。適用しつつ移行計画を立てる |
| 共有レンタルサーバー | 管理画面でPHPバージョンを切り替え。事業者のメンテ告知に従う |
| いずれも不可 | 更新できるまで停止する選択肢を検討（10/21に国内54組織が実際にそうしています） |

## 4. すぐ上げられない場合の緩和策

公式アドバイザリは "There is no complete generic configuration mitigation" と明記しています。時間を稼ぐための設定です。

**適用場所に注意**：`ini_set()` や `.user.ini` ではリクエスト初期化より後に効くため意味がありません。`php.ini`、FPMプール設定、Apacheの `php_admin_value` で入れて、**全ワーカーを再起動**してください。

```diff
# CVE-2026-109619 向け
- session.upload_progress.enabled = On
+ session.upload_progress.enabled = Off

# PHP 8.5 のみ
- fatal_error_backtraces = On
+ fatal_error_backtraces = Off

# CVE-2026-109618 向け：巨大リクエストボディを拒否
- post_max_size = 128M
+ post_max_size = 8M          ; アップロード不要なら
+ upload_max_filesize = 4M    ; post_max_size より小さく
```

```diff
# FPM プール設定（性能低下を許容できる場合）
- pm.max_requests = 500
+ pm.max_requests = 1
```

:::message alert
`memory_limit = -1` を緩和策として挙げる資料と、疑問を呈する資料の両方があります。採用するなら必ずGHSA原文を読んでからにしてください。本記事では推奨も非推奨もしません。
:::

:::details 公式が挙げている「リスクを下げる」追加項目
- 攻撃者が到達できる経路で致命的エラー（未捕捉例外、`require` 失敗、`Maximum execution time exceeded`）を出さない。アップロード、インポート、エラーページ、デストラクタ、出力ハンドラ、プラグインも対象。
- 初期化前に適用されるINI設定（`.htaccess` の `php_value` 等）で警告・非推奨通知が出ていれば解消する。`error_reporting` や `@` で隠しても効果はない。
- Webサーバー／プロキシで、デコード後のリクエストサイズ、フィールド数、multipartパート数、ヘッダー数、ネスト深度を制限する。`Content-Length` なしのリクエストにも適用する。
- `RLIMIT_AS` / `RLIMIT_DATA` は脆弱な割り当て失敗を誘発しうるので使わない。
:::

## 5. ログで兆候を見る

Wiz Researchが挙げている兆候をgrepにしました。

```bash
# メモリ上限エラー
grep -E 'Allowed memory size of [0-9]+ bytes exhausted' /var/log/php*/error.log /var/log/php-fpm/*.log 2>/dev/null | tail -50

# 実行時間超過
grep -E 'Maximum execution time of [0-9]+ seconds exceeded' /var/log/php*/*.log 2>/dev/null | tail -50

# HTTP 500 の連続（Nginx）
awk '$9 == 500 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# 異常に長い PHPSESSID
grep -oE 'PHPSESSID=[^; ]{64,}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# FPM ワーカーの異常な入れ替わり
grep -E 'child [0-9]+ exited (on signal|with code)' /var/log/php-fpm/*.log 2>/dev/null | tail -50
```

ログに何もなくても安全の証明にはなりません（公式も明記）。

## 6. サーバーが持つ権限を棚卸しする

侵害されていた場合、横展開の範囲はサーバーの権限で決まります。

```bash
aws sts get-caller-identity
aws iam list-attached-role-policies --role-name <role>
mysql -e "SHOW GRANTS FOR CURRENT_USER();"
```

`ALL PRIVILEGES` や `*:*` が付いていれば、今回を機に最小化します。

## 7. WAFの仮想パッチを「適用済み」と混同しない

Cloudflare（10/21 03:02 UTCから）、Akamai（ルール3000991）、Palo Alto（シグネチャ543004）が防御ルールを配信しています。有効ですが、**PHP本体の修正ではありません**。Cloudflare自身が「攻撃者はトラフィックを変えて検知を回避しようとしている」と報告しています。

## 「PHPだから」ではない

今回が特別なのは、**コアランタイムに認証なしRCEがあり、mod_phpではアプリコードが動く前に攻撃が成立しうる**ことです。ただし他の言語も例外ではありません。

| 言語 | コアランタイム | 典型的な痛点 |
|---|---|---|
| Java | 比較的堅牢 | Log4Shell（CVE-2021-44228）のようにライブラリ1本で世界が止まる |
| JavaScript / Node.js | 重大問題は稀 | npm依存関係で問題が起きやすい |
| Ruby | インタプリタ・gemで過去に重大問題 | 依存管理 |
| C / C++ | メモリ安全性なし | php2shellもPHPエンジン（C実装）のメモリ管理に起因 |
| C# / .NET | マネージドでコアRCEは稀 | デシリアライズ周り |
| Go | コアRCEは非常に稀 | それでも標準ライブラリの修正は定期的に出る |

言語を変えても「最新を維持する」「依存関係を管理する」「不要な機能を無効化する」をやらなければ、次は別の言語で同じことが起きます。

一方で、PHP固有に痛い点もあります。EOLバージョンが現場に多すぎること、mod_php構成が残っていること、後方互換が良いせいで「上げなくても動いてしまう」こと。痛いのは言語ではなく、言語の寛容さに甘えてきた運用です。

## 限界

- 悪用の実態は機関によって見解が割れています。どちらが正しいかは本記事では判断できません。
- 「国内54組織が停止」はpiyologさんの集計で公式統計ではありません。
- 緩和策の細部はGHSA原文を正としてください。
- コマンドは一般的なディストリ向けの例です。環境によりパス・サービス名が異なります。
- 筆者はセキュリティ専門機関ではなく、一人の現場エンジニアです。

## まとめ

深刻なのは「PHPの設計」ではなく「コアランタイムの未認証RCE」で、これはどの言語でも起こりえます。言語を叩いている時間があるなら、`php -v` ではなくWebサーバーが使っているPHPのバージョンを確認する。そのほうがずっと建設的です。

## 参考

- PHP公式アナウンス（2026-10-21）: https://www.php.net/archive/2026.php
- GHSA-6wmf-4rv5-p3rp（CVE-2026-109619）: https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp
- GHSA-w8hw-w9fx-j79w（CVE-2026-109618）: https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w
- Wiz Research「PHP2Shell」: https://www.wiz.io/blog/php2shell-php-rce
- piyolog: https://piyolog.hatenadiary.jp/entry/2026/10/21/174516
- GMO Flatt Security Blog: https://blog.flatt.tech/entry/php2shell
- gihyo.jp: https://gihyo.jp/article/2026/10/php-critical-security-release
- セキュリティ対策Lab: https://rocket-boys.co.jp/security-measures-lab/php2shell-cve-2026-109618-19/
