---
title: "MixinとStatic Extension Methodを使ってAutoDispose"
date: 2019-12-27T11:57:54+09:00
draft: false
comments: true
images: ["/blog/2019/12/flutter-auto-dispose/eyecatch.png"]
---

Flutterアプリを作っていると、以下のようなコードをよく書くと思います。

```dart
class SampleWidgetSate extends State<SampleWidget> {
  AnimationController controller;
  
  ...
  
  @override
  void initState() {
    super.initState();
    controller = AnimationController(...);
  }
  
  @override
  void dispose() {
    controller.dispose();
    super.dispose();
  }
}
```

`AnimationController`、`ScrollController`、`InputController`などは`ChangeNotifier`
を継承しているので、Widgetの`dispose`のタイミングで`dipose`を呼ぶ必要があります。

ただ、管理すべき`~Controller`の数が増えると、
これらを複数行うのは面倒だしボイラープレートの冗長なコードが増えてしまいます。

そこで、MixinとDart2.7で追加された`Static Extension Function`でこれを簡単に書けるようにしました。

以下のようなMixInとExtensionを書きます。

```dart
import 'package:flutter/widgets.dart';

mixin AutoDisposeStateMixin<T extends StatefulWidget> on State<T> {
  final _changeNotifiers = <ChangeNotifier>[];

  void addDisposer(ReactionDisposer dispose) {
    _disposers.add(dispose);
  }

  @override
  void dispose() {
    super.dispose();
    for (final notifier in _changeNotifiers) {
      notifier?.dispose();
    }
  }
}

extension AutoDisposeChangeNotifier on ChangeNotifier {
  void disposedBy(AutoDisposeStateMixin state) {
    state.addChangeNotifiers(this);
  }
}
```

これをインポートすると、以下のように書くことができるようになります。

```dart
class SampleWidgetSate extends State<SampleWidget> with AutoDisposeStateMixin {
  AnimationController controller;
  
  ...
  
  @override
  void initState() {
    super.initState();
    controller = AnimationController(...).disposedBy(this);
  }
}
```

結構短く書けるようになりました。`Controller`が複数あるとさらに差がわかりやすいと思います。

ExtensionはDart2.7以降の機能なので、まだどのプロジェクトでも使えるわけではないですが、
Dartの可能性を大きく広げる機能だと思います。

ちなみに、今回の`AutoDispose`は`RxSwift`のAPIに似せて作りました。

すでに拡張関数の利用例が豊富なKotlinやSwiftなどの先輩言語から
拡張関数の利用ケースを輸入するだけでも、コードをよりシンプルにできると思います。
