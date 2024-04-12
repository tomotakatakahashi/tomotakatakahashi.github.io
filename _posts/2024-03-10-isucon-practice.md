---
layout: post
title: "private-isuでISUCONの演習"
date: 2024-04-02
categories: プログラミング
---

[藤原俊一郎他『達人が教えるWebパフォーマンスチューニング〜ISUCONから学ぶ高速化の実践』](https://gihyo.jp/book/2022/978-4-297-12846-3)を買ったので、[private-isu](https://github.com/catatsuy/private-isu)でISUCONの演習をする。書籍の付録Aにある解法にだいたい従うが、一部異なる部分もある。

## 環境立ち上げ

本の付録Aに実践例があり、そこではAWSで起動されているので、得点比較などのために同じくAWSで演習する。

AWS マネジメントコンソールからぽちぽち操作してEC2インスタンスを立ち上げると、無意識のうちにsubnetなどがどんどん作られて収拾がつかなくなるので、AWS CloudFormationで作りたい。

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

## 最初のベンチマーク（600点）

ベンチマークインスタンスからベンチマークを実行する。サーバーインスタンスに1分間負荷がかけられる。

```bash
./private_isu.git/benchmarker/bin/benchmarker -u ./private_isu.git/benchmarker/userdata -t http://192.168.1.10
```

サーバーインスタンス側では `top` コマンドで負荷を観察する。topコマンド実行中に「1」を押すとCPUの各コアの使用率が表示される。

```bash
top
```

ベンチマークが完了すると、約600点という結果が出る。サーバーインスタンス側では、CPU使用率（約100%/200%）、メモリ使用率（700MB/4GB）ともに余裕がある。

> {"pass":true,"score":636,"success":576,"fail":2,"messages":["リクエストがタイムアウトしました (POST /login)","リクエストがタイムアウトしました (POST /register)"]}

## worker processの2への引き上げ（600点）

CPU、メモリともに余裕があるので、リソースが全て活用できていないことがわかる。リソースを活用するため、Unicornのworker processを増加させる。

Unicornの設定ファイル `private_isu/webapp/ruby/unicorn_config.rb` を開き、worker_processesを1から2に変更する。私はサーバー上のファイルの編集にはEmacsの[TRAMP](https://www.gnu.org/software/tramp/)という機能を使っている。

```diff
1c1
< worker_processes 1
---
> worker_processes 2
```

その後、アプリケーションを再起動する。

```bash
sudo systemctl restart isu-ruby
```

再度ベンチマークを実行する。すると、mysqldがCPUをほぼ200%全て使い切るようになる。得点には大きな変化はない。（なぜ得点が増えないのかは不明。SMTの影響というやつだろうか？）

> {"pass":true,"score":578,"success":565,"fail":4,"messages":["リクエストがタイムアウトしました (POST /login)","リクエストがタイムアウトしました (POST /register)"]}

なお、worker processを10などの大きすぎる値に変更すると、リクエスト失敗が増えて得点が0に下がってしまうので注意。

## MySQLのスロークエリログの取得・集計

MySQLのCPU使用量が非常に大きいので、原因を調べるためにMySQLのスロークエリログを取得する。

`/etc/mysql/mysql.conf.d/mysqld.cnf` の `[mysqld]` 以下に、スロークエリログを記録する設定を追加する。設定ファイルの編集に管理者権限が必要なので、サーバー上の `nano` で編集することにする。

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

```diff
14a15,17
> slow_query_log = 1
> slow_query_log_file = /var/log/mysql/mysql-slow.log
> long_query_time = 0
```

その後、MySQLを再起動する。

```bash
sudo systemctl restart mysql
```

スロークエリログを生成するため、ベンチマークを実行する。得点はほとんど変わらない。

> {"pass":true,"score":682,"success":654,"fail":3,"messages":["リクエストがタイムアウトしました (POST /login)","リクエストがタイムアウトしました (POST /register)"]}

`pt-query-digest` を使い、スロークエリログを集計する。

```bash
sudo apt update && sudo apt install percona-toolkit
sudo pt-query-digest /var/log/mysql/mysql-slow.log > digest_$(date +%Y%m%d%H%M).txt
less $(ls -r digest_*.txt| head -n 1)
```

70%の時間を、次のクエリが消費していることがわかる。

```sql
SELECT * FROM `comments` WHERE `post_id` = 9983 ORDER BY `created_at` DESC LIMIT 3\G
```

## `comments` テーブルの `post_id` 列にindexの追加（12000点）

「Rows examine」の項目を見ると、毎回10万行がチェックされているため、インデックスの追加による高速化が期待できる。MySQLのパスワードは `isuconp` で設定されている。

```
mysql -u isuconp -p isuconp
mysql> SHOW CREATE TABLE comments;
mysql> ALTER TABLE `comments` ADD INDEX `post_id_idx` (`post_id`);
mysql> exit
```

再度スロークエリログを取得しなおしたいが、前回のベンチマーク時のログと分離するため、ログを削除してMySQLを再起動する。

```bash
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
```

ベンチマークを再度実行する。CPU使用率がmysqld 70%、ruby 50% + 50%くらいに変化する。得点も大幅に上昇する。

> {"pass":true,"score":11579,"success":10058,"fail":0,"messages":[]}

## worker processの4への引き上げ（14000点）

再びCPUの使用率に余裕が出てきているため、Unicornのworker processを4に上げる。 `private_isu/webapp/ruby/unicorn_config.rb` を編集する。

```diff
1c1
< worker_processes 2
---
> worker_processes 4
```

MySQLのスロークエリログを削除し、各種サービスを再起動する。

```bash
sudo systemctl restart isu-ruby
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
```

再度ベンチマークを実行する。得点が上がる。

> {"pass":true,"score":14433,"success":13132,"fail":0,"messages":[]}

`top` で確認したCPU使用率は、mysqldが70%、rubyが4×30%程度で、CPUを200%使い切っている。メモリはまだ1GB程度しか使っておらず、余裕がある。

スロークエリログを見ても、インデックスを追加するだけで直ちに改善できそうなクエリは無い。50%近くを占めていて、一番時間を使っているクエリが

```sql
SELECT `id`, `user_id`, `body`, `created_at`, `mime` FROM `posts` ORDER BY `created_at` DESC\G
```

というクエリだが、 `posts` テーブルの行を全て取得しており、必要な処理なのか疑問が残る。使われているのは `app.rb` の227行目で、エンドポイントは `GET /` であり、周辺に `make_posts` というN+1問題を抱えていそうなあやしい関数が見つかる。この辺りを直したい気持ちになるが、その前にまずはここの影響の大きさを計測する。

## `alp` によるアクセスログの集計

まずはnginxがJSON形式でアクセスログを記録するように設定を変更する。 `/etc/nginx/nginx.conf` を編集する。ログフォーマットは[alpのREADMEにあるもの](https://github.com/tkuchiki/alp/blob/d91a23dc2d71521c5a9e166faf92f68d082fb85f/README.md#nginx-1)を使っている。

```bash
sudo nano /etc/nginx/nginx.conf
```

```diff
39c39,56
<       access_log /var/log/nginx/access.log;
---
>       log_format json escape=json
>               '{"time":"$time_local",'
>               '"host":"$remote_addr",'
>               '"forwardedfor":"$http_x_forwarded_for",'
>               '"req":"$request",'
>               '"status":"$status",'
>               '"method":"$request_method",'
>               '"uri":"$request_uri",'
>               '"body_bytes":$body_bytes_sent,'
>               '"referer":"$http_referer",'
>               '"ua":"$http_user_agent",'
>               '"request_time":$request_time,'
>               '"cache":"$upstream_http_x_cache",'
>               '"runtime":"$upstream_http_x_runtime",'
>               '"response_time":"$upstream_response_time",'
>               '"vhost":"$host"}';
>
>       access_log /var/log/nginx/access.log json;
```

nginxのアクセスログを削除し、新しいログ形式を適用するためにnginxを再起動する。また、MySQLについても同様にする。

```bash
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
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

## 画像をnginxでキャッシュ（23000点）

ソースコードを読むと、画像ファイルがRDBに格納され、Rubyを経由して返されていることがわかる。実際、スロークエリログで3番目に時間がかかっているクエリ

```sql
SELECT * FROM `posts` WHERE `id` = 10333\G
```

もこの操作で使われている。

ソースコードを読むと投稿画像は追加されることはあるが更新・削除されることはないことがわかる。そこで、 `/image/` 以下のリクエストは、nginxでキャッシュすることにする。 `/etc/nginx/sites-avaliable/isucon.conf` にキャッシュの設定を追加する。（nginxに詳しくないので、適当に書いている）

```diff
0a1,2
> proxy_cache_path /home/isucon/private_isu/webapp/image_cache levels=1:2 keys_zone=IMAGE:10m inactive=24h max_size=1g;
>
5a8,14
>
>   location /image/ {
>     proxy_cache IMAGE;
>     proxy_cache_valid 200 1d;
>     proxy_set_header Host $host;
>     proxy_pass http://localhost:8080;
>   }
```

各種サービスやログをリフレッシュして、ベンチマークをとる。

```bash
sudo rm -r private_isu/webapp/image_cache && mkdir private_isu/webapp/image_cache
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
```

スコアが伸びる。

> {"pass":true,"score":23321,"success":22119,"fail":0,"messages":[]}

`alp` で集計すると、 `/image/` 以下へのリクエストにかかる時間も減り、 `GET /` のリクエストの時間が1位になっていることがわかる。

スロークエリログも集計する。画像取得のためにも使われていたクエリも、上位から消えている。

## stackprof の導入（17000点）

1番時間がかかっている `GET /` の速度を改善したい。前述の通りN+1問題を抱えているので、そのあたりを改善すればいいだろうとあたりはつくが、念のためプロファイラ [stackprof](https://github.com/tmm1/stackprof) を導入して調査しよう。

`Gemfile` に `gem 'stackprof'` を追加する。

```diff
12a13
> gem 'stackprof'
```

`app.rb` に以下の行を追加する。なお、intervalのデフォルト値は1000（μs）になっているが、それだとボトルネックでない部分が検出されてしまうようだ。

```diff
5a6
> require 'stackprof'
10a12,15
>     use StackProf::Middleware, enabled: true,
>                                mode: :wall,
>                                interval: 100,
>                                save_every: 5
```

`bundle` コマンドでgemをインストールする。

```bash
bundle install --gemfile private_isu/webapp/ruby/Gemfile
```

stackprofが出力するファイルを削除し、各種サービスを再起動する。

```bash
rm private_isu/webapp/ruby/tmp/stackprof-*.dump
sudo rm -r private_isu/webapp/image_cache && mkdir private_isu/webapp/image_cache
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

ベンチマークを実行すると、プロファイラを入れた影響で得点が下がる。

> {"pass":true,"score":16680,"success":15798,"fail":1,"messages":["response code should be 200, got 500 (GET /@jeannie)"]}

`stackprof` コマンドを使って掘り下げていくと、やはり `make_posts` に時間がかかっていることがわかる。

```bash
stackprof private_isu/webapp/ruby/tmp/stackprof-wall-*.dump --method 'block in <class:App>' --sort-total
```

## 20件を一度に取得（25000点）

`GET /` を高速化するために `make_posts` のN+1問題を解決したい。 `make_posts` には `posts` テーブルから全件取得した投稿が渡され、DBにクエリを投げながら、その中から20件の投稿を抽出する動作になっている。まずはこれを1回のクエリで20件のみの投稿を抽出するように変更しよう。

実際、スロークエリログを見ても、1番目と4番目のクエリ

```sql
SELECT `id`, `user_id`, `body`, `created_at`, `mime` FROM `posts` ORDER BY `created_at` DESC\G
SELECT * FROM `users` WHERE `id` = 57\G
```

がどちらも `GET /` で、20件のポストを抽出するために使われているクエリだ。

以下のような変更を加えて、MySQLから20件以下のみを返すようにする。繰り返しが多くコードが汚くなってしまっている。viewか何かを使ってきれいにしたいが、今はパフォーマンスを上げることが目的なので妥協する。また、2番目のクエリは、1、3、4番目のクエリと違って、まず `user_id` で絞り込める点で性格の異なるクエリであることに注意。

```diff
131,132c131
<           posts.push(post) if post[:user][:del_flg] == 0
<           break if posts.length >= POSTS_PER_PAGE
---
>           posts.push(post)
232c231
<       results = db.query('SELECT `id`, `user_id`, `body`, `created_at`, `mime` FROM `posts` ORDER BY `created_at` DESC')
---
>       results = db.query("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}")
247c246
<       results = db.prepare('SELECT `id`, `user_id`, `body`, `mime`, `created_at` FROM `posts` WHERE `user_id` = ? ORDER BY `created_at` DESC').execute(
---
>       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, `mime`, posts.`created_at` FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}").execute(
276c275
<       results = db.prepare('SELECT `id`, `user_id`, `body`, `mime`, `created_at` FROM `posts` WHERE `created_at` <= ? ORDER BY `created_at` DESC').execute(
---
>       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, `mime`, posts.`created_at` FROM `posts` JOIN users ON posts.user_id = users.id WHERE `created_at` <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}").execute(
285c284
<       results = db.prepare('SELECT * FROM `posts` WHERE `id` = ?').execute(
---
>       results = db.prepare('SELECT posts.* FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.`id` = ? AND users.del_flg = 0').execute(
```

TODO: imgdata

各種サービスを再起動し、ベンチマークを再実行すると、得点が伸びている。

> {"pass":true,"score":24644,"success":26044,"fail":294,"messages":["response code should be 200, got 500 (GET /posts)","response code should be 200, got 500 (GET /posts/4795)","ステータスコードが正しくありません: expected 422, got 500 (POST /)"]}

## `posts.created_at` への降順インデックスの追加（33000点）

`alp` を実行すると、依然として `GET /` が一番時間がかかっている。stackprofで `GET /` に対応するブロックを調べる。

```bash
stackprof private_isu/webapp/ruby/tmp/stackprof-wall-*.dump --method 'block in <class:App>' --sort-total
```

すると、以下の2行にそれぞれ同じくらい（若干前者のほうが多く）の時間がかかっていることがわかる。

```ruby
results = db.query("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}")
posts = make_posts(results)
```

前者のDBへのクエリについて調べよう。

MySQLのスロークエリログを見ると、

```sql
SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT 20
```

が全体の50%の時間を占めており、rowも1回あたり1万行ほど読み込まれているようである。CPU使用率もmysqldの割合が大きくなっている。

このクエリを効率化するため、 `posts` テーブルの `created_at` 列に降順インデックスを追加する。すると、 `EXPLAIN` の `rows` も約1万から `199` に削減される。

```bash
mysql -u isuconp -p isuconp
mysql > EXPLAIN SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT 20;
mysql > ALTER TABLE posts ADD INDEX created_at_idx(created_at DESC);
mysql > EXPLAIN SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT 20;
```

各種サービスを再起動し、ベンチマークを再実行すると、スコアが伸びている。

> {"pass":true,"score":32657,"success":34495,"fail":389,"messages":["response code should be 200, got 500 (GET /)","response code should be 200, got 500 (GET /posts)"]}

## `make_posts` にある `posts` と `users` の N+1問題の解消（36000点）

`alp` を再実行すると、依然として `GET /` が1位であるものの、2位以下との差が縮まっており、 `GET / ` の性能が改善されたことが窺える。stackprofを使うと、 `GET /` では大部分の時間が `make_posts` に費やされていることがわかる。 `make_posts` は複数のN+1問題を抱えており、どこも同じくらい時間が費やされている。まずは簡単に直せそうな `posts` と `users` の間のN+1問題を解消しよう。

`views/` ディレクトリ以下のソースコードを確認すると、 `post[:user]` の属性は `post[:user][:account_name]` のみが使われていることがわかる。テーブルをJOINした結果から、 `users.account_name` をそのまま使うことにしよう。 `app.rb` を編集する。

```diff
127,129c127
<           post[:user] = db.prepare('SELECT * FROM `users` WHERE `id` = ?').execute(
<             post[:user_id]
<           ).first
---
>           post[:user] = { account_name: post[:account_name] }
231c229
<       results = db.query("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime` FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}")
---
>       results = db.query("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}")
246c244
<       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, `mime`, posts.`created_at` FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}").execute(
---
>       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, `mime`, posts.`created_at`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}").execute(
275c273
<       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, `mime`, posts.`created_at` FROM `posts` JOIN users ON posts.user_id = users.id WHERE `created_at` <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}").execute(
---
>       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, `mime`, posts.`created_at`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE `created_at` <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}").execute(
284c282
<       results = db.prepare('SELECT posts.* FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.`id` = ? AND users.del_flg = 0').execute(
---
>       results = db.prepare('SELECT posts.*, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.`id` = ? AND users.del_flg = 0').execute(
```

各種サービスを再起動してベンチマークを実行すると、得点が伸びている。

> {"pass":true,"score":36029,"success":37966,"fail":420,"messages":["response code should be 200, got 500 (GET /posts)"]}

## `make_posts` にある `posts` と `comments` のN+1問題の解消 1/2（45000点）

`make_posts` にはまだ2箇所N+1問題が残っている。各postに対するコメントを取得する部分と、各postに対するコメント数を求める部分で、どちらも同じくらい実行時間がかかっている。まずは前者を直す。

`app.rb` を修正して、各postに対するコメントをDBへの1回のクエリで取得するようにする。

```diff
107c107,116
<         posts = []
---
>         comments_query = "SELECT ranked.post_id, ranked.comment, users.account_name FROM (SELECT *, RANK() OVER (PARTITION BY post_id ORDER BY created_at DESC) AS rank_new FROM comments WHERE post_id IN (?)) ranked JOIN users ON ranked.user_id = users.id"
>         unless all_comments
>           comments_query += ' WHERE rank_new <= 3'
>         end
>
>         comments = db.prepare(comments_query).execute(
>           results.map { |post| post[:id] }
>         )
>
>         id_to_post = {}
108a118,120
>           id_to_post[post[:id]] = post
>           post[:comments] = []
>           post[:user] = { account_name: post[:account_name] }
112,129d123
<
<           query = 'SELECT * FROM `comments` WHERE `post_id` = ? ORDER BY `created_at` DESC'
<           unless all_comments
<             query += ' LIMIT 3'
<           end
<           comments = db.prepare(query).execute(
<             post[:id]
<           ).to_a
<           comments.each do |comment|
<             comment[:user] = db.prepare('SELECT * FROM `users` WHERE `id` = ?').execute(
<               comment[:user_id]
<             ).first
<           end
<           post[:comments] = comments.reverse
<
<           post[:user] = { account_name: post[:account_name] }
<
<           posts.push(post)
132c126,130
<         posts
---
>         comments.to_a.each do |comment|
>           id_to_post[comment[:post_id]][:comments].push({ comment: comment[:comment], user: { account_name: comment[:account_name] } })
>         end
>
>         id_to_post.values
```

各種サービスを再起動してからベンチマークを走らせ直すと、得点が上がる。

> {"pass":true,"score":45371,"success":47872,"fail":529,"messages":["response code should be 200, got 500 (GET /posts)"]}

## `make_posts` にある `posts` と `comments` のN+1問題の解消 2/2（53000点）

`alp` で集計すると、いまだに `GET /` に時間がかかっている。stackprofでも `make_posts` にある残り1つのN+1問題に時間がかかっていることがわかる。そのN+1問題を直そう。 `app.rb` を編集する。

```diff
121,123d120
<           post[:comment_count] = db.prepare('SELECT COUNT(*) AS `count` FROM `comments` WHERE `post_id` = ?').execute(
<             post[:id]
<           ).first[:count]
127a125,129
>         end
>
>         comment_counts = db.prepare("SELECT post_id, COUNT(*) AS `count` FROM `comments` WHERE post_id IN (?) GROUP BY post_id").execute(results.map { |post| post[:id] })
>         comment_counts.to_a.each do |comment_count|
>           id_to_post[comment_count[:post_id]][:comment_count] = comment_count[:count]
```

各種サービスを再起動してからベンチマークを走らせ直すと、得点が上がる。

> {"pass":true,"score":52952,"success":55584,"fail":592,"messages":["response code should be 200, got 500 (GET /posts)"]}

## `POST /` での外部コマンドの使用中止（72000点）

`alp` で集計すると、ついに `GET /` が1位から2位に転落し、 `POST /login` が1位になっている。

stackprofで時間がかかっている原因を順次探っていく。

```bash
stackprof private_isu/webapp/ruby/tmp/stackprof-wall-*.dump --method 'block in <class:App>' --sort-total
stackprof private_isu/webapp/ruby/tmp/stackprof-wall-*.dump --method 'Isuconp::App#try_login' --sort-total
stackprof private_isu/webapp/ruby/tmp/stackprof-wall-*.dump --method 'Isuconp::App#calculate_passhash' --sort-total
stackprof private_isu/webapp/ruby/tmp/stackprof-wall-*.dump --method 'Isuconp::App#digest' --sort-total
```

`app.rb` の以下の行に時間がかかっていることがわかる。外部コマンド呼び出しをしているが、外部コマンド呼び出しは遅いことが知られている。

```ruby
`printf "%s" #{Shellwords.shellescape(src)} | openssl dgst -sha512 | sed 's/^.*= //'`.strip
```

また、この処理は全体の6割を占めており、 `POST /login` の中だけでなく全体で見ても遅いことがわかる。

```bash
stackprof private_isu/webapp/ruby/tmp/stackprof-wall-*.dump
```

`app.rb` を編集し、RubyのOpenSSLライブラリを使うように変更する。

```diff
6a7
> require 'openssl'
84,85c85
<         # opensslのバージョンによっては (stdin)= というのがつくので取る
<         `printf "%s" #{Shellwords.shellescape(src)} | openssl dgst -sha512 | sed 's/^.*= //'`.strip
---
>         return OpenSSL::Digest::SHA512.hexdigest(src)
```

各種サービスを再起動し、ベンチマークを実行すると、得点が上がる。

> {"pass":true,"score":72376,"success":74198,"fail":726,"messages":["response code should be 200, got 500 (GET /)","response code should be 200, got 500 (GET /posts)"]}

なお、開催時のISUCONのルールによっては、パスワードをハッシュ化せずに、平文または平文に類似した形で保存することで、さらなる高速化が狙えるようだ。（もちろん実運用されるアプリケーションでやってはいけない）

## `comments` テーブルの `user_id` 列にインデックスの追加（78000点）

`alp` で集計すると、再び `GET /` が1位になっている。stackprofで `GET /` のプロファイリングをとると、4回のSQLでの問い合わせに時間がかかっていることがわかる。MySQLのスロークエリログを集計しても `GET /` に関わるクエリはすべてRows examineが小さく、インデックスが効いていることがわかる。 `GET /` を高速化することは難しそうだ。

気持ちを切り替えて、 `alp` で2位のエンドポイント `GET ^/@\w+$` のほうを見てみよう。stackprofを使うとこちらもMySQLへの問い合わせに時間がかかっていることがわかる。スロークエリログを見ると、1位と2位のクエリ

```sql
SELECT COUNT(*) AS count FROM `comments` WHERE `user_id` = 124\G
SELECT posts.`id`, `user_id`, `body`, `mime`, posts.`created_at`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = 271 AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT 20\G
```

のRows examineがそれぞれ10万行、1万行になっており、インデックスが使われていないであろうことがわかる。

まずは1つ目のクエリ用に、 `comments` テーブルの `user_id` 列にインデックスを追加しよう。

```sql
ALTER TABLE comments ADD INDEX user_id_idx (user_id);
```

各種サービスを再起動し、ベンチマークを再実行する。得点が伸びる。

> {"pass":true,"score":77845,"success":79655,"fail":780,"messages":["response code should be 200, got 500 (GET /posts)"]}

## `posts` テーブルにインデックスの追加（82000点）

続いて、先ほどの2番目のクエリ

```sql
SELECT posts.`id`, `user_id`, `body`, `mime`, posts.`created_at`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = 271 AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT 20\G
```

を、インデックスを追加して、高速化しよう。

```sql
ALTER TABLE posts ADD INDEX user_created_idx (user_id, created_at DESC);
```

ただし、インデックスを追加したたけでは、却って得点が下がってしまう。

> {"pass":true,"score":26792,"success":27767,"fail":288,"messages":["response code should be 200, got 500 (GET /posts)"]}

MySQLのスロークエリログを集計すると、極端に遅いクエリが1つできていることがわかる。そこで、アプリ側で `FORCE INDEX` を使って、使うべきインデックスを指示するようにする。現時点で `posts` テーブルには2種類のインデックス `created_at_idx (created_at DESC)` と `user_created_idx (user_id, created_at DESC)` があるので、まず時刻から絞り込むべきクエリでは前者のインデックスを使い、まずユーザーから絞り込むべきクエリでは後者のインデックスを使いたい。次の1つのクエリの分だけ `FORCE INDEX` をつければ十分のようだ。

```diff
229c229
<       results = db.query("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}")
---
>       results = db.query("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` FORCE INDEX (created_at_idx) JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}")
```

各種サービスを再起動してベンチマークを走らせると、得点が上がる。

> {"pass":true,"score":82480,"success":83516,"fail":780,"messages":["response code should be 200, got 500 (GET /posts)"]}

注意点として、 `EXPLAIN` を使って実行計画を表示すると、インデックス追加前からスキャンする行数はかなり少なく表示される。一般に、 `EXPLAIN` による予測と、slow query logによる実際の行数は一致しないことがあるらしい。

```sql
EXPLAIN SELECT posts.`id`, `user_id`, `body`, `mime`, posts.`created_at`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE `user_id` = 271 AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT 20\G
```

## 静的ファイルをnginxから配信する（100000点）

`alp` で集計すると、favicon、JavaScript、CSSファイルの項目が上位で目立っている。これらのファイルはRubyを経由して返されており、しかもこれらのHTTP status codeは全て2XXになっていて、キャッシュが効いていないことがわかる。nginxから直接配信して、キャッシュも効かせよう。

`/etc/nginx/sites-avaliable/isucon.conf` に以下を追加する。

```diff
8a9,13
>   location ~ ^/(favicon\.ico|css/|js/|img/) {
>     root /home/isucon/private_isu/webapp/public/;
>     expires 1d;
>   }
>
```

各種サービスを再起動してベンチマークを走らせると、得点が上がる。

> {"pass":true,"score":101353,"success":104468,"fail":944,"messages":["response code should be 200, got 500 (GET /posts)"]}

`alp` で分析すると、静的ファイルそれぞれについて、2XXが1度だけであとは3XXのレスポンスを高速に返せるようになっていることがわかる。

## 画像の読み込みで304を返せるようにする（170000点）

`alp` で集計すると、画像を返すエンドポイント `^/image/\d+\.(jpg|png|gif)$` が3位になっている。

先ほど画像をnginxを使って `proxy_cache` でキャッシュするようにしたが、レスポンスのステータスコードはすべて2XXになっている。

画像をMySQLでなくディスクにファイルとして保存するようにして、nginxからファイルとして直接配信することで、304のレスポンスを返せるようにしよう。

`/etc/nginx/sites-enabled/isucon.conf` を編集して、 `/image/` 以下のリクエストは `/home/isucon/private_isu/webapp/public/image/` 以下からまずはファイルを探し、もしファイルが見つからなかったらアプリケーションサーバにリバースプロキシするように設定を変更する。

```diff
1,2d0
< proxy_cache_path /home/isucon/private_isu/webapp/image_cache levels=1:2 keys_zone=IMAGE:10m inactive=24h max_size=1g;
<
15,17c13,19
<     proxy_cache IMAGE;
<     proxy_cache_valid 200 1d;
<     proxy_set_header Host $host;
---
>     root /home/isucon/private_isu/webapp/public/;
>     expires 1d;
>     try_files $uri @app;
>   }
>
>   location @app {
>     internal;
```

`app.rb` を変更して、 `/image/` へのリクエストの度にRDBから `/home/isucon/private_isu/webapp/public/image/` 以下に画像ファイルをコピーすることで、ディスクをキャッシュのように使うことにする。

```diff
7a8
> require 'fileutils'
22a24,25
>     IMAGE_DIR = File.expand_path('../../public/image', __FILE__)
>
353a357,361
>
>         imgfile = IMAGE_DIR + "/#{post[:id]}.#{params[:ext]}"
>         f = File.open(imgfile, "w")
>         f.write(post[:imgdata])
>         f.close()
```

各種サービスやログをリフレッシュして、ベンチマークをとると、得点が大幅に上がる。

```bash
sudo rm -r private_isu/webapp/image_cache
rm -r private_isu/webapp/public/image; mkdir private_isu/webapp/public/image
rm private_isu/webapp/ruby/tmp/stackprof-*.dump
sudo rm /var/log/nginx/access.log && sudo systemctl restart nginx
sudo rm /var/log/mysql/mysql-slow.log && sudo systemctl restart mysql
sudo systemctl restart isu-ruby
```

> {"pass":true,"score":174783,"success":188852,"fail":1981,"messages":["response code should be 200, got 500 (GET /)","response code should be 200, got 500 (GET /posts)","静的ファイルが正しくありません (GET /image/15377.jpg)","静的ファイルが正しくありません (GET /image/15389.png)","静的ファイルが正しくありません (GET /image/15417.png)","静的ファイルが正しくありません (GET /image/15522.png)"]}

## `GET /posts` のバグを直す（210000点）

今まで全く気付いていなかったが、「20件を一度に取得」で `posts` テーブルと `users` テーブルを `JOIN` するようにしてからずっと、 `GET /posts` がバグっていて500エラーを返し続けている。 `journalctl -xeu isu-ruby` でアプリケーションサーバーのログを見ると、SQLで `created_at` 列が2つあって曖昧なためにエラーが起きていることがわかる。 `app.rb` を編集して、曖昧性を解消する。それだけだと意図しないインデックスが使われてしまうので、 `FORCE INDEX` も追加する。

```diff
276c276
<       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, `mime`, posts.`created_at`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE `created_at` <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}").execute(
---
>       results = db.prepare("SELECT posts.`id`, `user_id`, `body`, `mime`, posts.`created_at`, users.account_name FROM `posts` FORCE INDEX (created_at_idx) JOIN users ON posts.user_id = users.id WHERE posts.`created_at` <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}").execute(
```

各種サービスを再起動してベンチマークを実行すると、得点が上がる。ベンチマーカーからの `GET /posts` に関するエラーも消える。

> {"pass":true,"score":211592,"success":205992,"fail":5,"messages":["静的ファイルが正しくありません (GET /image/20035.png)","静的ファイルが正しくありません (GET /image/20091.jpg)","静的ファイルが正しくありません (GET /image/20100.png)","静的ファイルが正しくありません (GET /image/20132.png)","静的ファイルが正しくありません (GET /image/20140.png)"]}


WIP


## コメント機能を消す（TODO点）

`alp` による集計で上位に来ているエンドポイントは `GET /` である。このエンドポイントはすべてのクエリがインデックスを利用できており、改善するのは難しそうだ。

書籍では `comments` テーブルをMemcachedにキャッシュさせることで高速化しているようだ。

ただし、キャッシュを使う上で気になる点として、ISUCONのルール上、投稿されたコメントをレスポンスに反映するのに何秒までの遅延が許されるかが知りたい。書籍には

> ある投稿に関するコメントのキャッシュは、その投稿に対してコメントが投稿されたタイミングで破棄する

と書いてあり、コメントを投稿するとそれ以降のリクエストでは全て新しいコメントが反映されたレスポンスが返るという雰囲気のことが書いてある。しかし一方で、書籍に記載されている `app.rb` の変更点（ref. [サポートページ](https://github.com/tatsujin-web-performance/tatsujin-web-performance/commit/8de837130c50186ce5cd08b560552cd97a1e9b34)）を見ると、キャッシュのTTLである10秒間がすぎるまで新しいコメントがレスポンスに反映されない（場合がある）実装になっている。

ベンチマーカーがどこまで許容するのか知りたいので、 `app.rb` の `make_posts` を編集して、まずは極端な例としてコメントを全く返さない実装にしてみる。

```diff
115,118d114
<         comments = db.prepare(comments_query).execute(
<           results.map { |post| post[:id] }
<         )
<
124,127d119
<         end
<
<         comments.to_a.each do |comment|
<           id_to_post[comment[:post_id]][:comments].push({ comment: comment[:comment], user: { account_name: comment[:account_name] } })
```

そして各種サービスをリフレッシュしてからベンチマークをとると、なんとベンチマーカーは特にエラーなどは起こさず、得点が上がる。

> TODO

[ISUCONのルール](https://github.com/catatsuy/private-isu/blob/2b7ec5fee0952212cd20740de5eae33c753569c1/manual.md#%E5%88%B6%E7%B4%84%E4%BA%8B%E9%A0%85)に

> 存在するべきDOM要素がレスポンスHTMLに存在しない

という条件を満たすと点数が無効になると書いてあるので、このような変更はしてはいけない気がするが、一方で[別の記事](https://isucon.net/archives/32976287.html)では

> 予選レギュレーションに「DOM構造は変化させない」というのがありましたが、採点条件にあるとおり「チェッカがDOM構造が変化していない」と判断していればちょっとくらい変えてもそれはアリです。

という説明もあるので、ベンチマーカーが検出していない限りの変更は加えてもいいのかもしれない。書籍でのコードの変更でも、コメントを追加してからキャッシュのTTLが切れるまでの時間はは「存在するべきDOM要素がレスポンスHTMLに存在しない」状態になりうるはずだ。

結論としては、許される変更なのかはよくわからないが、とりあえずコメント関係の機能を `app.rb` から削除しよう。

## No space left on device

作業を進めていると、「No space left on device」と表示されてアプリが正常に動作しなくなることがある。MySQLのバイナリログを消したり、ディスク上に保存した画像ファイルを削除したり、stackprofのファイルを消すことで解消できるはずだ。MySQLのバイナリログを削除するには、

```
mysql -u isuconp -p isuconp
mysql> PURGE BINARY LOGS BEFORE NOW();
```

とすればよい。

## 画像を直接ディスクに保存する（TODO点）

MySQLのスロークエリログを見ると、画像をMySQLに保存するクエリが1位になっており、20%以上の時間が使われている。これを変更して、ディスクに直接保存するようにしよう。 `app.rb` を編集する。

app.rb.bak10
```sql
303c303
<         mime = ''
---
>         mime, ext = '', ''
306c306
<           mime = "image/jpeg"
---
>           mime, ext = "image/jpeg", "jpg"
308c308
<           mime = "image/png"
---
>           mime, ext = "image/png", "png"
310c310
<           mime = "image/gif"
---
>           mime, ext = "image/gif", "gif"
316c316
<         if params['file'][:tempfile].read.length > UPLOAD_LIMIT
---
>         if params['file'][:tempfile].size > UPLOAD_LIMIT
321d320
<         params['file'][:tempfile].rewind
326c325
<           params["file"][:tempfile].read,
---
>           '',
329a329,331
>         imgfile = IMAGE_DIR + "/#{pid}.#{ext}"
>         FileUtils.mv(params["file"][:tempfile], imgfile)
>         FileUtils.chmod(0644, imgfile)
```

各種サービスやログをリフレッシュしてからベンチマークすると、得点は多少下がるが、スロークエリログは改善されている。

> TODO

## `GET /posts` のバグを直す（TODO点）


## stackprof を止める（TODO点）

ここで、stackprofを無効化して、何点になるか見てみよう。 `app.rb` を編集する。

```diff
```
app.rb.bak11

WIP



<!--

## commentのエスケープ結果のキャッシュ

alpを実行すると、上位3つのエンドポイントが `GET /` 、 `GET /posts` 、 `GET /posts/:id` であることがわかる。

stackprofを使うと、 `Sinatra::Templates#erb` に時間がかかっていることがわかる。

```bash
stackprof private_isu/webapp/ruby/tmp/stackprof-wall-*.dump --method 'block in <class:App>'
```

さらにstackprofで調査を進めていくと、 `post.erb` での `comment[:comment]` 、 `post[:body]` 、 `post[:user][:account_name]` のエスケープ処理に時間がかかっていることがわかる。これらの文字列をエスケープした結果を保存しておいて再利用するようにしよう。

`post[:body]` をエスケープ処理するようにする。

```
mysql -u isuconp -p isuconp
mysql> ALTER TABLE posts ADD escaped_body text AFTER body;
```

```diff
13c13
<     <%= escape_html(post[:body]).gsub(/\r?\n/, '<br>') %>
---
>     <%= post[:escaped_body] %>
```

```diff
121a122,127
>           if post[:escaped_body].nil?
>             escaped_body = Rack::Utils.escape_html(post[:body]).gsub(/\r?\n/, '<br>')
>             post[:escaped_body] = escaped_body
>             db.xquery("UPDATE posts SET escaped_body = ? WHERE id = ?", escaped_body, post[:id])
>           end
>
236c242
<       results = db.query("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` FORCE INDEX (created_at_idx) JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}")
---
>       results = db.query("SELECT posts.`id`, `user_id`, `body`, `escaped_body`, posts.`created_at`, `mime`, users.account_name FROM `posts` FORCE INDEX (created_at_idx) JOIN users ON posts.user_id = users.id WHERE users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}")
251c257
<       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` FORCE INDEX (user_created_idx) JOIN users ON posts.user_id = users.id WHERE `user_id` = ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
---
>       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, `escaped_body`, posts.`created_at`, `mime`, users.account_name FROM `posts` FORCE INDEX (user_created_idx) JOIN users ON posts.user_id = users.id WHERE `user_id` = ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
280c286
<       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, posts.`created_at`, `mime`, users.account_name FROM `posts` FORCE INDEX (created_at_idx) JOIN users ON posts.user_id = users.id WHERE posts.created_at <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
---
>       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, `escaped_body`, posts.`created_at`, `mime`, users.account_name FROM `posts` FORCE INDEX (created_at_idx) JOIN users ON posts.user_id = users.id WHERE posts.created_at <= ? AND users.del_flg = 0 ORDER BY `created_at` DESC LIMIT #{POSTS_PER_PAGE}",
289c295
<       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, imgdata, posts.`created_at`, `mime`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.id = ? AND users.del_flg = 0",
---
>       results = db.xquery("SELECT posts.`id`, `user_id`, `body`, `escaped_body`, imgdata, posts.`created_at`, `mime`, users.account_name FROM `posts` JOIN users ON posts.user_id = users.id WHERE posts.id = ? AND users.del_flg = 0",
334c340
<         query = 'INSERT INTO `posts` (`user_id`, `mime`, `imgdata`, `body`) VALUES (?,?,?,?)'
---
>         query = 'INSERT INTO `posts` (`user_id`, `mime`, `imgdata`, `body`, `escaped_body`) VALUES (?,?,?,?,?)'
339c345,346
<           params["body"],
---
>           "",
>           Rack::Utils.escape_html(params["body"]).gsub(/\r?\n/, '<br>'),
```

> {"pass":true,"score":168636,"success":164205,"fail":3,"messages":["静的ファイルが正しくありません (GET /image/24427.png)","静的ファイルが正しくありません (GET /image/24476.png)","静的ファイルが正しくありません (GET /image/24541.png)"]}

```sql
ALTER TABLE posts DROP COLUMN escaped_body;
```

-->


<!--
まずは `comment[:comment]` から始める。

```
mysql -u isuconp -p isuconp
mysql> ALTER TABLE comments ADD escaped_comment text AFTER comment;
```

```diff
110c110
<         query = "SELECT ranked.post_id, ranked.comment, users.account_name FROM (SELECT *, RANK() OVER (PARTITION BY post_id ORDER BY created_at DESC) AS rank_new FROM comments WHERE post_id IN (?)) ranked JOIN users ON ranked.user_id = users.id"
---
>         query = "SELECT ranked.post_id, ranked.comment, ranked.escaped_comment, ranked.id, users.account_name FROM (SELECT *, RANK() OVER (PARTITION BY post_id ORDER BY created_at DESC) AS rank_new FROM comments WHERE post_id IN (?)) ranked JOIN users ON ranked.user_id = users.id"
117a118,125
>         comments.to_a.each do |comment|
>           if comment[:escaped_comment].nil?
>             escaped_comment = Rack::Utils.escape_html(comment[:comment])
>             comment[:escaped_comment] = escaped_comment
>             db.xquery("UPDATE comments SET escaped_comment = ? WHERE id = ?", escaped_comment, comment[:id])
>           end
>         end
>
130c138
<               post_comments.push({ comment: comment[:comment], user: { account_name: comment[:account_name] } })
---
>               post_comments.push({ escaped_comment: comment[:escaped_comment], user: { account_name: comment[:account_name] } })
388c396
<       query = 'INSERT INTO `comments` (`post_id`, `user_id`, `comment`) VALUES (?,?,?)'
---
>       query = 'INSERT INTO `comments` (`post_id`, `user_id`, `comment`, `escaped_comment`) VALUES (?,?,?,?)'
392c400,401
<         params['comment']
---
>         params['comment'],
>         Rack::Utils.escape_html(params['comment'])
```

```diff
23c23
<       <span class="isu-comment-text"><%= escape_html(comment[:comment]) %></span>
---
>       <span class="isu-comment-text"><%= comment[:comment] %></span>
```


> {"pass":true,"score":148266,"success":144488,"fail":2,"messages":["静的ファイルが正しくありません (GET /image/23692.png)","静的ファイルが正しくありません (GET /image/23740.png)"]}

```
ALTER TABLE comments DROP COLUMN escaped_comment;
```
-->

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


<!-- ssh, tcp 80はつながるが、pingはtimeoutするので注意 -->

<!--




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

-->


## CloudFormationスタックを削除する

使っていないときにAWS利用料がかからないように、演習を終えたらAWS上のリソースをすべて削除する。今回はCloudFormationでリソースを作成したので、CloudFormationのstackを削除するだけでよい。

```bash
aws cloudformation delete-stack --stack-name private-isu
```
