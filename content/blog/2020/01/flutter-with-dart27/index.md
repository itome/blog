---
title: "Dart2.7時代のFlutter"
date: 2020-01-06T12:33:10+09:00
draft: false
comments: true
images:
---

Flutterのバージョン1.12.13が正式リリースされたときに、
同時に同梱されているDartのバージョンが2.7になりました。

Dartは2.0のリリース以降、Flutterのための多くの言語拡張をしていて、
以前よりもFlutterのコードをシンプルに書くことができるようになりました。

しかし、Flutterの記事や公式のサンプルでも、
Dart1.0系の文法で書かれているものも多く、
またバージョン2.3以降の機能はFlutterのデフォルトの設定では使えないこともあって
まだまだ浸透していません。

筆者としては、Flutter開発にとってDartの後方互換性よりも、
新しい便利な機能を使えることのほうが重要だと考えています。

そこで、この記事では、Dart2.7時代のFlutter開発の変化をできるだけ網羅的に紹介していきます。

## FlutterプロジェクトでDart2.7を使えるようにする
`$ flutter create`コマンドで作るFlutterプロジェクトの雛形は、
Dart2.1以上で動く設定になっています。

このままでもローカル環境でDart2.7があれば新しい機能を使うことができますが、
Dart2.1環境で文法エラーが出てしまったり、コード中に後方互換性に関する警告が出たりします。

FlutterプロジェクトでどのDartのバージョンを使うかは、`pubspec.yaml`で指定できます。

```yaml
 environment:
-   sdk: ">=2.1.0 <3.0.0"
+   sdk: ">=2.7.0 <3.0.0"
```

## `new`キーワードのOptional化(Dart2.0)
FlutterはWidgetのインスタンス化によって画面を組み立てていくので、
大量のコンストラクタ呼び出しがあります。

Dart1.0系では、コンストラクタの呼び出しには以下のように明示的に`new`キーワードを使う必要がありましたが、
2.0以降ではつけてもつけなくても良くなりました。

現在では、つけないことが推奨されています。

**~2.0**
```dart
  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: new AppBar(title: const new Text('sample')),
      body: const new Center(
        child: new Text('Hello world'),
      ),
    );
  }
```

**2.0~**
```dart
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('sample')),
      body: const Center(
        child: Text('Hello world'),
      ),
    );
  }
```

## `int`→`double`への自動キャスト(Dart2.1~)
Flutterでは、文字、画像、アイコン、その他Widgetのサイズの指定に`double`型を多用します。
以前は`double`型の引数には明示的に`double`型の値(`1`でなく`1.0`)を渡す必要がありましたが、
Dart2.1以降では`int`型を渡すことができるようになりました。

**~2.1**
```dart
  Text('Hello world', style: TextStyle(fontSize: 14.0)),
```

**2.1~**
```dart
  Text('Hello world', style: TextStyle(fontSize: 14)),
```

## Spread演算子(Dart2.3~)
Dart2.3は、公式のリリースノートにも、以下のようにあるとおり、FlutterでUIのを作りやすくすることを
目的としたアップデートでした。

> Flutter is growing rapidly, which means many Dart users are building UI in code out of big deeply-nested expressions. 
> Our goal with 2.3.0 was to make that kind of code easier to write and maintain. 
>
> Flutterは急速に成長しており、そのため多くのDartユーザーは深くネストされた大きな式からUIを作っています。
> 2.3.0の私達の目的は、そのような種類のコードを、より簡潔に、メンテナンスしやすいものにすることです。

そのためFlutter開発が便利になるような、特にリスト操作に関する文法がいくつか追加されています。

Spread演算子はそのうちの１つめです。

Javascriptユーザーには馴染み深い文法ですが、リストをその場に展開することができます。

```dart
[1, 2, 3, ...[4, 5], 6] // [1, 2, 3, 4, 5, 6]
```

これは、例えばセルとDividerをリストで返すような関数を使うときに役立ちます。

**~2.3**
```dart
List<Widget> _buildItemAndDivider() {
  return [
    Text('Hello world'),
    Divider(),
  ];
}

@override
Widget build(BuildContext context) {
  return Column(
    children: [
      Text('Hello`),
    ]..addAll(_buildItemAndDivider())
      ..add(Text('Thank you!')),
  );
}
```

**2.3~**
```dart
List<Widget> _buildItemAndDivider() {
  return [
    Text('Hello world'),
    Divider(),
  ];
}

@override
Widget build(BuildContext context) {
  return Column(
    children: [
      Text('Hello`),
      ..._buildItemAndDivider(),
      Text('Thank you!'),
    ],
  );
}
```

ちなみにJavascriptのようなオブジェクトの展開はできません。

## Collection if
例えば`Column`Widgetの`children`にある条件が正であるときだけ`Text`Widgetを追加したいとします。

これまでは、以下のように先にリストを作っておいて条件が成立するときにだけ`add`するか

```dart
Widget build(BuildContext context) {
  var children = [
    IconButton(icon: Icon(Icons.menu)),
    Expanded(child: title)
  ];

  if (isAndroid) {
    children.add(Text('Hello world'));
  }

  return Column(children: children);
}
```

以下のように条件が成立しないときに`null`になるようにしておいて`filter`で弾く必要がありました。

```dart
Widget build(BuildContext context) {
  return Column(
    children: [
      IconButton(icon: Icon(Icons.menu)),
      Expanded(child: title)
      isAndroid ? Text('Hello world') : null,
    ].filter((widget) => widget != null).toList(),
  );
}
```

Dart2.3以降のCollection ifを使うことで以下のように書くことができます。

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

Spread演算子と組み合わせることによって、複数のWidgetを条件によって出し分けることもできます。

```dart
Widget build(BuildContext context) {
  return Row(
    children: [
      IconButton(icon: Icon(Icons.menu)),
      if (isAndroid) ...[
        Expanded(child: title),
        IconButton(icon: Icon(Icons.search)),
      ]
    ],
  );
}
```

## Collection for
データのリストをWidgetの`children`として展開するために、これまでは`Iterable.map`関数を使っていました。

```dart
final animals = ['Dog', 'Cat', 'Wolf'];

Widget build(BuildContext context) {
  return Column(
    children: animals.map((animal) => Text(animal),
  );
}
```

これはこれで良さそうですが、例えば複数のリストを変換したいときには、結局以下のように書かなくてはいけませんでした。

```dart
final animals = ['Dog', 'Cat', 'Wolf'];
final foods = ['Meet', 'Fish', 'Bread'];

Widget build(BuildContext context) {
  final children = [];
  for (final animal in animals) {
    children.add(Text(animal));
  }
  for (final food in foods) {
    children.add(Center(Text(food)));
  }
  
  return Column(children: children);
}
```

Collection forを使えばこのようなコードを簡潔に書くことができます

```dart
final animals = ['Dog', 'Cat', 'Wolf'];
final foods = ['Meet', 'Fish', 'Bread'];

Widget build(BuildContext context) {
  return Column(
    children: [
      for (final animal in animals) Text(animal),
      for (final food in foods) Center(Text(food)),
    ]
  );
}
```

Spread演算子、Collection if/forを組み合わせることで、
複雑なリストでも宣言的に組み立てることができるようになります。

```dart
Widget build(BuildContext context) {
  return Column(
    children: [
      for (final animal in animals) Text(animal),
      if (isAndroid) ...[
        for (final food in foods) Center(Text(food)),
      ]
    ]
  );
}
```

## static extension methods (Dart2.7~)
Dart2.6でプレビューとして公開された拡張関数が2.7で正式リリースとなりました。
Swiftの拡張関数に似た文法で、使い方次第でいろいろな可能性を持っています。

例えば、`Theme.of(context)`のような
static methodを使って以下のように書くことができるようになります。

**~2.7**
```dart
  @override
  Widget build(BuildContext context) {
    return Text(
      'Hello world',
      style: Theme.of(context).textTheme.body1,
    );
  }
```

**2.7~**
```dart
extension BuildContextExt on BuildContext {
  ThemeData get theme {
    return Theme.of(this);
  }
}

...

  @override
  Widget build(BuildContext context) {
    return Text(
      'Hello world',
      style: context.theme.textTheme.body1,
    );
  }
```

拡張関数を使って`AutoDispose`を実現するコードも以下の記事で書いているので、
より具体的な利用方法はそちらを読んでください。

> **MixinとStatic Extension Methodを使ってAutoDispose**
>
> https://itome.team/blog/2019/12/flutter-auto-dispose/

## Non-nullable by default(NNBD) (3.0~?)
**※ NNBDはプレビューです**

DartにはNull安全な関数呼び出し(`?.`)やNullガード(`??`)があるものの、
型レベルでNullableかNonNullかを判断する方法がないため、Null安全な言語とは言えません。

しかし、以下のスレッドで議論が活発に行われており、Dart2.7ではプレビューながらNNBDが
追加されました。

> https://github.com/dart-lang/language/issues/110

これまでの`String hoge = 'Hello world'`のような型宣言では、全てNonNullである型として扱われるようになり、
`String hoge = null`のように`null`を代入することはできなくなります。

代わりに`String?`型を用いることで、型がNullableであることが示されます。

Nullableな変数は`?.`などのNull安全な呼び出しを矯正されるので、NNBDが導入されたDartは
型安全であると言えます。
