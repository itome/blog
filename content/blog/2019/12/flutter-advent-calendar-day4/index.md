---
title: "すぐにFlutterを始めたい人のためのDart入門(後編)"
date: 2019-12-04T16:00:00+09:00
draft: false
---

この記事は[Flutter 全部俺 Advent Calendar](https://adventar.org/calendars/4140) 4日目の記事です。

## このアドベントカレンダーについて
このアドベントカレンダーは [@itome](https://twitter.com/itometeam) が全て書いています。

基本的にFlutterの公式ドキュメントとソースコードを参照しながら書いていきます。誤植や編集依頼はTwitterにお願いします。

この記事は[3日目の記事](https://itome.team/blog/2019/12/flutter-advent-calendar-day3/)の後編です。

## Null-aware演算子
DartはKotlinやSwiftのように型レベルのnullable/nonnulを持っていませんが、nullableな値を安全にアンラップするための演算子をいくつか持っています。

### `?.`

KotlinやSwiftではおなじみのnull-safetyなメソッド呼び出し

```dart
nullableObject?.doSomething();

// ↓と同じ
if (nullableObject != null) {
  nullableObject.doSomething();
}

// フィールドに対しても使える
var something = nullableObject?.something;
```

### `??`

Swiftと同じ、Kotlinでは `?:`

```dart
String str = nullableNum?.toString() ?? '';

// ↓と同じ
String str = nullableNum != null ? nullableNum.toString() : '';
```

### `??=`

左辺がnullのときだけ右辺の値を代入する

```dart
String str = null;
str ??= '';

// ↓と同じ
String str = null;
if (str == null) {
  str = '';
}
```

## カスケード演算子( `..` )

同じオブジェクトに対して連続で複数の処理をするときに便利な機能です。他の言語にはない珍しい文法だと思います。
Kotlinを知っている人なら `.apply {}` と同じように使えます。

```dart
var list = [0, 1, 2, 3];
list
  ..add(4)
  ..addAll([5, 6, 7]);
```

## Future

コールバックのネストを避けてメソッドチェーンで非同期処理を書くことができるクラスです。
APIリクエストなどのネットワークの待ち合わせに限らず、Flutterで頻出の機能です。Javascriptの `Promise` とほとんど同じです。

```dart
import 'package:http/http.dart' as http;

http
  .get(url)
  .then((response) {
    print(response);
  })
  .catchError((error) => print(error));
```

このようにメソッドチェーンで書くこともできますが、ほとんどは後述の `async/await` をつかって書きます。
筆者はFlutterアプリを作っていて `Future` のメソッドチェーンを使ったことはありません。

## Stream

Rx~系やJavaのStreamAPIと同等のライブラリが標準で用意されています。非同期に更新される複数の値を監視するのに使います。

```dart
someStream.listen((data){
  print(data);
});
```

`Stream` 自体は値を送出するためのメソッドを持っていないので、自分で値を流したいときは対になる `Sink` クラスと、`Stream` と `Sink` を管理する `StreamController` クラスを使います。

```dart
final controller = StreamController<String>();

controller.stream.listen((data) {
  print(data);
})

controller.sink.add('Hello');
```

StreamControllerにはRxの `PublishSubject` 相当の機能しかないので、 `BehaviorSubject` 相当のものがほしいときは、RxDartを使います。
RxDartは独自実装ではなく単なるStreamのラッパーライブラリです。

## async/await

Javascriptにある `async` / `await` と同じで、 `Future` をメソッドチェーンを使わずに手続き的に書くための機能です。
関数のブロックの前に `async` キーワードをつけることで、内部で `await` キーワードが使えるようになります。 `async` 関数の返り値は通常の返り値をFutureでラップしたものになります。

エラーハンドリングは通常の `try/catch` 文が使えます。

```dart
import 'package:http/http.dart' as http;

// Futureのメソッドチェーン
void sendRequest() {
  http
    .get(url)
    .then((response) {
      print(response);
    })
    .catchError((error) => print(error));
}

// async / await
Future<void> sendRequest() async {
  try {
    final response = await http.get(url);
    print(response);
  } on Exception caach (error) {
    print(error);
  }
}
```

### await for

`async` 関数の中では `await for` を使ってStreamの値の取り出しすることもできます。

```dart
Future<int> sumStream(Stream<int> stream) async {
  var sum = 0;
  await for (var value in stream) {
    sum += value;
  }
  return sum;
}
```

### async*/yield

`async` 関数が `Future` を返すのに対して、 `async*` 関数は `Stream` を返します。 `async*` 関数の中で `yield` キーワードを使うことで、返り値のStreamに値を流すことができます。

```dart
Stream<S> mapLogErrors<S, T>(
  Stream<T> stream,
  S Function(T event) convert,
) async* {
  var streamWithoutErrors = stream.handleError((e) => log(e));
  await for (var event in streamWithoutErrors) {
    yield convert(event);
  }
}
```

## 新しく追加された文法

Dartに最近追加された/今後追加される機能です。DartはFlutterに採用されてからかなりの新機能を追加しています。
便利な機能も多いので積極的にキャッチアップしていきましょう。

### スプレッド演算子( `...` ) [Dart2.3~]
Listをバラすことができます。childrenの合成を宣言的な構文を崩さずに書きたいときに便利です。

```dart
var list1 = [0, 1, 2, 3];
var list2 = [4, 5, 6];

var list3 = [...list1, ...list2]; // [0, 1, 2, 3, 4, 5, 6]
```

### Collection if [Dart2.3~]

Flutterのchildrenが書きやすいようになるためだけに生まれたような新機能です。childrenの要素に条件分岐が入るときでも宣言的な構文を崩さずにかけます。

```dart
Widget build(BuildContext context) {
  var children = [
    IconButton(icon: Icon(Icons.menu)),
    Expanded(child: title)
  ];

  if (isAndroid) {
    children.add(IconButton(icon: Icon(Icons.search)));
  }

  return Row(children: children);
}
```

↓

```dart
Widget build(BuildContext context) {
  return Row(
    children: [
      IconButton(icon: Icon(Icons.menu)),
      Expanded(child: title),
      if (isAndroid)
        IconButton(icon: Icon(Icons.search)),
    ],
  );
}
```

### Collection for [Dart2.3~]

Collection ifと一緒に追加された機能。こちらもFlutterの影響がかなり大きいです。

```dart
Widget build(BuildContext context) {
  
  return Row(
    children: titles.map((title) => Text(title)).toList(),
  );
}
```

```dart
Widget build(BuildContext context) {
  return Row(
    children: [
      for (final title in titles) Text(title),
    ],
  );
}
```

### Static extension methods [Dart2.6~]

KotlinやSwiftユーザーにとっては待望のextension methodsです。既存のクラスにあとから関数を追加することができます。
Dartの言語機能を補うようなライブラリも出始めているので、これからが楽しみです。

> https://github.com/leisim/dartx

```dart
extension MyFancyList<T> on List<T> {
  /// Whether this list has an even length.
  bool get isLengthEven => this.length.isEven;
}

[0, 1, 2].isLengthEven // -> false
```

## プロジェクト構成

ソースコードは `lib` ディレクトリ以下に置きます。ディレクトリ構造がそのままパッケージ構造になります。

外部ライブラリを読み込みたいときは、 `import` 文を使います。

```dart
import 'pakcage:{package_name}/{file_path}.dart';
```

例えばFlutterのMaterial Widgetをimportするときは以下のようになります。

```dart
import 'package:flutter/material.dart';
```

プロジェクト内の他のファイルを使いたいときは、絶対パスでimportする方法と相対パスでimportする方法があります。
筆者は基本的にすべて絶対パスで指定しています。

```text
project_name
|
|-- lib
     |-- a.dart
     |-- b.dart
     |-- c
         |-- d.dart
```

↑のようなディレクトリ構造で、 `a.dart` に `d.dart` をインポートしたいときは以下のようにできます。

```dart
import 'package:project_name/c/d.dart';

// もしくは
import './c/d.dart';
```

### pubspec.yaml
依存ライブラリの指定などのプロジェクト管理は、 `pubspec.yaml` ファイルで行います。
FlutterSDKのバージョン、アプリのバージョン、アプリ名、画像などの外部アセットの読み込みなどをこのファイルで指定します。
Nodeに触れたことのある人であれば、npmの `package.json` と同じようなものです。

依存ライブラリの指定は以下のように書きます。

```yaml
dependencies:
  flutter:
    sdk: flutter

  # The following adds the Cupertino Icons font to your application.
  # Use with the CupertinoIcons class for iOS style icons.
  cupertino_icons: ^0.1.2

dev_dependencies:
  flutter_test:
    sdk: flutter
```

`dependencies` と `dev_dependencies` がありますが、`dev_dependencies` は
テストやLintツール、Utility系のパッケージなど、成果物のアプリに含まれないパッケージです。

以下のコマンドで、依存パッケージのインストールができます。

`$ flutter pub get`

パッケージのインストールをすると、 `pubspec.lock` が自動生成されます。
これは、実際にインストールしたパッケージのバージョンを固定するもので、
開発環境によるパッケージのバージョンのばらつきを防ぐためのものです。特に理由がなければバージョン管理対象に含めましょう。
