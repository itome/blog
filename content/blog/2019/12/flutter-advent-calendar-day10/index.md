---
title: "FlutterのNavigationとRoutingを理解する"
date: 2019-11-20T18:25:10+09:00
draft: true
comments: true
images:
---

この記事は[Flutter 全部俺 Advent Calendar](https://adventar.org/calendars/4140) 10日目の記事です。


## このアドベントカレンダーについて
このアドベントカレンダーは [@itome](https://twitter.com/itometeam) が全て書いています。

基本的にFlutterの公式ドキュメントとソースコードを参照しながら書いていきます。誤植や編集依頼はTwitterにお願いします。

## Flutterの画面遷移
FlutterではすべてがWidgetなので画面もまたWidgetで、画面内の他のWidgetと明確な区別はありません。

`Route` というWidgetが一画面を表していて、 `Navigator` によって表示する `Route` を切り替えることによって画面遷移が実現されています。

## 使い方
内部の実装を詳しく紹介する前に基本的な使い方をみていきましょう。

ほとんどの場合 `MaterialApp` が持っている `Navigator` を使って画面遷移します。
`Navigator` を自分でインスタンス化して管理することはまれです。

以下のように、 `Widget` を `MaterialPageRoute` に渡して `Route` を作って、
`Navigator` の `push` メソッドで画面として表示することができます。 `Navigator.of(context)` の部分は
`MaterialApp` Widgetより下の `BuildContext` でしか使えないので注意が必要です。
([6日目の記事](https://itome.team/blog/2019/12/flutter-advent-calendar-day6)参照)

```dart
Navigator.of(context).push(
  MaterialPageRoute<void>(
    builder: (BuildContext context) {
      return Scaffold(
        appBar: AppBar(title: Text('Hello')),
        body: ...,
      );
    },
  ),
);
```

`MaterialApp` に予め `Route` を登録しておくことによってさらにシンプルに書くことができます。

```dart
runApp(
  MaterialApp(
    initialRoute: '/',
    routes: <String, WidgetBuilder> {
      '/': (BuildContext context) => MyPage(title: 'initial page'),
      '/a': (BuildContext context) => MyPage(title: 'page A'),
      '/b': (BuildContext context) => MyPage(title: 'page B'),
    },
  ),
);
```

```dart
Navigator.of(context).pushNamed('/a');
```


