---
title: "FlutterでAndroid/iOSのネイティブのAPIを使う"
date: 2019-12-04T17:44:04+09:00
draft: true
comments: true
images:
---

この記事は[Flutter 全部俺 Advent Calendar](https://adventar.org/calendars/4140) 1日目の記事です。


## このアドベントカレンダーについて
このアドベントカレンダーは [@itome](https://twitter.com/itometeam) が全て書いています。

基本的にFlutterの公式ドキュメントとソースコードを参照しながら書いていきます。誤植や編集依頼はTwitterにお願いします。

## FlutterからAndroid/iOSのAPIにアクセスする
FlutterはUIの描画こそ自前で完結させることができますが、ハードウェアの情報へのアクセスや
ネイティブでしか提供されていないSDKの利用など、どうしてもネイティブのAPIにアクセスする必要があることもあります。

Flutterでは`PlatformChannel`という仕組みを使って、ネイティブのAPIにアクセスすることができます。

以下の[公式ドキュメント](https://flutter.dev/docs/development/platform-integration/platform-channels)の
図がわかりやすいです。

![platform channel](./platform_channels.png)

Dartとネイティブ側で、チャンネル名を使って同じ`MethodChannel`にアクセスします。`MethodChannel`は糸電話のようなものなので、
両側が受信も発信もすることができます。

## やりとりできるデータ型
`MethodChannel`に流せるデータ型は以下の表にあるものです。データ型の共有などはできません。

| Dart                       | Android             | iOS                                            |
|----------------------------|---------------------|------------------------------------------------|
| null                       | null                | nil (NSNull when nested)                       |
| bool                       | java.lang.Boolean   | NSNumber numberWithBool:                       |
| int                        | java.lang.Integer   | NSNumber numberWithInt:                        |
| int, if 32 bits not enough | java.lang.Long      | NSNumber numberWithLong:                       |
| double                     | java.lang.Double    | NSNumber numberWithDouble:                     |
| String                     | java.lang.String    | NSString                                       |
| Uint8List                  | byte[]              | FlutterStandardTypedData typedDataWithBytes:   |
| Int32List                  | int[]               | FlutterStandardTypedData typedDataWithInt32:   |
| Int64List                  | long[]              | FlutterStandardTypedData typedDataWithInt64:   |
| Float64List                | double[]            | FlutterStandardTypedData typedDataWithFloat64: |
| List                       | java.util.ArrayList | NSArray                                        |
| Map                        | java.util.HashMap   | NSDictionary                                   |

## 実際に実装してみる
今回はFlutterのアプリからOSのダークモードを有効にするネイティブのAPIにアクセスできるようなコードを書いていきましょう。
対象はバージョン10以上のAndroidとします。
