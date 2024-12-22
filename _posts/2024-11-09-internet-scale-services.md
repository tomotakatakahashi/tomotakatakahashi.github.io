---
layout: post
title: "「On Designing and Deploying Internet-Scale Services」を読んだ"
date: 2024-11-09
categories: プログラミング
---

「[On Designing and Deploying Internet-Scale Services](https://www.usenix.org/legacy/events/lisa07/tech/full_papers/hamilton/hamilton.pdf)」を読んだので、気になったところを中心に感想を書く。

2007年にMicrosoftのWindows Live Services Platformの人が発表した論文で、運用効率のいいシステムを作るためのベストプラクティスについて説明している。

> Partition the service.

関連情報として、[Amazon S3はprefixでpartitioningしている](https://dev.classmethod.jp/articles/amazon-s3-announces-increased-request-rate-performance/)という話がある。

> Understand access patterns.

新機能を計画するときにはbackendにどのような負荷がかかるか考えておけということで、それはそう。新機能の負荷で既存機能が破壊されるのは困るというのはよくわかる。

> Keep the unit/functional tests from the previous release.

リリースごとに自動テストを消そうとする人は私は見たことがないのだけれど、2007年当時は珍しくなかったということなのだろうか？

> Fail services regularly.

Chaos Monkeyっぽい考えがすでに2007年当時あったということに驚いた。

> Another potentially counter-intuitive approach we favor is deployment mid-day rather than at night.

不測の事態に備えて、利用者が少ない深夜にデプロイしたい気もするが、周りに相談相手の多い昼にやったほうがいいということか。直感には反するが、なるほどという感じ。

> Ship often.

頻繁にリリースしたほうがいいというのは、それはそう。リリースの機会が増えるとプロセスも洗練される。開発計画も柔軟に立てやすくなる。
3ヶ月ごとのリリースと書いてあるが、17年経っている現在、Microsoft社内がどうなっているかは気になる。

> Stress test for load.

> Perform capacity and performance testing prior to new releases.

言われてみればそれはそう、ではあるのだが、言うは易く行うは難し、新機能を開発し終わったと思ったところに追加で負荷テストをするのは簡単なことではないような気がしている。負荷テストをサボったことで後から問題を起こすほうがよっぽど対処が大変ではあるのだが。

> Soft delete only.

なるほど確かにそうだろうとは思うが、実際の事故を当事者として見たことがないので実感は湧かない。

> Make everything configurable.

まあ言っていることはわかるのだけど、これもありがたみが体感としてはわからず、実感が湧かない。

> Any time there is a configuration change, the exact change, who did it, and when it was done needs to be logged in the audit log.

これはその通りだし、普段の仕事で気をつけたいと思った。

> Track all fault tolerance mechanisms.

2000台のシステムが400台しか動かない状態になっていたのに最初は気づかなかったという話が面白い。プログラミングの世界でいうところの「例外を握りつぶすな」というのと似ていると思う。

> Systems fail, and there will be times when latency or other issues must be communicated to customers.

システムが落ちたとき用に、ウェブなどのコミュニケーション手段を用意しておけというのは、なるほど。GitHubとかAWSとかはたしかにそういうWebページがある。小規模なサービスでもTwitterで告知をしていたりする。無料サービスでもやっておけというのもなるほど。
