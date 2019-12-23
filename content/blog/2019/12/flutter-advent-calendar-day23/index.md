---
title: "Flutterのアプリ設計(Mobx)"
date: 2019-12-23T00:00:00+09:00
draft: false
comments: true
images: ["/blog/2019/12/flutter-advent-calendar-day23/flutter_with_mobx.png"]
---

この記事は[Flutter 全部俺 Advent Calendar](https://adventar.org/calendars/4140) 1日目の記事です。


## このアドベントカレンダーについて
このアドベントカレンダーは [@itome](https://twitter.com/itometeam) が全て書いています。

基本的にFlutterの公式ドキュメントとソースコードを参照しながら書いていきます。誤植や編集依頼はTwitterにお願いします。

## MobXとは
Flutterのアプリ設計最後の今日はMobXの紹介です。
MobXはReduxと同じくJavascriptのために考えられた設計で、主にReactと共に使われます。
MobXの公式リポジトリで`mobx.dart`というDart用のMobXパッケージと
`flutter_mobx`というFlutterと連携させるためのパッケージが用意されているので、
それをインストールして使います。

![MobX](./mobx.png)

上の図をみてわかるように、MobXは大きく3つの要素からできています。

### `Observables`
アプリの状態を持つ変数です。mobx.dartでは、単なるクラスの変数として定義したうえで、`@observable`アノテーションをつけます。
`@observable`をつけることで、変数の値が変更されたことを検知することができるようになります。

### `Actions`
アプリの状態を変更する関数です。`Observables`の値の変更はすべてこの関数の中で行われなければいけません。
`mobx.dart`では、通常のクラスメソッドに`@action`というアノテーションをつけて定義します。

### `Reactions`
`Observables`の変更を検知して、実際にアプリに副作用を起こす(画面を再描画したり画面遷移したりする)部品のことです。
`mobx.dart`に`reaction` `autorun` `when`など、場合に応じて使い分けることのできる関数が用意されています。
また、`flutter_mobx`パッケージには、画面を再描画するための`Reactions`として、`Observer`Widgetが用意されています。

## 実際にコードを書いてみる
MobXは、パッケージ側でコード生成などをして面倒な部分を隠蔽してくれているので、実際に書くコードはすごくシンプルになります。
そのため、中身の実装を理解しようとする前に実際にコードを書いてみた方が取り掛かりやすいと思います。

今回もカウンターアプリを作っていきます。

### `flutter_mobx`と`mobx_codegen`をインストールする
プロジェクトルートの`pubspec.yaml`ファイルに以下のように追記してから`$ flutter pub get`を実行します。

```
dependencies:
  flutter_mobx: ^0.3.3+3

dev_dependencies:
  build_runner: ^1.7.2
  mobx_codegen: ^0.3.10+1
```

`mobx`パッケージ自体は自動的に追加されます。`build_runner`と`mobx_codegen`パッケージは、
コード生成のためのパッケージで開発時にしかつかわない(アプリに同梱する必要がない)ので
`dev_dependencies`にしています。

### `CounterStore`クラスを作る
MobXでは、`Observables` `Actions`をまとめて状態管理のための一つのクラスにすることが多いです。
今回は`CounterStore`クラスを作ります。

`counter.dart`という名前のファイルを新たに作って以下のように編集してください。
ファイル名を別の名前にするとコード生成のときに別名のファイルが出力されてサンプルコードと合わなくなるので注意してください。

初めは以下のようにエラーが出ると思います。以下のコマンドを実行することで、
`counter.g.dart`というファイルが同じディレクトリに自動生成されると共にエラーが消えます。

```
$ flutter pub run build_runner build
```

```dart
import 'package:mobx/mobx.dart';

part 'counter.g.dart';

class CounterStore = CounterStoreBase with _$CounterStore;

abstract class CounterStoreBase with Store {
  @observable
  int count = 0;

  @action
  void increment() {
    count++;
  }
}
```

クラスが`CounterBase`クラスになっていたり、`part 'counter.g.dart'`や
`class CounterStore = CounterStoreBase with _$CounterStore;`という謎の行が追加されていたり
色々気になるところはありますが、一旦無視してクラスの中だけ見てみましょう。

```dart
abstract class CounterStoreBase with Store {
  @observable
  int count = 0;

  @action
  void increment() {
    count++;
  }
}
```

ここだけみると、かなりシンプルです。状態である`count`変数には`@observable`、それを変更する関数の`increment`には
`@action`アノテーションがそれぞれついていますが、それ以外は説明不要なシンプルな実装です。

### `CounterStore`をWidgetから使う
まず、Widget側のクラスで`CounterStore`を初期化します。

```dart
class _MyHomePageState extends State<MyHomePage> {
  final store = CounterStore();

  @override
  Widget build(BuildContext context) {
  ...
```

次に、表示したい変数をそのまま使います。

```dart
            Text(
              '${store.count}',
              style: Theme.of(context).textTheme.display1,
            ),
```

しかし、これだけでは`store.count`が変更されたときに合わせて`Text`が再描画されません。

そこで、`Text`Widgetを`flutter_mobx`パッケージの`Observer`Widgetで囲みます。

```dart
            Observer(
              builder: (context) {
                return Text(
                  '${store.count}',
                  style: Theme.of(context).textTheme.display1,
                );
              },
            ),
```

`Observer`クラスで囲むと、内部使われているで`@observable`がつけられた変数(ここでは`store.count`)が
変更されたときに、自動的にWidgetの再描画が行われます。

最後に、`CounterStore`の`increment`関数をボタンに紐付けます。

```dart
      floatingActionButton: FloatingActionButton(
        onPressed: store.increment,
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ),
```

これで、カウンターアプリの完成です。

## MobXの他の機能

サンプルアプリでは最低限の構成で実装しましたが、MobXには他にもいくつか便利な機能があります。

### `@computed`

```dart
abstract class CounterStoreBase with Store {
  @observable
  int count = 0;
  
  @computed
  int get doubledCount => count * 2;

  @action
  void increment() {
    count++;
  }
}
```

`@observable`な値から計算して得られる値を`computed value`と呼びます。
通常のDartのgetterに`@computed`アノテーションをつけることで、
計算に使っているオリジナルの`@observable`変数が変更されたときに自動的に自身の変更も通知します。

### `reaction` `autorun` `when`

サンプルでは`Observer`で値の変更を監視しましたが、それ以外の方法で値の変更検知をしたい場合もあります。
そんなケースの時に使える関数が3つ用意されています。

**`reaction`**

第一引数で返されたobservableな変数が変更されたときに、第二引数の関数を実行します。
値が変更されたときに、前の値と同じ値がセットされた場合は無視されるため、同じ値で何度も実行されることはありません。

```dart
reaction((_) => store.count, (count) => print(count));
```

**`autorun`**

渡した関数内で使われているobservableな変数のうちいづれかが変更されたときに再実行されます。
セットされた値が変わっていなくても再実行されます。

```dart
autorun((_) => print(store.count));
```

**`when`**

第一引数に渡した関数が`true`を返す時のみ第二引数の関数を実行します。第二引数は条件に合致した最初の一回しか
実行されません。つまり、下の例では`store.count`が`4`→`5`→`6`→`5`と変わっても、`'Count reach to 5'`が
表示されるのは一回だけです。

```dart
when((_) => store.count == 5, () => print('Count reach to 5'));
```

ここでは省きましたが、`react` `autorun` `when`はすべて`dispose`関数を返します。
これは、値の監視をやめるための関数なので、`StatefulWidget`の`dispose`メソッドで実行してメモリリークを防ぎましょう。

```dart
class _MyHomePageState extends State<MyHomePage> {
  final store = CounterStore();
  ReactionDisposer dispose;

  @override
  void initState() {
    super.initState();
    dispose = autorun((_) => print(store.count));
  }

  @override
  void dispose() {
    super.dispose();
    dispose();
  }
```

`dispose`が複数ある場合は、リストで管理する方が便利です。

```dart
class _MyHomePageState extends State<MyHomePage> {
  final store = CounterStore();
  final disposes = <ReactionDisposer>[];

  @override
  void initState() {
    super.initState();
    disposes.add(autorun((_) => print(store.count)));
  }

  @override
  void dispose() {
    super.dispose();
    for (final dispose in disposes) {
      dispose?.call()
    }
  }
```

### `async action`

非同期処理をする場合は`Future<T>`を返す`async`関数に`@action`アノテーションをつけるだけです。

```dart
@action
Future<void> incrementDelayed() async {
  await delayed(Duration(seconds: 1));
  counter++;
}
```

## テストを書く
`CounterStore`は単純なクラスなので、テストも容易です。

```dart
void main() {
  test('increment', () {
    final store = CounterStore();
    expect(store.count, 0);
    store.increment();
    expect(store.count, 1);
  });
}
```

<br>

> **22日目: Flutterのアプリ設計(Redux)** :
>
> https://itome.team/blog/2019/12/flutter-advent-calendar-day22
>
> **24日目: Flutterの自作パッケージを作る** :

> https://itome.team/blog/2019/12/flutter-advent-calendar-day24
