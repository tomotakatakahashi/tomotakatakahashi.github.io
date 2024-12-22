---
layout: post
title: "Compose MultiplatformプロジェクトをXcode Cloudでビルドする"
date: 2025-01-19
categories: プログラミング
---

[Compose Multiplatform](https://www.jetbrains.com/compose-multiplatform/)のプロジェクトをXcode CloudでビルドするときのJavaの設定について書く。

## 前提
プロジェクトは、[Kotlin Multiplatform Wizard | JetBrains](https://kmp.jetbrains.com/)からAndroid、iOS、Desktop、Webの4つをチェックし、iOSについては「Share UI」もチェックしてからダウンロードしたものをベースにしている。
また、 `composeApp/build.gradle.kts` ファイルでJava 11を指定している箇所が3箇所あったので、すべてJava 17を指定するように置換している。

## Java問題
初期状態で[Xcode Cloudのビルドを設定](https://developer.apple.com/documentation/xcode/configuring-your-first-xcode-cloud-workflow/)すると、 `Run xcodebuild archive` のステップで以下のエラーが出る。

```
Run custom shell script 'Compile Kotlin Framework'
The operation couldn’t be completed. Unable to locate a Java Runtime.
Please visit http://www.java.com for information on installing Java.
```

### Java 17のインストール

Xcode CloudからJavaのプログラムを実行できるように、Java RuntimeをXcode Cloud上でインストールするようにする。

リポジトリの `iosApp/ci_scripts/ci_post_clone.sh` に以下の内容のファイルを置く。ここにスクリプトを置いておくと、[Gitリポジトリをcloneしたあとで自動的に実行される](https://developer.apple.com/documentation/xcode/writing-custom-build-scripts)。なお、[Xcode CloudではHomebrewが最初から使えるようになっている](https://developer.apple.com/documentation/xcode/making-dependencies-available-to-xcode-cloud)。

```sh
#!/bin/sh

brew install openjdk@17
```

そのままでも動作するが、警告が出てしまうので実行権限を付与する。

```sh
chmod +x iosApp/ci_scripts/ci_post_clone.sh
```

### 環境変数の設定

このままではまだJavaが見つけられないというエラーが出てしまう。

Homebrewからのメッセージでは以下のように指示をされるが、Xcode Cloud環境では `sudo` コマンドが実行できないようなので、これはそのままは使えない。

```
For the system Java wrappers to find this JDK, symlink it with
  sudo ln -sfn /usr/local/opt/openjdk@17/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-17.jdk
```

そこで、Xcode Cloudの Manage Workflows > Edit > Environment > Environment Variables から、 `NAME` を `JAVA_HOME` 、 `VALUE` を `/usr/local/opt/openjdk@17` とする環境変数を追加する。

すると、Compile Kotlin FrameworkがJavaを見つけられるようになり、ビルドが成功する。

## その他の注意点

`iosApp/Configuration/Config.xcconfig` の `APP_NAME` の値を日本語にしてしまうと、 `Missing or invalid signature.` というエラーになってしまうようだ。関連する公式ドキュメントはおそらくこのあたり。

- [Technical Note TN2407: iOS Code Signing Troubleshooting Index](https://developer.apple.com/library/archive/technotes/tn2407/_index.html#//apple_ref/doc/uid/DTS40014991-CH1-LIST_OF_CODE_SIGNING_ERRORS-SIGNATURE_VERIFICATION_ERRORS)
- [Technical Note TN2318: Troubleshooting Failed Signature Verification](https://developer.apple.com/library/archive/technotes/tn2318/_index.html#//apple_ref/doc/uid/DTS40013777-CH1-TNTAG0-AVOID_SPECIAL_CHARACTERS_IN_EXECUTABLE_NAMES)

## 参考
- [Compose Multiplatform を Xcode Cloud で使う際にハマったポイント #iOS - Qiita](https://qiita.com/tsumuchan/items/bdc91121b54bff4c32c6)
- [Writing custom build scripts \| Apple Developer Documentation](https://developer.apple.com/documentation/xcode/writing-custom-build-scripts)
- [Making dependencies available to Xcode Cloud \| Apple Developer Documentation](https://developer.apple.com/documentation/xcode/making-dependencies-available-to-xcode-cloud)
- [Technical Note TN2407: iOS Code Signing Troubleshooting Index](https://developer.apple.com/library/archive/technotes/tn2407/_index.html#//apple_ref/doc/uid/DTS40014991-CH1-LIST_OF_CODE_SIGNING_ERRORS-SIGNATURE_VERIFICATION_ERRORS)
- [Technical Note TN2318: Troubleshooting Failed Signature Verification](https://developer.apple.com/library/archive/technotes/tn2318/_index.html#//apple_ref/doc/uid/DTS40013777-CH1-TNTAG0-AVOID_SPECIAL_CHARACTERS_IN_EXECUTABLE_NAMES)
