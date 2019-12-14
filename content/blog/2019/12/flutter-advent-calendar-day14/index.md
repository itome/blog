---
title: "FlutterでAndroid/iOSのネイティブのAPIを使う"
date: 2019-12-14T12:00:00+09:00
draft: false
comments: true
images: ["/blog/2019/12/flutter-advent-calendar-day14/platform_channels.png"]
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
今回はFlutterのアプリからデバイスのフラッシュライトにアクセスする機能を実装してみます。
対象はバージョン10以上のAndroidとします。

### Android側の実装
Androidで`MethodChannel`を受け付けて、Flutterからのメッセージを受け取れるようにします。
`<project_root>/android/app/src/main/kotlin/<your>/<domain>/<project_name>/MainActivity.kt`
を以下のように書き換えます。

```kotlin
package com.example.flash_light

import android.content.Context
import android.hardware.camera2.CameraManager
import android.os.Build
import android.os.Bundle

import io.flutter.app.FlutterActivity
import io.flutter.plugin.common.MethodChannel
import io.flutter.plugins.GeneratedPluginRegistrant

class MainActivity : FlutterActivity() {
    companion object {
        private const val CHANNEL_NAME = "com.example/flash_light"
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        GeneratedPluginRegistrant.registerWith(this)


        MethodChannel(flutterView, CHANNEL_NAME).setMethodCallHandler { methodCall, result ->
            when (methodCall.method) {
                "setTorchMode" -> (methodCall.arguments as HashMap<String, *>)["enabled"]?.let {
                    setTorchMode(it as Boolean, result)
                }
                else -> result.notImplemented()
            }
        }
    }

    private fun setTorchMode(enabled: Boolean, result: MethodChannel.Result) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            val cameraManager = getSystemService(Context.CAMERA_SERVICE) as CameraManager
            val cameraId = cameraManager.cameraIdList[0]
            cameraManager.setTorchMode(cameraId, enabled)
            result.success(enabled)
        } else {
            result.error("UNAVAILABLE", "Flash not available", null)
        }
    }
}
```

いくつかポイントがあります。

- `MethodChannel(flutterView, CHANNEL_NAME).setMethodCallHandler` でFlutter側から来たメッセージのハンドラをセットする。

`FlutterView`と`MethodChannel`の名前を指定して`MethodChannel`を初期化します。
`MethodChannel`の名前はFlutter側と合わせる必要があります。特に命名ルールは決まっていませんが
ライブラリ間でチャンネル名の衝突を避けるために`<reverse_domain>/<package_name>`で書くのがスタンダードです。

`setMethodCallHandler { methodCall, result -> ... }` の`methodCall`からメッセージの内容が取得でき、`result`で
返り値や成功/失敗を返すことができます。

- `MethodCall`から取得できる情報を確認する

`methodCall.method`で、呼ばれたメソッド名を`String`型で取得できます。メソッドに渡された引数は`methodCall.arguments`で取得できます。
`methodCall.method`の型はFlutter側の呼び出しに依存します。今回は`HashMap`型で引数を渡しています。

- `Result`から値を返す

`result.success(<somevalue>)`で任意の値を返すことができます。ネイティブ側でエラーが発生した場合は`result.error("message")`で
エラーを通知することができます。Flutter側から未定義のメソッドが呼び出された場合は`result.notImplemented()`で、未定義であることを返してあげましょう。

### Flutter側の実装

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

void main() => runApp(MyApp());

class MyApp extends StatefulWidget {
  @override
  _MyAppState createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  static const channel = MethodChannel('com.example/flash_light');

  bool _isFlashLightOn = false;

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData(primarySwatch: Colors.blue),
      home: Scaffold(
        appBar: AppBar(title: Text('Flash Light Sample')),
        body: Center(
          child: RaisedButton(
            onPressed: _setTorchMode,
            child: Text('Flash Light: ${_isFlashLightOn ? "ON" : "OFF"}'),
          ),
        ),
      ),
    );
  }

  void _setTorchMode() async {
    final isEnabled = await channel.invokeMethod<bool>(
      "setTorchMode",
      {"enabled": !_isFlashLightOn},
    );
    setState(() => _isFlashLightOn = isEnabled);
  }
}
```

Flutter側はシンプルです。

- `MethodChannel`を初期化する

Android側で指定したチャンネル名と合わせた名前で`MethodChannel`を初期化します。

- `MethodChannel`を使ってネイティブ側にメッセージを送る

`channel.invokeMethod`でメソッド名と引数名を指定します。こちらも先程Android側で使ったメソッド名と合わせます。

これでFlutterからデバイスのフラッシュライトにアクセスすることができるようになりました。
自分で試して実際にフラッシュライトがつくのを確認してみてください。

`PlatformChannel`を使えば簡単にネイティブのAPIにアクセスすることができます。
Android/iOSのみで提供されている外部SDKを使ったり、自前の`PlatformView`とのやり取りに使ったりなど、
実用性の高いアプリを作るのに便利な機能なので、ぜひ試してみてください。

<br/>

> **13日目: FlutterのPlatformViewを理解する** :
>
> https://itome.team/blog/2019/12/flutter-advent-calendar-day13
>
> **15日目: Flutterのアニメーションを理解する(前編)** :
> https://itome.team/blog/2019/12/flutter-advent-calendar-day15
