---
layout: post
title: "private-isuでISUCONの演習"
date: 2024-03-10
categories: プログラミング
---

[藤原俊一郎他『達人が教えるWebパフォーマンスチューニング〜ISUCONから学ぶ高速化の実践』](https://gihyo.jp/book/2022/978-4-297-12846-3)を買ったので、[private-isu](https://github.com/catatsuy/private-isu)でISUCONの演習をする。

## 環境立ち上げ

本の付録Aに実践例があり、そこではAWSで起動されているので、得点比較などのために同じくAWSで演習する。

AWS マネジメントコンソールからぽちぽち操作してEC2インスタンスを立ち上げると、気付かないうちにsubnetなどがどんどん作られて収拾がつかなくなるので、AWS CloudFormationで作りたい。

有志の方がCloudFormation templateを[公開されている](https://gist.github.com/tohutohu/024551682a9004da286b0abd6366fa55)ので、これを使う。念のため変なリソースが指定されていないか、特にAMI image IDがprivate-isu公式のものと一致しているかを確認しておく。

AWS CLIから、AWS CloudFormationのstackをdeployする。EC2 key pairとGitHubのアカウント（SSH公開鍵を設定しているもの）は事前に用意しておく。

```bash
wget https://gist.githubusercontent.com/tohutohu/024551682a9004da286b0abd6366fa55/raw/e8a86dd7195c18efc9322f32639fde8021062dd8/private-isu.yaml
aws cloudformation deploy --stack-name private-isu --region ap-northeast-1 --template-file private-isu.yaml --parameter-overrides KeyPairName=your_key_name GitHubUsername=your_github_name
```

サーバーインスタンスとベンチマーカーインスタンスのIPアドレスを探し、それぞれSSHでログインする。

```bash
aws ec2 describe-instances
ssh isucon@your_benrhmark_instance_public_ip
ssh isucon@your_server_instance_public_ip
```

サーバーインスタンスで、ベンチマーカーを実行しやすいように `cd` しておく。

```bash
cd private_isu.git/benchmarker/
```

## 最初のベンチマーク

ベンチマークインスタンスからベンチマークを実行する。サーバーインスタンスに1分間負荷がかけられる。

```bash
./bin/benchmarker -u userdata -t http://192.168.1.10
```

サーバーインスタンス側では `top` コマンドで負荷を観察する。topコマンド実行中に「1」を押すとCPUの各コアの使用率が表示される。

```bash
top
```

ベンチマークが完了すると、約500点という結果が出る。サーバーインスタンス側では、CPU使用率（約100%/200%）、メモリ使用率（700MB/4GB）ともに余裕がある。

> {"pass":true,"score":592,"success":559,"fail":3,"messages":["リクエストがタイムアウトしました (POST /login)","リクエストがタイムアウトしました (POST /register)"]}

## worker processの引き上げ

リソースが全て活用できていない。リソースを活用するため、Unicornのworker processを増加させる。

Unicornの設定ファイルを開き、worker_processesを1から10に変更する。その後、サーバーを再起動する。

```bash
nano private_isu/webapp/ruby/unicorn_config.rb
sudo systemctl restart isu-ruby
```

10に設定したのは、CPUのコア数が2つ（ref. `less /proc/cpuinfo` ）であり、書籍に

> 筆者の経験では、プロセス外部のミドルウェアとの通信が多い典型的なWebアプリケーションの場合、CPUコア数の5倍程度を設定するのが適切な場合が多くありました。

と書いてあったため。

再度ベンチマーカーを実行すると、CPU使用率がmysqldが200%、メモリ使用量が1GB程度になる。また、リクエスト失敗が増えて得点が0に下がってしまう。（ところでこのような失敗は書籍には載っていない。自分で手を動かすのは重要だと改めて思う。）

> {"pass":true,"score":0,"success":174,"fail":57,"messages":["リクエストがタイムアウトしました (GET /)","リクエストがタイムアウトしました (GET /@bessie)","リクエストがタイムアウトしました (GET /@irma)","リクエストがタイムアウトしました (GET /@janet)","リクエストがタイムアウトしました (GET /@marquita)","リクエストがタイムアウトしました (GET /@shelly)","リクエストがタイムアウトしました (GET /image/6910.jpg)","リクエストがタイムアウトしました (GET /posts/9399)","リクエストがタイムアウトしました (POST /login)","リクエストがタイムアウトしました (POST /register)"]}

得点が0なのは困るので、worker_processesを2に変更する。依然としてmysqldがCPUを200%使い切っているが、正の得点が得られる。

> {"pass":true,"score":584,"success":583,"fail":4,"messages":["リクエストがタイムアウトしました (POST /login)","リクエストがタイムアウトしました (POST /register)"]}

## MySQLへのindexの追加

ボトルネックがMySQLのCPU使用量に移動したため、MySQLのスロークエリログを取得する。


`/etc/mysql/mysql.conf.d/mysqld.cnf` の `[mysqld]` 以下を追記する。

```
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 0
```

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

その後、MySQLを再起動する。

```bash
sudo systemctl restart mysql
```

スロークエリログを生成するため、ベンチマークを実行する。スロークエリログを有効化したためか、ベンチマークの得点が落ちる。

> {"pass":true,"score":586,"success":587,"fail":5,"messages":["リクエストがタイムアウトしました (GET /favicon.ico)","リクエストがタイムアウトしました (GET /logout)","リクエストがタイムアウトしました (POST /login)","リクエストがタイムアウトしました (POST /register)"]}

`pt-query-digest` を使い、スロークエリログを集計する。

```bash
sudo apt update && sudo apt install percona-toolkit
sudo pt-query-digest /var/log/mysql/mysql-slow.log | tee digest_$(date +%Y%m%d%H%M).txt
```

70%の時間を、次のクエリが消費していることがわかる。

```sql
SELECT * FROM `comments` WHERE `post_id` = 9984 ORDER BY `created_at` DESC LIMIT 3
```

「Rows examine」の項目を見ると、毎回10万行がチェックされているため、インデックスによる高速化が期待できる。MySQLのパスワードは `isuconp` で設定されている。

```
mysql -u isuconp -p isuconp
mysql> SHOW CREATE TABLE comments;
mysql> ALTER TABLE `comments` ADD INDEX `post_id_idx` (`post_id`);
mysql> exit
```

再度スロークエリログを取得しなおしたいが、前回のベンチマーク時のログと分離するため、ログを削除してMySQLを再起動する（もっといいやり方がある気はする）。

```bash
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
```

ベンチマークを再度実行する。CPU使用率がmysqld 60%、ruby 40% + 40%くらいに変化する。得点も大幅に上昇する。

> {"pass":true,"score":10208,"success":8851,"fail":0,"messages":[]}

## worker processの引き上げとボトルネック探し

再びCPUの使用率に余裕が出てきているため、Unicornのworker processを4に上げる。

```bash
nano private_isu/webapp/ruby/unicorn_config.rb
sudo systemctl restart isu-ruby
```

スロークエリログを削除した上で、再度ベンチマークを実行する。

```bash
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
```

> {"pass":true,"score":13300,"success":12064,"fail":0,"messages":[]}

`top` で確認したCPU使用率は、mysqldが70%、rubyが4×30%程度で、CPUを200%使い切っている。メモリはまだ1GB程度しか使っておらず、余裕がある。

スロークエリログを見ても、インデックスを追加するだけで直ちに改善できそうなクエリは無い。一番時間を使っているクエリが

```sql
SELECT `id`, `user_id`, `body`, `created_at`, `mime` FROM `posts` ORDER BY `created_at` DESC
```

というクエリだが、 `posts` テーブルの行を全て取得しており、必要な処理なのか疑問が残る。使われているのは `app.rb` の227行目で、エンドポイントは `GET /` であり、周辺に `make_posts` というN+1問題を抱えていそうなあやしい関数が見つかる。この辺りを直したい気持ちになるが、その前にまずはここの影響の大きさを計測する。

## `alp` によるアクセスログの集計

まずはnginxがJSON形式でアクセスログを記録するように設定を変更する。 `/etc/nginx/nginx.conf` を編集する。

TODO: 説明丁寧に

```bash
sudo nano /etc/nginx/nginx.conf
```

```
        access_log /var/log/nginx/access.log;
```

```
  log_format json escape=json '{"time": "$time_iso8601",'
    '"host": "$remote_addr",'
    '"status": "$status",'
    '"method": "$request_method",'
    '"uri": "$request_uri",'
    '"body_bytes": "$body_bytes_sent",'
    '"request_time": "$request_time",'
    '"response_time": "$upstream_response_time",'
    '"ua": "$http_user_agent",'
    '"referrer": "$http_referer"}';

        access_log /var/log/nginx/access.log json;
```


ログファイルを削除し、新しいログ形式を適用するためにnginxを再起動する。

```bash
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
```

alpをダウンロードし、展開する。

```bash
wget https://github.com/tkuchiki/alp/releases/download/v1.0.21/alp_linux_amd64.tar.gz
tar xzvf alp_linux_amd64.tar.gz
```

ベンチマーカーを実行し、alpでアクセスログを集計する。

```bash
sudo ./alp json --file /var/log/nginx/access.log --sort sum -r -m "^/image/\d+\.(jpg|png|gif)$,^/posts/\d+$,^/@\w+$"
```

エンドポイント `^/image/\d+\.(jpg|png|gif)$` が一番時間がかかっていることがわかる。

## 画像をnginxから配信

ソースコードを読むと、画像ファイルがRDBに格納され、Rubyを経由して返されていることがわかる。実際、スロークエリログで2番目に時間がかかっているクエリ 

```sql
SELECT * FROM `posts` WHERE `id` = 3906`
```

もこの操作で使われている。

静的ファイルはなるべくアプリケーション部分を介さずにnginxから直接返すようにする。nginxの設定を変更し、 `/image/` 以下のリクエストは `/home/isucon/private_isu/webapp/public/image/` 以下からまずはファイルを探し、もしファイルが見つからなかったらアプリケーションサーバにリバースプロキシするように設定を変更する。

```bash
sudo nano /etc/nginx/sites-enabled/isucon.conf
```

```
server {
  listen 80;

  client_max_body_size 10m;
  root /home/isucon/private_isu/webapp/public/;

  location /image/ {
    root /home/isucon/private_isu/webapp/public/;
    expires 1d;
    try_files $uri @app;
  }

  location @app {
    internal;
    proxy_pass http://localhost:8080;
  }

  location / {
    proxy_set_header Host $host;
    proxy_pass http://localhost:8080;
  }
}
```

画像に関してアプリとRDBを全く使わないようにすることも可能ではあるが、実装が大変なので、ここではリクエストの度にRDBから `/home/isucon/private_isu/webapp/public/image/` 以下に画像ファイルをコピーすることで、ディスクをキャッシュのように使うことにする。 `app.rb` を以下のように変更する。

```diff
@@ -3,6 +3,7 @@ require 'mysql2'
 require 'rack-flash'
 require 'shellwords'
 require 'rack/session/dalli'
+require 'fileutils'

 module Isuconp
   class App < Sinatra::Base
@@ -14,6 +15,8 @@ module Isuconp

     POSTS_PER_PAGE = 20

+    IMAGE_DIR = File.expand_path('../../public/image', __FILE__)
+
     helpers do
       def config
         @config ||= {
@@ -349,6 +352,11 @@ module Isuconp
           (params[:ext] == "png" && post[:mime] == "image/png") ||
           (params[:ext] == "gif" && post[:mime] == "image/gif")
         headers['Content-Type'] = post[:mime]
+
+        imgfile = IMAGE_DIR + "/#{post[:id]}.#{params[:ext]}"
+        f = File.open(imgfile, "w")
+        f.write(post[:imgdata])
+        f.close()
         return post[:imgdata]
       end
```

画像を格納するディレクトリを作っておく。

```bash
mkdir private_isu/webapp/public/image
```

設定が済んだら、再起動する。

```bash
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

再度ベンチマークを実行すると、得点が伸びていることがわかる。

> {"pass":true,"score":24378,"success":23137,"fail":0,"messages":[]}

alpでアクセスログを集計すると、 `image/` にかかる時間の割合が減り、 `GET /` が一番時間がかかるようになっている。

```bash
sudo ./alp json --file /var/log/nginx/access.log --sort sum -r -m "^/image/\d+\.(jpg|png|gif)$,^/posts/\d+$,^/@\w+$"
```

スロークエリログも集計しておこう。

```bash
sudo pt-query-digest /var/log/mysql/mysql-slow.log | tee digest_$(date +%Y%m%d%H%M).txt
```

画像取得のためにも使われていた、以前2番目に時間がかかっていたクエリ

```sql
SELECT * FROM `posts` WHERE `id` = 3906;
```

も、はるか下方10番目まで下がっている。

## `GET /` の改善

`GET /` については、 `app.rb` を見るとわかるように、 `make_posts` 関数がN+1問題を抱えている。スロークエリログを見ても、1番目と3番目のクエリ

```sql
SELECT `id`, `user_id`, `body`, `created_at`, `mime` FROM `posts` ORDER BY `created_at` DESC;
SELECT * FROM `users` WHERE `id` = 876;
```

がどちらもこのエンドポイントで使われているクエリである。

N+1問題を解消したいところではあるが、その前にまずは、 `SELECT` で`posts` を全件取得してからあとで20件に絞り込んでいるところを改善し、MySQLから20件のpostのみを返すように変更しよう。

以下のような変更を加えて、MySQLから20件以下のみを返すようにする。（繰り返しが多くコードが汚くなってしまっている。viewか何かを使ってきれいにしたいが、今はパフォーマンスを上げることが目的なので妥協する。）

```diff
128,130c128
<
<           posts.push(post) if post[:user][:del_flg] == 0
<           break if posts.length >= POSTS_PER_PAGE
---
>           posts.push(post)
230c228
<       results = db.query('SELECT `id`, `user_id`, `body`, `created_at`, `mime` FROM `posts` ORDER BY `created_at` DESC')
---
>       results = db.query("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}")
245c243
<       results = db.prepare('SELECT `id`, `user_id`, `body`, `mime`, `created_at` FROM `posts` WHERE `user_id` = ? ORDER BY `created_at` DESC').execute(
---
>       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}").execute(
274c272
<       results = db.prepare('SELECT `id`, `user_id`, `body`, `mime`, `created_at` FROM `posts` WHERE `created_at` <= ? ORDER BY `created_at` DESC').execute(
---
>       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.created_at <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}").execute(
283c281
<       results = db.prepare('SELECT * FROM `posts` WHERE `id` = ?').execute(
---
>       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, imgdata, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.id = ? AND users.del_flg = 0").execute(
```

各種サービスを再起動し、ベンチマークを再実行する。

```bash
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

```bash
./bin/benchmarker -u userdata -t http://192.168.1.10
```

得点が伸びている。

> {"pass":true,"score":43605,"success":41532,"fail":0,"messages":[]}

## `posts.created_at` への降順インデックスの追加


`alp` を実行すると、依然として GET / が一番時間がかかっている。MySQLのスロークエリログを見ると、

```sql
SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT 20
```

が全体の40%の時間を占めており、rowも多く読み込まれているようである。CPU使用率もmysqldの割合が大きくなっている。

`posts` テーブルの `created_at` 列に降順インデックスを追加する。すると、 `EXPLAIN` の `rows` も `8769` から `199` に削減される。

```bash
mysql -u isuconp -p isuconp
mysql > EXPLAIN SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT 20;
mysql > ALTER TABLE posts ADD INDEX created_at_idx(created_at DESC);
mysql > EXPLAIN SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT 20;
```

各種サービスを再起動し、ベンチマークを再実行する。ところでいままで画像ファイルのキャッシュを消していなかったので、ベンチマーク前に削除するようにしたほうがいいのではないだろうか。といいつつ、消しても消さなくてもベンチマーク結果がほぼ同じだったのでそのままにする。

TODO: 消さないとディスクいっぱいになりやすいかも

```bash
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

```bash
./bin/benchmarker -u userdata -t http://192.168.1.10
```

スコアが伸びている。

> {"pass":true,"score":59995,"success":57191,"fail":0,"messages":[]}

`./alp` を再実行すると、依然として `GET /` が1位であるものの、2位以下との差が縮まっており、 `GET / ` の性能が改善されたことが窺える。

MySQLのスロークエリログでは、以下の2つのクエリがほぼ同率1位になっている。

```sql
administrator command: Prepare;
```

```sql
SELECT * FROM `users` WHERE `id` = 218;
```

`top` では、メモリ使用量には依然として余裕があり、CPU使用率はmysqldが100%、rubyが4 * 17%、nginxが7%程度が使われており、MySQLのスロークエリを改善するのがよさそうである。


## prepared statementを削減
「`administrator command: Prepare`」のほうは、[mysql2](https://github.com/brianmario/mysql2)でprepared statementを使っている部分を[mysql2-cs-bind](https://github.com/tagomoris/mysql2-cs-bind)を使ってwebapp側で処理することで、prepared statementを使うことを回避できるようである。（ただし、この変更がセキュアかどうかは不明）

まずは `Gemfile` に以下の変更を加え、

```diff
9c9
< gem "mysql2"
---
> gem "mysql2-cs-bind"
```

`bundle` で `mysql2-cs-bind` をインストールする。

```bash
bundle
```

その後、 `app.rb` の `db.prepare(q).execute(p)` を `db.xquery(q, p)` に書き換える。ただし、画像を保存しているクエリのみ、 mysql2-cs-bind からの

> Encoding::CompatibilityError - incompatible character encodings: UTF-8 and ASCII-8BIT

というエラーを回避するため、そのままにしている。

```diff
2c2
< require 'mysql2'
---
> require 'mysql2-cs-bind'
57c57
<           db.prepare(s).execute
---
>           db.query(s)
62c62
<         user = db.prepare('SELECT * FROM users WHERE account_name = ? AND del_flg = 0').execute(account_name).first
---
>         user = db.xquery('SELECT * FROM users WHERE account_name = ? AND del_flg = 0', account_name).first
96c96
<           db.prepare('SELECT * FROM `users` WHERE `id` = ?').execute(
---
>           db.xquery('SELECT * FROM `users` WHERE `id` = ?',
107c107
<           post[:comment_count] = db.prepare('SELECT COUNT(*) AS `count` FROM `comments` WHERE `post_id` = ?').execute(
---
>           post[:comment_count] = db.xquery('SELECT COUNT(*) AS `count` FROM `comments` WHERE `post_id` = ?',
115c115
<           comments = db.prepare(query).execute(
---
>           comments = db.xquery(query,
119c119
<             comment[:user] = db.prepare('SELECT * FROM `users` WHERE `id` = ?').execute(
---
>             comment[:user] = db.xquery('SELECT * FROM `users` WHERE `id` = ?',
125c125
<           post[:user] = db.prepare('SELECT * FROM `users` WHERE `id` = ?').execute(
---
>           post[:user] = db.xquery('SELECT * FROM `users` WHERE `id` = ?',
200c200
<       user = db.prepare('SELECT 1 FROM users WHERE `account_name` = ?').execute(account_name).first
---
>       user = db.xquery('SELECT 1 FROM users WHERE `account_name` = ?', account_name).first
208c208
<       db.prepare(query).execute(
---
>       db.xquery(query,
235c235
<       user = db.prepare('SELECT * FROM `users` WHERE `account_name` = ? AND `del_flg` = 0').execute(
---
>       user = db.xquery('SELECT * FROM `users` WHERE `account_name` = ? AND `del_flg` = 0',
243c243
<       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}").execute(
---
>       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
248c248
<       comment_count = db.prepare('SELECT COUNT(*) AS count FROM `comments` WHERE `user_id` = ?').execute(
---
>       comment_count = db.xquery('SELECT COUNT(*) AS count FROM `comments` WHERE `user_id` = ?',
252c252
<       post_ids = db.prepare('SELECT `id` FROM `posts` WHERE `user_id` = ?').execute(
---
>       post_ids = db.xquery('SELECT `id` FROM `posts` WHERE `user_id` = ?',
260c260
<         commented_count = db.prepare("SELECT COUNT(*) AS count FROM `comments` WHERE `post_id` IN (#{placeholder})").execute(
---
>         commented_count = db.xquery("SELECT COUNT(*) AS count FROM `comments` WHERE `post_id` IN (#{placeholder})",
272c272
<       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.created_at <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}").execute(
---
>       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.created_at <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
281c281
<       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, imgdata, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.id = ? AND users.del_flg = 0").execute(
---
>       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, imgdata, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.id = ? AND users.del_flg = 0",
347c347
<       post = db.prepare('SELECT * FROM `posts` WHERE `id` = ?').execute(params[:id].to_i).first
---
>       post = db.xquery('SELECT * FROM `posts` WHERE `id` = ?', params[:id].to_i).first
381c381
<       db.prepare(query).execute(
---
>       db.xquery(query,
424c424
<         db.prepare(query).execute(1, id.to_i)
---
>         db.xquery(query, 1, id.to_i)
```

各種サービスを再起動し、ベンチマークを実行しよう。

```bash
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

```bash
./bin/benchmarker -u userdata -t http://192.168.1.10
```

スコアが伸びている。

> {"pass":true,"score":71506,"success":68490,"fail":0,"messages":[]}

`top` コマンドの傾向には変化無し。 `alp` による傾向も変化無し。MySQLのスロークエリログの集計からは、「administrator command: Prepare」が消え、

```sql
SELECT * FROM `users` WHERE `id` = '907';
```

が1位のクエリになった。

## `make_posts` にある `posts` と `users` のN+1問題の解消

前述のクエリ自体は、すでにインデックスも使用されており、高速化の余地は小さい。ただし、 `make_posts` 関数がN+1問題を抱えており、前述のクエリを繰り返し呼んでいるので、クエリの呼び出し回数を減らせることが期待できる。

まずは `make_posts` 以下での `posts` と `users` の間にあるN+1問題を解消しよう。

`views/` ディレクトリ以下のソースコードを確認すると、 `post[:user]` の属性は `post[:user][:account_name]` のみが使われていることがわかるため、 `users` の全ての列を追加する必要はない。

```diff
125,127c125
<           post[:user] = db.xquery('SELECT * FROM `users` WHERE `id` = ?',
<             post[:user_id]
<           ).first
---
>           post[:user] = { account_name: post[:account_name], }
228c226
<       results = db.query("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}")
---
>       results = db.query("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}")
243c241
<       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
---
>       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
272c270
<       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.created_at <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
---
>       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.created_at <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
281c279
<       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, imgdata, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.id = ? AND users.del_flg = 0",
---
>       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, imgdata, posts.`created_at`, `mime`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.id = ? AND users.del_flg = 0",
```

各種サービスを再起動してベンチマークを実行すると、スコアが伸びている。

```bash
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

```bash
./bin/benchmarker -u userdata -t http://192.168.1.10
```

> {"pass":true,"score":80486,"success":77166,"fail":0,"messages":[]}

## `make_posts` にある `comments` と `users` のN+1問題の解消

次に、 `make_posts` 以下で各 `comment` に対して `users` に複数回クエリを投げている問題を解消しよう。再び `views/` ディレクトリ以下のソースコードを確認すると、 `comment` に関しては `comment[:comment]` と `comment[:user][:account_name]` のみが使われていることがわかる。そこで、 `app.rb` を以下のように書き換える。

```diff
111c111
<           query = 'SELECT * FROM `comments` WHERE `post_id` = ? ORDER BY `created_at` DESC'
---
>           query = 'SELECT comments.comment, users.account_name FROM `comments` JOIN users ON comments.user_id = users.id WHERE `post_id` = ? ORDER BY comments.`created_at` DESC'
119,121c119
<             comment[:user] = db.xquery('SELECT * FROM `users` WHERE `id` = ?',
<               comment[:user_id]
<             ).first
---
>             comment[:user] = { account_name: comment[:account_name], }
```

ベンチマークを実行し、10万点を得る。

```bash
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

```bash
./bin/benchmarker -u userdata -t http://192.168.1.10
```

> {"pass":true,"score":100571,"success":96944,"fail":0,"messages":[]}

依然としてMySQLの使用率が高く、スロークエリログを見ると、以下のクエリが30%の時間を使っている。このクエリは `make_posts` にある `posts` と `comments` の間のN+1問題を抱えているクエリである。

```sql
SELECT comments.comment, users.account_name FROM `comments` JOIN users ON comments.user_id = users.id WHERE `post_id` = '7382' ORDER BY comments.`created_at` DESC LIMIT 3;
```

## `make_posts` にある `posts` と `comments` のN+1問題の解消

N+1問題を解消しよう。各postに対するコメント数を求める部分と、各postに対するコメントを取得する部分がある。まずは後者を直そう。 `app.rb` を修正する。計算量O(MN)の部分ができてしまっているが、私がRubyに詳しくなくてプログラムを書くのが難しいのと、当面はボトルネックにならないはずなので今は妥協する。

```diff
104a105,112
>         if all_comments
>           query = "SELECT ranked.post_id, ranked.comment, users.account_name FROM (SELECT *, RANK() OVER (PARTITION BY post_id ORDER BY created_at DESC) AS rank_new FROM comments) ranked JOIN users ON ranked.use\
r_id = users.id AND post_id IN (?) ORDER BY ranked.created_at DESC"
>         else
>           query = "SELECT ranked.post_id, ranked.comment, users.account_name FROM (SELECT *, RANK() OVER (PARTITION BY post_id ORDER BY created_at DESC) AS rank_new FROM comments) ranked JOIN users ON ranked.use\
r_id = users.id WHERE rank_new <= 3 AND post_id IN (?) ORDER BY ranked.created_at DESC"
>         end
>
>         comments = db.xquery(query, results.map { |post| post[:id] })
>
110,121c118,122
<
<           query = 'SELECT * FROM `comments` WHERE `post_id` = ? ORDER BY `created_at` DESC'
<           unless all_comments
<             query += ' LIMIT 3'
<           end
<           comments = db.xquery(query,
<             post[:id]
<           ).to_a
<           comments.each do |comment|
<             comment[:user] = db.xquery('SELECT * FROM `users` WHERE `id` = ?',
<               comment[:user_id]
<             ).first
---
>           post_comments = []
>           comments.to_a.each do |comment|
>             if comment[:post_id] == post[:id]
>               post_comments.push({ comment: comment[:comment], user: { account_name: comment[:account_name] } })
>             end
123c124
<           post[:comments] = comments.reverse
---
>           post[:comments] = post_comments
```

各種サービスを再起動し、ベンチマークを実行すると、スコアが上がっている。

```bash
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

```bash
./bin/benchmarker -u userdata -t http://192.168.1.10
```

> {"pass":true,"score":122521,"success":118421,"fail":0,"messages":[]}

スロークエリログを見ると各postのコメント数を数えるN+1問題が一番時間がかかるようになっているので、そこを直す。こちらもアルゴリズムが計算量O(MN)になってしまっているが、直すのは後にしよう。

```diff
112a113,114
>         comment_counts = db.xquery("SELECT post_id, COUNT(*) AS `count` FROM `comments` WHERE post_id IN (?) GROUP BY post_id", results.map { |post| post[:id] })
>
115,117c117,121
<           post[:comment_count] = db.xquery('SELECT COUNT(*) AS `count` FROM `comments` WHERE `post_id` = ?',
<             post[:id]
<           ).first[:count]
---
>           comment_counts.to_a.each do |comment_count|
>             if comment_count[:post_id] == post[:id]
>               post[:comment_count] = comment_count[:count]
>             end
>           end
```

サービスを再起動して、ベンチマークを取ると、スコアが上がっている。

```bash
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

```bash
./bin/benchmarker -u userdata -t http://192.168.1.10
```

> {"pass":true,"score":153057,"success":148409,"fail":0,"messages":[]}

CPU使用率は、mysqldが70%、rubyが4 * 17%、nginxが14% + 5%くらいで、MySQLからボトルネックが移動しつつある。

`alp` でアクセスログを集計すると、 `GET /` は2位に落ち、 `POST /login` が1位に躍り出ている。

```bash
sudo ./alp json --file /var/log/nginx/access.log --sort sum -r -m "^/image/\d+\.(jpg|png|gif)$,^/posts/\d+$,^/@\w+$"
```

MySQLのスロークエリログを集計すると、1位と2位のクエリがそれぞれ以下になっている。

```sql
SELECT COUNT(*) AS count FROM `comments` WHERE `user_id` = '369';
SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = '63' AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT 20;
```

それぞれ、Rows examinedが100k、10k行になっており、インデックスが効いていないことが観察される。

## stackprofによるprofiling

`POST /login` のボトルネックを探すため、[stackprof](https://github.com/tmm1/stackprof)を用いてprofilingする。

`Gemfile` に以下の行を追加する。

```
gem 'stackprof'
```

`app.rb` に以下の行を追加する。なお、intervalのデフォルト値は1000（μs）になっているが、それだとボトルネックでない部分が検出されてしまうようだ。

TODO: wall -> cpu

```diff
6a7
> require 'stackprof'
9a11,14
>     use StackProf::Middleware, enabled: true,
>                                mode: :wall,
>                                interval: 100,
>                                save_every: 5
```

`bundle` でGemをインストールする。

```bash
bundle
```

stackprofが出力するファイルを削除し、各種サービスを再起動する。

```bash
rm private_isu/webapp/ruby/tmp/stackprof-*.dump
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

ベンチマークを実行する。

```bash
./bin/benchmarker -u userdata -t http://192.168.1.10
```

プロファイラを入れた影響で、スコアは下がる。

> {"pass":true,"score":91964,"success":88927,"fail":0,"messages":[]}

プロファイリングした結果を集計する。

```bash
stackprof private_isu/webapp/ruby/tmp/stackprof-wall-*.dump
```

すると、以下の行が突出して時間がかかっていることがわかる。

```
     42523  (47.2%)       42523  (47.2%)     Kernel#`
```

この行に着目して再びプロファイリング結果を集計すると、 `Isuconp::App#digest` で呼ばれている外部コマンド呼び出しに時間がかかっていることがわかる。

```bash
stackprof private_isu/webapp/ruby/tmp/stackprof-wall-*.dump --method 'Kernel#`'
```

```
Kernel#` (<cfunc>:1)
  samples:  42523 self (47.2%)  /   42523 total (47.2%)
  callers:
    42523  (  100.0%)  Isuconp::App#digest
  code:
        SOURCE UNAVAILABLE
```

## `digest` の高速化

外部コマンド呼び出しを止めて、RubyのOpenSSLライブラリを使うように変更する。

```diff
7a8
> require 'openssl'
87,88c88
<         # opensslのバージョンによっては (stdin)= というのがつくので取る
<         `printf "%s" #{Shellwords.shellescape(src)} | openssl dgst -sha512 | sed 's/^.*= //'`.strip
---
>         return OpenSSL::Digest::SHA512.hexdigest(src)
```

各種サービスを再起動し、ベンチマークを実行すると、スコアが上がる。

```bash
rm private_isu/webapp/ruby/tmp/stackprof-*.dump
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

```bash
./bin/benchmarker -u userdata -t http://192.168.1.10
```

> {"pass":true,"score":125407,"success":120420,"fail":0,"messages":[]}

なお、開催時のISUCONのルールによっては、パスワードをハッシュ化せずに、平文または平文に類似した形で保存することで、さらなる高速化が狙えるようだ。（もちろん実運用されるアプリケーションでやってはいけない）

`alp` で集計すると再び `GET /` が1位になっている。stackprofで集計すると `Mysql2::Client#_query` で27%、 `String#gsub` で10%の時間がかかっている。MySQLのスロークエリを集計すると、遅いクエリは変わっていない。

## No space left on device

作業を進めていると、「No space left on device」と表示されてアプリが正常に動作しなくなることがある。MySQLのバイナリログを消したり、ディスク上に保存した画像ファイルを削除したり、stackprofのファイルを消すことで解消できるはずだ。MySQLのバイナリログを削除するには、

```
mysql -u isuconp -p isuconp
mysql> PURGE BINARY LOGS BEFORE NOW();
```

とすればよい。

TODO: 保存しないようにしないとディスクが……


## `comments` テーブルにインデックスを追加する

`comments` テーブルの `user_id` 列にインデックスを追加することで、 `EXPLAIN` したときの `rows` が10万行程度から100行程度に減少する。

```
mysql -u isuconp -p isuconp
mysql> EXPLAIN SELECT COUNT(*) AS count FROM `comments` WHERE `user_id` = '369';
mysql> SHOW CREATE TABLE comments;
mysql> ALTER TABLE comments ADD INDEX user_id_idx (user_id);
mysql> SHOW CREATE TABLE comments;
mysql> EXPLAIN SELECT COUNT(*) AS count FROM `comments` WHERE `user_id` = '369';
mysql> exit;
```

```bash
rm private_isu/webapp/ruby/tmp/stackprof-*.dump
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

```bash
./bin/benchmarker -u userdata -t http://192.168.1.10
```

> {"pass":true,"score":126353,"success":121038,"fail":0,"messages":[]}

## `posts` テーブル
続いて、もう一つのクエリでもインデックスが効くようにする。まずはRDBMSにインデックスを追加する。

```
mysql -u isoconp -p isuconp
mysql> ALTER TABLE posts ADD INDEX user_created_idx (user_id, created_at DESC);
```

アプリ側では、 `FORCE INDEX` を使って使うべきインデックスを指示するようにする。現時点で `posts` テーブルには2種類のインデックス `created_at_idx(created_at DESC)` と `user_created_idx (user_id, created_at DESC)` があり、クエリにもuserを指定しないものと指定するものがある。このアプリの変更を入れないと、インデックスの追加によって却って得点が下がってしまう。

```diff
236c236
<       results = db.query("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}")
---
>       results = db.query("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` FORCE INDEX (created_at_idx) JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}")
251c251
<       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
---
>       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` FORCE INDEX (user_created_idx) JOIN users ON posts.user_id = users.id WHERE `user_id` = ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
280c280
<       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.created_at <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
---
>       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` FORCE INDEX (created_at_idx) JOIN users ON posts.user_id = users.id WHERE posts.created_at <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
```

ここで注意点として、 `EXPLAIN` を使って実行計画を表示すると、インデックス追加前からスキャンする行数はかなり少なく表示される。一般に、 `EXPLAIN` による予測と、slow query logによる実際の行数は一致しないことがあるらしい。

```sql
EXPLAIN SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = '63' AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT 20;
```

さて、ベンチマークを再実行しよう。

```bash
rm private_isu/webapp/ruby/tmp/stackprof-*.dump
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

```bash
./bin/benchmarker -u userdata -t http://192.168.1.10
```

得点が上がっている。

> {"pass":true,"score":150649,"success":144327,"fail":0,"messages":[]}

mysqldのCPU使用率が落ちて、50%ほどに減っている。rubyが4 * 30%、nginxが15% + 5%程度を占めている。

## M




<!--

```sql
SELECT ranked.post_id, ranked.comment, users.account_name FROM (SELECT *, RANK() OVER (PARTITION BY post_id ORDER BY created_at DESC) AS rank_new FROM comments WHERE post_id IN ('9885','9884','9883','9882','9881','9880','9879','9878','9877','9876','9875','9874','9873','9872','9871','9870','9869','9868','9867','9866') ORDER BY created_at DESC) ranked JOIN users ON ranked.user_id = users.id WHERE rank_new <= 3\G
```

```sql
ALTER TABLE comments ADD INDEX post_created_idx (post_id, created_at DESC);
```
効果なかった

-->


TODO: 続き


<!--

何か設定が間違っているが、 `/var/log/nginx/error.log` を開いても何も書いていないので、 `/etc/nginx/nginx.conf` を以下のように書き換える。

```diff
-        error_log /var/log/nginx/error.log;
+        error_log /var/log/nginx/error.log debug;
```

その後、nginxを再起動し、error.logを見ると、

```bash
sudo systemctl restart nginx
sudo less /var/log/nginx/error.log
```



mkdir webapp/public/image
nginx error_log deubg

-->

<!--

## `make_posts` の O(MN) の解消

`make_posts` 中に前に作ってしまったO(MN)の非効率なアルゴリズムを改善しよう。

```diff
120c120
<         posts = []
---
>         id_to_post = {}
122,133c122,129
<           comment_counts.to_a.each do |comment_count|
<             if comment_count[:post_id] == post[:id]
<               post[:comment_count] = comment_count[:count]
<             end
<           end
<           post_comments = []
<           comments.to_a.each do |comment|
<             if comment[:post_id] == post[:id]
<               post_comments.push({ comment: comment[:comment], user: { account_name: comment[:account_name] } })
<             end
<           end
<           post[:comments] = post_comments
---
>           id_to_post[post[:id]] = post
>           id_to_post[post[:id]][:comments] = []
>           id_to_post[post[:id]][:user] = { account_name: post[:account_name] }
>         end
>
>         comment_counts.to_a.each do |comment_count|
>           id_to_post[comment_count[:post_id]][:comment_count] = comment_count[:count]
>         end
135,136c131,132
<           post[:user] = { account_name: post[:account_name], }
<           posts.push(post)
---
>         comments.to_a.each do |comment|
>           id_to_post[comment[:post_id]][:comments].push({ comment: comment[:comment], user: { account_name: comment[:account_name] } })
139c135
<         posts
---
>         id_to_post.values
```

```bash
rm private_isu/webapp/ruby/tmp/stackprof-*.dump
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

```bash
./bin/benchmarker -u userdata -t http://192.168.1.10
```

> {"pass":true,"score":119208,"success":114476,"fail":0,"messages":[]}
-->



## CloudFormationスタックを削除する

使っていないときにAWS利用料がかからないように、演習を終えたらAWS上のリソースをすべて削除する。今回はCloudFormationでリソースを作成したので、CloudFormationのstackを削除するだけでよい。

```bash
aws cloudformation delete-stack --stack-name private-isu
```
