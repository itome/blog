---
title: "Flutter2.0で新しくなったこと"
date: 2021-03-04T10:00:00+09:00
draft: false
comments: true
images: []
---

日本時間の3/4の未明に行われた[Flutter Engage](https://events.flutter.dev/)でFlutter2.0が発表されました。

変更点をまとめていきます。

## Web/Windows/MacOS/LinuxのサポートがStableに
元々モバイル向けのクロスプラットフォームであったFlutterは、ベータ機能としてWeb、 Windows、 MacOS、Linuxをサポートしていましたが、
Flutter2.0でこれらのサポートがStableになりました。

従来の「モバイルフレームワーク」から、「ポータブルフレームワーク」へ変わるぞ！と発表されています。

Flutter for WebではWebassemblyとSkiaを使って直接CanvasにUIを描画する[CanvasKit](https://skia.org/user/modules/canvaskit)が紹介されました。
これは従来のdomを直接操作する方法と比べてパフォーマンスに優れ、モバイル版との差異も少ないレンダリング方法ですが、
Skiaをダウンロードしないといけないため、ダウンロードサイズが2mb増加します。

どちらのレンダリング方法を洗濯するかは開発者が選択でき、デフォルトではモバイルの場合は従来の手法を使い、デスクトップの場合にはCanvasKitを使うように設定されています。

> https://flutter.dev/docs/development/tools/web-renderers

そのほかにもPWA Manifest、URLの書き換えをサポートすることでPWAやSPAでも動作することを目指しているそうです。

<br/>

Linuxでは、UbuntuがデフォルトのGUIフレームワークとしてFlutterを採用したり

{{<tweet 1367063203600031746>}}

WindowsやSurfaceDuoのようなフォルダブルサポートのためにMicrosoftと協力していたり

{{<tweet 1367169456183472128>}}

Toyotaが自動車のUIにFlutter採用を予定していたりなど

{{<tweet 1367174378387988480>}}

有名企業の採用例が紹介されました。

Google以外のビッグカンパニーがFlutterを採用したりFlutterの開発に積極的に参加していたりすることはコミュニティーの活性化を意味するのでとてもいいことですね。

## AutoCompleteCoreとScaffoldMessangerの追加
`AutoCompleteCore` が追加されたことでFlutterでも公式にAutoCompleteができるようになりました。

また、 `ScaffoldMessanger` が追加されたことで、これまお世辞にも使いやすいとは言えなかった `SnackBar` の仕様が改善され、画面遷移を跨ぐような使い方もできるようになりました。

## iOSもちゃんとやってるよ
`Cupertino` 系のFormWidgetがいくつか追加されました。[iOSのパフォーマンス改善](https://github.com/flutter/flutter/issues/60267#issuecomment-762786388)も引き続きやっていくそうです。

## Google Mobile Ads SDK
Flutterは15000以上のパッケージを抱えていますが、今回そこにGoogle Mobile Ads SDKが追加されました。

> https://pub.dev/packages/google_mobile_ads

こういったネイティブのAPIを使うようなパッケージはFlutterが苦手とするところでしたが、コミュニティの発展に伴ってサポートされている機能が随分充実してきました。

## スクロールバーが操作可能に
Flutterのスクロールバーはこれまで単にスクロール位置を表示するためだけのものでしたが、ドラッグやクリックができるようになりました。

主にWebやDesktopでの使いやすさのためのアップデートみたいです。

## Sounds Null Safety
Flutter2と同時にDartの2.12が発表され、待望のNull Safetyがリリースされました。

移行は依存しているパッケージのNull Safety対応が完了してからすることが推奨されているのでもう少し時間が必要かもしれませんが、
Firebaseなどの有名パッケージを含む1000以上のパッケージがすでに対応済みのようなので、今年中くらいには完全移行できるかもしれません。

## Dart FFIがStableに
Null Safetyと同時にDart2.12からFFIがStableになりました。FFIはForeign function interfaceの略で、別言語（主にC言語）で書かれた機能にDartから直接アクセスすることを可能にします。

これまで同様のことをFlutterでしようとした場合、

```
Dart → Message Channel → 各プラットフォームのFFI → C言語の機能
```

のように一度JavaやObjective-Cを経由しないといけなかったため面倒でしたが、これからはDart FFI経由で直接アクセスできるようになります。

## Add-to-appを同一アプリ内で複数使ったときのパフォーマンスが改善
既存のアプリに一部だけFlutterを導入するAdd-to-appは、Flutterの恩恵を部分的に受けることができる機能でしたが、
同一アプリ内で複数のインスタンスを利用した場合、インスタンスごとにFlutterのコアが起動されてしまい非常にメモリ効率が悪くなってしまいました。

Flutter2からはこれが改善され、同時起動時の2つめ以降のインスタンスにかかるメモリコストが最大99%削減され、180kbほどで立ち上げることができるようになりました。

それによって、Widget単位でFlutterを導入したりなどのこれまでできなかった使い方ができるようになります。


## `dart fix` コマンドの追加

Deprecatedになったコードの一覧表示、自動修正をしてくれる  `dart fix` コマンドが追加されました。
コマンドラインからだけでなく、InttelijやVSCodeからの修正にも対応しています。

![Inttelij上での自動修正](https://miro.medium.com/max/1204/0*I9GWJb-XsUhJUKrH)

## Flutter DevToolsが新しくなった

これまでは単に `Dev Tools` と呼ばれていましたが、わかりやすくするために `Flutter DevTools` に改名されたそうです。

以下の機能が追加されています。

- Flutter DevToolsを起動前に、エラーが怒ったタイミングでIntellijやVSCodeコードがFlutter DevToolsの利用を提案してくれるようになった

- Flutter DevToolsのタブにエラーのバッジがつくようになった

- Invert Oversized Image機能が追加され、有効化すると画面解像度に対して大きすぎる画像を上下反転してくれるようになった

- FlexレイアウトのWidgetだけでなく、 `Text` などのFixedレイアウトのWidgetに関してもサイズやPaddingなどの詳細がみられるようになった

- 失敗したネットワーク経由のリクエストが赤く強調されるようになった

- FPSの平均値を出してくれるなど、Flutter frames chartが使いやすくなった

- メモリの監視UIが改善された

- Logが検索、絞り込みできるようになった

- DevTools起動前のログも確認できるようになった

- `Performance` View が `CPU Profiler` に改名された

- `Timeline` Viewが `Performance` に改名された

- ...etc

## まとめ
Null Safetyがリリースされたり、よりマルチプラットフォームになっていたり着実に進化していってるなと思います。
大企業も積極的に開発に参加していて、これからの進展に期待ができそうです。

## 参考

> https://developers.googleblog.com/2021/03/announcing-flutter-2.html
>
> https://medium.com/flutter/flutter-web-support-hits-the-stable-milestone-d6b84e83b425
>
> https://medium.com/dartlang/announcing-dart-2-12-499a6e689c87
