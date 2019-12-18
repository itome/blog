---
title: "Flutterのアプリ設計(Redux)"
date: 2019-12-18T16:43:08+09:00
draft: true
comments: true
images:
---


この記事は[Flutter 全部俺 Advent Calendar](https://adventar.org/calendars/4140) 1日目の記事です。


## このアドベントカレンダーについて
このアドベントカレンダーは [@itome](https://twitter.com/itometeam) が全て書いています。

基本的にFlutterの公式ドキュメントとソースコードを参照しながら書いていきます。誤植や編集依頼はTwitterにお願いします。

## Redux

ReduxはもともとReactの状態管理をするためのライブラリで、以下のような3原則があります。

- **Single source of truth**

ソースが複数あるデータは、常に不整合の可能性があります。たとえば、タイムラインでは投稿にいいねがついているのに、
詳細画面ではいいねがついていないようなことが起ってしまいます。
これを開発者の注意で解決するのはとても骨の折れる作業です。

Reduxでは、状態管理を巨大なオブジェクト一つに任せることによってデータの不整合を防いでいます。
巨大なJsonにアプリの全ての状態が書いてあって、それを変更していくことで状態を変更するようなイメージです。

- **State is read only**

状態を直接操作することはできず、`Action`オブジェクトを発行することでしかできないように決められています。

これによって、意図しない変更が起こってしまったり、変更の競合が起こってしまったりすることを防ぐことができます。

- **Changes are made with pure function**

上でJsonを変更していくと書きましたが、状態はread only(読み取り専用)で変更してはいけないというルールがあります。
ではどうやって状態を変更するのかというと、元の状態から変更したい部分だけ変えた状態を新しく作る関数を実行します。

この関数は`(元の状態, 変更したい情報) => 新しい状態`という形式の純粋関数で、
状態を受け取って状態を返す以外は何もしてはいけません。

純粋な関数のみで状態変更を実装することで複雑になりやすい状態変化を見通しやすくしたり、テストが容易になったりします。

ここまでが、Reduxの教科書的な説明です。実際にFlutterとReduxを組み合わせて実装してみましょう。

## ReduxをFlutterで使う
`flutter_redux`というパッケージを使って実装していきます。特に理由がなければよく使われているこのライブラリに
乗っかっておくのが良さそうです。

今回もカウンターアプリを作っていきます。

### Stateを作る
今回は状態が`count`しかないので、`State`に階層構造を持たせずにそのまま`int`型を入れてももんだありませんが、
実用性に乏しいので、`RootState`クラスと`CounterState`クラスを作っておきます。Jsonで表すと以下のようなイメージです。

```json
{
  "counter": {
    "count": 0
  }
}
```

`RootState`クラスと`CounterState`クラスの定義は以下の通りです。
Stateクラスは全てデフォルト引数を設定しておくようにしてください。
こうしておくことで、あとで`RootState`の初期化が簡単になります。

```dart
class CounterState {
  final int count;

  CounterState({this.count = 0});
}

class RootState {
  final CounterState counter;

  RootState({this.counter = CounterState()});
}
```

次に`IncrementAction`クラスを定義します。いくつ数を増やすかの`count`定数だけを持っています。

```dart
class IncrementAction {
  final int count;

  IncrementAction(this.count);
}
```

`IncrementAction`を処理する`counterReducer`を実装します。`action`引数は`dynamic`型で受け取っていますが、
`if (action is IncrementAction) {...}`で型の条件を書くと、カッコ内では自動的にキャストしてくれます。

```dart
CounterState counterReducer(CounterState state, action) {
  if (action is IncrementAction) {
    return CounterState(count: state.count + action.count);
  } else {
    return state;
  }
}
```

最後に`RootState`を処理する`rootReducer`を実装します。この関数は`counterReducer`に処理を丸投げしているだけです。

```dart
RootState rootReducer(RootState state, action) {
  return RootState(
    counter: counterReducer(state.counter, action)
  );
}
```

あとは、ここまで作った`RootState`と`rootReducer`を使って`Store`を作るだけです。

```dart
final store = Store<RootState>(
  rootReducer, 
  initialState: RootState(), 
);
```

ここで作った`store`がアプリ内で唯一状態を持っているインスタンスです。これをWidgetツリー全体から使えるようにするために、
`flutter_redux`パッケージでは、`StoreProvider`が提供されています。

`StoreProvider`の使い方は`Provider`パッケージと同じです。
`Provider`パッケージに関して詳しくは
[6日目の記事](https://itome.team/blog/2019/12/flutter-advent-calendar-day6/)と
[7日目の記事](https://itome.team/blog/2019/12/flutter-advent-calendar-day7/)を読んでください。

アプリ全体から`store`にアクセスしたいので、`MaterialApp`よりも上に`StoreProvider`をおきます。

```dart
class MyApp extends StatelessWidget {
  final store = Store<RootState>(
    rootReducer,
    initialState: RootState(),
  );

  @override
  Widget build(BuildContext context) {
    return StoreProvider(
      store: store,
      child: MaterialApp(
        title: 'Flutter Demo',
        theme: ThemeData(
          primarySwatch: Colors.blue,
        ),
        home: MyHomePage(title: 'Flutter Demo Home Page'),
      ),
    );
  }
}
```

これで準備は完了です。画面からReduxの`store`にアクセスしてみましょう。

`StoreProvider`で用意した`store`にアクセスするためには、`StoreBuilder`か`StoreConnector`を使います。

まず`StoreBuilder`から見ていきましょう。この`Widget`は`builder`関数に`store`から`store`を渡し、`RootState`に
変更があるたびに`builder`関数を実行して再描画を行います。

```dart
            StoreBuilder<RootState>(
              builder: (context, store) {
                return Text(
                  '${store.state.counter.count}',
                  style: Theme.of(context).textTheme.display1,
                );
              },
            ),
```

つまり、`store`に対するあらゆる状態の変更が、`StoreBuilder`を使っている全てのWidgetの再描画を引き起こすということです。
これでは、パフォーマンスに対する影響があるのは明らかです。

この問題を解決するためには、`StoreConnector`というWidgetを使います。

`StoreConnector`は`converter`関数に`store`を渡して、その返り値を`builder`関数に渡します。

例えば今回の場合は、`store`から`counter`だけを取り出す`converter`を渡しています。
オリジナルのReduxを使ったことがある人はselectorの概念に近いです。

```dart
            StoreConnector<RootState, int>(
              converter: (store) => store.state.counter?.count,
              builder: (context, count) {
                return Text(
                  '$count',
                  style: Theme.of(context).textTheme.display1,
                );
              },
            ),
```

さらに`distinct: true`とすることで、`converter`の返り値(今回の場合は`count`)に変更があったときにだけ
Widgetの再描画を行うようになります。これでさらに無駄なWidgetの描画を減らすことができます。

```dart
            StoreConnector<RootState, int>(
              distinct: true,
              converter: (store) => store.state.counter?.count,
              builder: ...
            ),
```

基本的に`StoreBuilder`よりもこちらを使うようにしましょう。

`store`の状態を変更したい場合は、`store.dispatch`関数を使いましょう。`StoreConnector`には、
`store`を受け取って関数を返す関数を渡します。

```dart
      floatingActionButton: StoreConnector<RootState, VoidCallback>(
        converter: (store) => () => store.dispatch(IncrementAction(1)),
        builder: (context, dispatchIncrement) {
          return FloatingActionButton(
            onPressed: dispatchIncrement,
            tooltip: 'Increment',
            child: Icon(Icons.add),
          );
        },
      ),
```

これで、ボタンを押すとカウントの表示が変更されるようになりました。

## テストを書く

Reduxの特徴の一つにテストが簡単にかけることが挙げられます。今回は状態が少ないのでテストが必要な部分が少ないですが、
`counterReducer`のテストを書いてみましょう。

```dart
void main() {
  test('Counter value should be incremented', () {
    final currentState = CounterState();
    expect(nextState.count, 0);
    
    final action = IncrementAction(1);
    final nextState = counterReducer(currentState, action);

    expect(nextState.count, 1);
  });
}
```

現在の状態と`IncrementAction(1)`を`counterReducer`に渡すと、返り値の新しい状態で`count`が増加しているかを
テストするコードです。
`counterReducer`は純粋関数なので、このテストコードだけで動作を保証することができます。

<br>

> **21日目: Flutterのアプリ設計(Bloc)** :
>
> https://itome.team/blog/2019/12/flutter-advent-calendar-day21
>
> **23日目: Flutterのアプリ設計(MobX)** :

> https://itome.team/blog/2019/12/flutter-advent-calendar-day23
