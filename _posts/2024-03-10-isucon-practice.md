---
layout: post
title: "private-isuでISUCONの演習"
date: 2024-03-10
categories: プログラミング
---

[藤原俊一郎他『達人が教えるWebパフォーマンスチューニング〜ISUCONから学ぶ高速化の実践』](https://gihyo.jp/book/2022/978-4-297-12846-3)を買ったので、[private-isu](https://github.com/catatsuy/private-isu)でISUCONの演習をする。

## 環境立ち上げ

本の付録Aに実践例があり、そこではAWSで起動されているので、得点比較などのために同じくAWSで演習する。

AWS マネジメントコンソールからぽちぽち操作してEC2インスタンスを立ち上げると、気付かないうちにsubsetなどがどんどん作られて収拾がつかなくなるので、AWS CloudFormationで作りたい。

有志の方がCloudFormation templateを[公開されている](https://gist.github.com/tohutohu/024551682a9004da286b0abd6366fa55)ので、これを使う。念のため変なリソースが指定されていないか、特にAMI image IDがprivate-isu公式のものと一致しているかを確認しておく。

AWS CLIから、AWS CloudFormationのstackをdeployする。EC2 key pairとGitHubのアカウント（SSH公開鍵を設定しているもの）は事前に用意しておく。

```bash
wget https://gist.githubusercontent.com/tohutohu/024551682a9004da286b0abd6366fa55/raw/e8a86dd7195c18efc9322f32639fde8021062dd8/private-isu.yaml
aws cloudformation deploy --stack-name private-isu --region ap-northeast-1 --template-file private-isu.yaml --parameter-overrides KeyPairName=your_key_name GitHubUsername=your_github_name
hashi
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

## worker threadの引き上げ

リソースが全て活用できていない。リソースを活用するため、Unicornのworker processを増加させる。

Unicornの設定ファイルを開き、worker_processesを1から10に変更する。その後、サーバーを再起動する。

```bash
nano private_isu/webapp/ruby/unicorn_config.rb
sudo systemctl restart isu-ruby
```

10に設定したのは、CPUのコア数が2つ（ref. `less /proc/cpuinfo` ）であり、書籍に

> 筆者の経験では、プロセス外部のミドルウェアとの通信が多い典型的なWebアプリケーションの場合、CPUコア数の5倍程度を設定するのが適切な場合が多くありました。

と書いてあったため。

再度ベンチマーカーを実行すると、CPU使用率がmysqldが200%、メモリ使用量が1GB程度になる。また、リクエスト失敗が増えて得点が0に下がってしまう。

> {"pass":true,"score":0,"success":174,"fail":57,"messages":["リクエストがタイムアウトしました (GET /)","リクエストがタイムアウトしました (GET /@bessie)","リクエストがタイムアウトしました (GET /@irma)","リクエストがタイムアウトしました (GET /@janet)","リクエストがタイムアウトしました (GET /@marquita)","リクエストがタイムアウトしました (GET /@shelly)","リクエストがタイムアウトしました (GET /image/6910.jpg)","リクエストがタイムアウトしました (GET /posts/9399)","リクエストがタイムアウトしました (POST /login)","リクエストがタイムアウトしました (POST /register)"]}

## MySQLへのindexの追加


