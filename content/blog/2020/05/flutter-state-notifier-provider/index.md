---
title: "StateNotifierを使ったFlutterのアプリ設計"
date: 2020-05-05T16:17:17+09:00
draft: false
comments: true
images: ["/blog/2020/05/flutter-state-notifier-provider/state_notifier.png"]
---

最近自分の周りでFlutterを始める人が多く、ありがたいことにFlutterに関する質問を個人的にもらうことが増えてきましたが、
特にその中でもアプリ全体の設計をどうするべきかのについてよく聞かれます。

2019年の12月に書いたアドベントカレンダーの中で`Bloc`,`Redux`,`MobX`の3つのアーキテクチャを紹介しましたが、
現在は、それらを使わずにアプリ設計をしています。

>Flutterのアプリ設計(Bloc)
>
>https://itome.team/blog/2019/12/flutter-advent-calendar-day21/

>Flutterのアプリ設計(Redux)
>
>https://itome.team/blog/2019/12/flutter-advent-calendar-day22/

>Flutterのアプリ設計(Mobx)
>
>https://itome.team/blog/2019/12/flutter-advent-calendar-day23/

そこで2020年5月現在、自分が最善だと思うFlutterアプリの実装パターンをまとめておきます。

## TL;DR
- `Collection if/for`や拡張メソッドを使うためにDart2.7~を有効にしておく
- 状態管理にはStateNotifierを使う
- LocatorMixinを使った依存性の注入
- Modelクラスの定義にはfreezedを使う

## StateNotifierについて
`StateNotifier`は[Provider](https://pub.dev/packages/provider)パッケージの作者の方による状態管理のためのパッケージです。

Providerパッケージをベースに作られているので、使ったことがない方は下の記事を読んでProviderパッケージについて理解しておくと
スムーズだと思います。

>FlutterのProviderパッケージを使いこなす
>
>https://itome.team/blog/2019/12/flutter-advent-calendar-day7/

Flutterには値の変更を通知する`ChangeNotifier`クラスと、
`ChangeNotifier`の持てる値を一つだけに限定した`ValueNotifier`クラスがあります。

Providerパッケージには`ChangeNotifierProvider`という、
`ChangeNotifier`の変更を子孫Widgetに通知するProviderがありますが、
`StateNotifierProvider`の基本的な理解はその`ValueNotifier`版です。

カウンターの値を管理するためのクラスを`ChangeNotifier`と`StateNotifier`で書き比べてみると以下のようになります。

```dart
// ChangeNotifierを使った実装
class CounterController extends ChangeNotifier {
  int count = 0;
  
  void increment() {
    count++;
    notifyListeners();
  }
}

// StateNotifierを使った実装
class CounterController extends StateNotifier<int> {
  CounterController(): super(0)

  void increment() {
    state++;
  }
}
```

`ChangeNotifier`ではカウントを`count`変数で管理しているのに対して、
`StateNotifier`では持てる唯一の値である`state`変数にカウントを割り当てています。

今度はカウントに加えて、カウンターのボタンが有効化されているかどうかも管理するようにしてみましょう。

```dart
// ChangeNotifierを使った実装
class CounterController extends ChangeNotifier {
  int count = 0;
  
  bool isEnabled = true;
  
  void increment() {
    count++;
    notifyListeners();
  }
  
  void disableCounter() {
    isEnabled = false;
    notifyListeners();
  }
}

// StateNotifierを使った実装
class CounterState {
  CounterState({
    this.count = 0,
    this.isEnabled = true,
  });
  int count;
  bool isEnabled;
}

class CounterController extends StateNotifier<CounterState> {
  CounterController(): super(CounterState())

  void increment() {
    state = state..count++;
  }

  void disableCounter() {
    state = state..isEnabled = false;
  }
}
```

単なる変数で状態を管理している`ChangeNotifier`は値の変更を通知するために
`notifyListeners()`を明示的に呼ぶ必要がありますが、代わりに変数を増やすことで複数の状態を管理することができます。

一方`StateNotifier`は`state`変数にしか状態を持てないため、
`CounterState`のようなモデルクラスを`state`変数にいれて複数の状態を管理することになります。

上の例では`CounterState`を直接書き換えていますが、`CounterState`はデータを保持するだけのクラスなので、
変更不可(immutable)なクラスにしたいです。

```dart
@immutable
class CounterState {
  CounterState({
    this.count = 0,
    this.isEnabled = true,
  });
  final int count;
  final bool isEnabled;
}

class CounterController extends StateNotifier<CounterState> {
  CounterController(): super(CounterState())

  void increment() {
    state = CounterState(
      count: state.count + 1,
      isEnabled: state.isEnabled,
    );
  }

  void disableCounter() {
    state = CounterState(
      count: state.count,
      isEnabled: false,
    );
  }
}
```

これで`CounterState`はimmutableなクラスになりましたが、
今度は値を変更するたびに`CounterState`を作成しなくてはいけなくなり、
書き換えたい値以外まで毎回指定する冗長なコードになってしまいました。

`freezed`パッケージを使うことでこれを解決することができます。
詳しい説明を後回しにして結論だけ書いてしまうと、以下のように書き換えることができます。

```dart
@freezed
class CounterState {
  CounterState({
    int count,
    bool isEnabled,
  });
}

class CounterController extends StateNotifier<CounterState> {
  CounterController(): super(CounterState(count: 0, isEnabled: true))

  void increment() {
    state = state.copyWith(count: state.count + 1);
  }

  void disableCounter() {
    state = state.copyWith(isEnabled: false);
  }
}
```

かなりすっきり書けるようになりました。

## freezedパッケージについて
こちらも`Provider`パッケージと同じ作者のもので、
コード生成によってimmutableなデータクラスを作成してくれるライブラリです。

以下のようなメソッドを自動生成してくれます。
(実際にはもっと複雑コードが生成されていますが、イメージがわかりやすいように簡略化しています。)

```dart
// generated
CounterState copyWith({
  int count,
  bool isEnabled,
}) {
  return CounterState(
    count: count == null ? this.count ? count,
    count: isEnabled == null ? this.isEnabled ? isEnabled,
  );
}
```

`copyWith`の自動生成以外にも、
- Unionクラス(Kotlinのsealed classやTypescriptのUnion Types相当)が生成できる
- `@required`をつけることでnullを弾くことができる(将来的に導入されるNNBD相当の機能が使える)
- `json_serializable`と連携してJsonのパースができるコードを生成できる
- `==`が生成される
- `toString()`が生成される
- `@late`でフィールドの遅延初期化が生成される

などの便利機能が自動生成されます。今回は`copyWith`以外の機能は使わないので詳しく紹介しませんが、
興味があれば[freezedのドキュメント](https://pub.dev/packages/freezed)に目を通してみてください。

## Widgetでstateを受け取る
`CounterController`の状態(`CounterState`)を受け取るために、
まず`CounterState`を表示したい画面より上位で`StateNotifierProvider`を使って、
`CounterController`を下流に流します。

```dart
StateNotifierProvider<CounterContrller, CounterState>(
  create: (context) => CounterContrller(),
  child: const CounterView(),
)
```

これで、`CounterPage`で`Provider.of<CounterState>(context)`を呼ぶことで`CounterState`を、
`Provider.of<CounterController>(context)`を呼ぶことで`CounterController`を、
それぞれ取得することができるようになります。

```dart
class CoutnerView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          Text(
            Provider.of<CounterState>(context, listen: true).count,
          ),
          IconButton(
            icon: Icon(Icons.add),
            onPressed: () {
              Provider.of<CounterController>(context, listen: false).increment(),
            }
          ),
        ],
      ),
    );
  }
}
```

`Provider.of`の`listen:`引数に`true`を渡すと、対象のクラスに変更があった場合に、
第一引数に渡した`context`の範囲全てが再描画されます。

今回の場合は`CounterState`に変更があった場合に引数に渡した`CounterView`全体が更新されます。

ちなみに、Dart2.7の機能を使えるようにした上で`Provider`の`dev`版を使うと、
以下のように書き換えることができるようになります。

```yaml
// pubspec.yml

// Dartの2.7の機能を使えるようにする
environment:
  sdk: ">=2.7.0 <3.0.0"

...

dependencies:
  ...
  // Providerのdev版を指定する
  provider: 4.1.0-dev+2
```

```dart
// Provider.of<CounterController>(context, listen: false);
context.read<CounterController>();

// Provider.of<CounterState>(context, listen: true).count;
context.watch<CounterState>().count;

// Provider.of<CounterState>(context, listen: true).count;
context.select<CounterState, int>((state) => state.count);
```

`context.select`と`context.watch`はどちらも同じコードの書き換えができますが、
`context.watch`は`CounterState`が変更されたときに毎回再描画が走るのに対して、
`context.select`は`CounterState`の中でも`counter`が変更されたときにだけ再描画されます。
どちらでも使える場合はパフォーマンスに優れた`context.select`の方を積極的に使ったほうがいいです。

## LocatorMixinでDIする
DIという概念にそもそも馴染みがない方もいると思うので、少し遠回りですが順番に説明していきます。

まず、さきほどの`CounterController`クラスに、APIから現在のカウントを取得するような機能をつけたいとします。

APIから値を取得するためコードは`Repository`クラスに隠蔽されていることにして以下のように書くことができます。

```dart
@freezed
class CounterState {
  CounterState({
    int count,
  });
}

class CounterController extends StateNotifier<CounterState> {
  CounterController(): super(CounterState(count: 0))
  
  final repository = CouterRepository()
  
  Future<void> fetchCount() async {
    state = state.copyWith(
      count: await repository.getCount();
    );
  }

  void increment() {
    state = state.copyWith(count: state.count + 1);
  }
}
```

しかし、これでは`CounterRepository`の実装に依存してしまうので、テストのときに通信をモックすることができません。

そこで`CounterRepository`をコンストラクタで受け取るようにしましょう。

```dart
// main.dart
StateNotifierProvider<CounterController, CounterState>(
  create: (context) => CounterController(CounterRepository()),
  child: const CounterView(),
),

// counter_controller.dart
class CounterController extends StateNotifier<CounterState> {
  CounterController(this.repository): super(CounterState(count: 0))
  
  final CouterRepository repository;
  
  ...
}
```

これで、テストのときにモックを外から注入できるようになりました。
Dartは`class`を定義すると暗黙的にその`interface`も定義されるのでテスト用に別で`interface`を
作成する必要はありません。

上のコードでは`CounterController`のインスタンス化のときに`CounterRepository`もインスタンス化していますが、
他の`StateNotifier`でも`CounterRepository`を使いまわしたいことがあると思います。

`CounterRepository`を使い回すために、`StateNotifierProvider`の上流に`Provider`を置いて、そこから
`CounterRepository`を取得するようにします。

```dart
// main.dart
Provider(
  create: () => CounterRepository(),
  child: ...
  
    ...
    
    StateNotifierProvider<CounterController, CounterState>(
      create: (context) => CounterController(context.read<CounterRepository>()),
      child: const CounterView(),
    ),
    
    StateNotifierProvider<AnotherController, AnotherState>(
      create: (context) => AnotherController(context.read<CounterRepository>()),
      child: const AnotherView(),
    ),
)
```

こうすることで、複数の`StateNotifier`で同じ`CounterRepository`を使うことができるようになりました。

さて、ここで`CoutnerContrller`の中で、`UserRepository`という別の`Repository`を使いたくなりました。

```dart
// main.dart
MultiProvider(
  providers: [
    Provider(create: (_) => CounterRepository()),
    Provider(create: (_) => UserRepository()),
  ],
  child: ...
  
    ...
    
    StateNotifierProvider<CounterController, CounterState>(
      value: (context) => CounterController(
        context.read<CounterRepository>(),
        context.read<UserRepository>(),
      ),
      child: const CounterView(),
    ),
)

// counter_controller.dart
class CounterController extends StateNotifier<CounterState> {
  CounterController(
    this.counterRepository,
    this.userRepository,
  ): super(CounterState(count: 0))
  
  final CouterRepository counterRepository;
  
  final UserRepository userRepository;
  
  ...
}
```

これでももちろんきちんと動きますが、依存しているRepositoryが増えるごとに、
コンストラクタがどんどん肥大化していってしまいます。

それを防ぐために、いっそのこと`context`を`CounterContrller`に渡してしまえば、
`CounterController`内部で`context.read`を呼ぶことで
`UserRepository`も`CounterRepository`も取得できるようになります。

```dart
// counter_controller.dart
class CounterController extends StateNotifier<CounterState> {
  CounterController(this.context): super(CounterState(count: 0))
  
  final BuildContext context;
  
  CouterRepository get counterRepository => context.read<CounterRepository>();
  
  UserRepository get userRepository => context.read<UserRepository>();
  
  ...
}
```

これで正常に動きはしますが、`context`は`Scaffold.of(context)`や`Navigator.of(context)`を使って多くのUI要素に
直接アクセスできてしまうので、乱用を避けるためにできるだけWidget以外のクラスで保持するべきではありません。

実際には`context.read`だけが使えれば十分なはずなので、
`context`を渡す代わりに`context.read`関数だけを`CounterContrller`に渡すことにしましょう。

```dart
// main.dart
MultiProvider(
  providers: [
    Provider(create: (_) => CounterRepository()),
    Provider(create: (_) => UserRepository()),
  ],
  child: ...
  
    ...
    
    StateNotifierProvider<CounterController, CounterState>(
      value: (context) => CounterController(context.read),
      child: const CounterView(),
    ),
);

// counter_controller.dart
class CounterController extends StateNotifier<CounterState> {
  CounterController(this.read): super(CounterState(count: 0))
  
  final T Function<T>() read;
  
  CouterRepository get counterRepository => read<CounterRepository>();
  
  UserRepository get userRepository => read<UserRepository>();
  
  ...
}
```

ジェネリクスが入って少しわかりづらくなりましたが、コンストラクタのときに`context.read`関数のみを受け取って、
それを使って`CounterContrller`内部から`Repository`を取得しているだけです。

上のコードで`read`の方として使われている`T Function<T>()`という型は
`Provider`パッケージで`Locator`として定義されているので、
以下のように書き換え可能です。

```dart
class CounterController extends StateNotifier<CounterState> {
  CounterController(this.read): super(CounterState(count: 0))
  
  // final T Function<T>() read;
  final Locator read;
  
  CouterRepository get counterRepository => read<CounterRepository>();
  
  UserRepository get userRepository => read<UserRepository>();
  
  ...
}
```

これで、依存する`~Repository`が増えてもコードをほとんど追加せずに取得できるようになりました。

この`Locator`パターンをさらにシンプルに書けるようにしたのが`LocatorMixin`です。
`LocatorMixin`を使うと、`StateNotifierProvider`と組み合わせて使ったとき限定ですが
`context.read`が自動的にフィールドに追加されます。

そのためコンストラクタに`context.read`を渡すコードもフィールドに`Locator`を定義するコードも省略できます。

```dart
// main.dart
MultiProvider(
  providers: [
    Provider(create: (_) => CounterRepository()),
    Provider(create: (_) => UserRepository()),
  ],
  child: ...
  
    ...
    
    StateNotifierProvider<CounterController, CounterState>(
      value: (context) => CounterController(),
      child: const CounterView(),
    ),
);

// counter_controller.dart
class CounterController extends StateNotifier<CounterState> with LocatorMixin {
  CounterController(): super(CounterState(count: 0))
  
  CouterRepository get counterRepository => read<CounterRepository>();
  
  UserRepository get userRepository => read<UserRepository>();
  
  ...
}
```

`LocatorMixin`を使うことで親以上の`Provider`の値を簡単に取得できるようになりました。
このように、依存するオブジェクトを外部から(暗黙的に)注入することはDI(依存性注入)と呼ばれています。

以下のようにWidgetツリーの構造が直接依存関係グラフになっているところが、
他のプラットフォームにおけるDIと比べて特徴的で面白いです。

```dart
Provider(
  create: (_) => CounterApiClient(),
  child: Provider(
    (context) => CounterRepository(context.read), // 内部でCounterApiClientを使うことができる
    StateNotifierProvider<CounterController, CounterState>(
      value: (context) => CounterController(), // 内部でCounterRepositoryを使うことができる
      child: const CounterView(),
    ),
  ),
)
```

シンプルに書ける以外にも`LocatorMixin`には以下のようなメリットがあります。

- **テスト時にモックを注入しやすい**
```dart
final mockCoutnerRepository = MockCoutnerRepository();
final mockUserRepository = MockUserRepository();

final contrlller = CounterController()
  ..debugMockDependency<CounterRepository>(mockCounterRepository);
  ..debugMockDependency<MockUserRepository>(mockUserRepository);
```

- **`initState`が使える**

`StateNotifier`の初期化時に読み込みをここで行うことができます。

```dart
class CounterController extends StateNotifier<CounterState> with LocatorMixin {
  CounterController(): super(CounterState(count: 0))
  
  @override
  void initState() {
    fetchCount();
  }
  
  Future<void> fetchCount() async {
    state = state.copyWith(
      count: await read<CounterRepository>().getCount();
    );
  }
}
```

- **`update`が使える**

依存している別のオブジェクトの変更監視ができます。(context.watch相当)

```dart
class CounterController extends StateNotifier<CounterState> with LocatorMixin {
  CounterController(): super(CounterState(count: 0))
  
  @override
  void update(Locator watch) {
    state = state.copyWith(
      count: watch<SettingContrller>().defaultCount;
    );
  }
}
```

## まとめ
`provider` `state_notifier` `freezed`という3つのパッケージを使ったFlutterのアプリ設計を紹介しました。
現在は実際の仕事も含めて自分が携わっているFlutterプロジェクトのほとんどはこの構成になっています。

3つのパッケージを開発された[Remi Rousselet](https://twitter.com/remi_rousselet)さんは、
他にも便利なパッケージをいくつも作っていたり、
TwitterでFlutterのTipsを教えてくれたりするので、是非フォローしておくことをおすすめします。

本来は`Controller`クラスで画面遷移やエラーの表示などのイベントを発火させる方法を更に紹介するつもりでしたが、
`LocatorMixin`の説明を詳しくしていたらやたら長い記事になってしまったので別の記事にします。
