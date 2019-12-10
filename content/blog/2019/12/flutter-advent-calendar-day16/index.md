---
title: "Flutterのアニメーションを理解する(後編)"
date: 2019-12-10T17:05:31+09:00
draft: true
comments: true
images:
---

## 応用的なアニメーションを作る手順
Flutterのカスタムアニメーションを作るときは、

- `AnimationController`の用意
- 各種`Tween`の割り当て
- `Animation`のWidgetへの割り当て

の順番で考えていきます。

### `AnimationController`の用意
アニメーションを扱うWidgetは`TickerProviderStateMixin`などを使いやすいように、
`StatefulWidget`にしておくのがいいです。

まずは、`initState`で`ActionController`を用意しておきましょう。

```dart
AnimationController _controller;

@override
void initState() {
  super.initState();
  _controller = AnimationController(
    duration: const Duration(milliseconds: 300),
    vsync: this,
  );
}
```

### 各種`Tween`の割り当て

作った`ActionCreator`に`Tween`を割り当てて`0.0`~`1.0`の`double`値を
必要な値にマッピングした`Animation`を作ります。
`AnimationController`の`drive`関数に`Tween`を渡すか、
`Tween`の`animate`関数に`AnimationController`を渡すことで、`Animation`が取得できます。

複数の`Tween`をつなげることもできます。例えば`TweenA`と`TweenB`をつなげると

```
元の値(0.0~1.0) → TweenAで変換した値 →TweenBで変換した値
```

のようになります。つまり、以下のコードは全て同じ挙動をします。ケースに分けて使い分けましょう。

```dart
...
Animation<Color> _animation;

@override
void initState() {
  ...
  final curvedAnimation = CurveTween(curve: Curves.bounceIn).animate(_controller);
  _animation = ColorTween(begin: Colors.blue, end: Colors.red,).animate(_controller);
}
```

```dart
...
Animation<Color> _animation;

@override
void initState() {
  ...
  _animation = _controller
      .drive(CurveTween(curve: Curves.bounceIn))
      .drive(ColorTween(begin: Colors.blue, end: Colors.red));
}
```

```dart
...
Animation<Color> _animation;

@override
void initState() {
  ...
  _animation = ColorTween(begin: Colors.blue, end: Colors.red)
      .chain(CurveTween(curve: Curves.bounceIn))
      .animate(_controller);
}
```

### `Animation`のWidgetへの割り当て
`Animation`の現在の値が`_animation.value`で取得できるので、Widgetのパラメーターとして使います。

```dart
@override
Widget build(BuildContext context) {
  return Container(color: _animation.value)
}
```

しかし、これでは`_animation`の値が変化してもWidgetの色を変えることができないので、
`AnimatedBuilder`を使います。


```dart
@override
Widget build(BuildContext context) {
  return AnimatedBuilder(
    animation: _animation,
    builder: (context, _) {
      return Container(color: _animation.value)
    },
  );
}
```

`AnimatedBuilder`は`animation: `に渡した`Animation`の値が変わると
`builder`で作られるWidgetが再描画されるWidgetです。
これを使うことで`_animation`の値に合わせてWidgetの色が変わるようになります。

また、`AnimatedWidget`を継承してWidgetをつくることでも、値の変化に対応することが可能です。
`AnimatedWidget`の場合も仕組みは同じで、`super(listenable: )`に渡した`Animation`が更新される
たびに`build`関数が再実行されます。

```dart
class ColorTransition extends AnimatedWidget {
  const ColorTransition({Animation color}): super(listenable: color);

  Animation<Color> get _color => listenable;

  @override
  Widget build(BuildContext context) {
    return Container(color: _color.value);
  }
}
```

## `Animated`系Widgetと`Transition`系Widget
ここまではアニメーションをすべて自前で実装する手順を紹介していましたが、
デフォルトで用意されているWidgetを使うことで、もっと手軽にアニメーションを組むことができるようになります。

Flutterには`AnimatedContainer`や`AnimatedTheme`など、`Animated~`という名前のWidgetと
`SlideTransition`や`FadeTransition`などの`~Transition`という名前のWidgetがあります。

これらはどちらもアニメーションを行うWidgetで、`AnimatedPositioned`と`PositionedTransition`のように
同じアニメーションをできるWidgetがそれぞれ用意されていたりするのでややこしいですが、
両者の違いは自前で`AnimationController`を持っているかどうかです。

`Animated`系Widgetは自前で`AnimationController`を持っているので、こうなってほしいという状態を渡すだけで
勝手にそこに向かって動いてくれます。

```dart
int _padding = 0;

@override
Widget build(BuildContext context) {

  return Stack(
    children: <Widget>[
      AnimatedPositioned(
        top: 0,
        left: _padding,
        width: 100,
        height: 100,
        duration: const Duration(milliseconds: 500),
        curve: Curves.easeInOut,
        child: ...,
      ),
    ],
  );
}
```

上のコードでは`_padding`の値を`setState(() => { _padding = ... })`で変えるだけで、
500ミリ秒かけてアニメーションをしてくれます。

一方`Transition`系Widgetは自前では`AnimationController`を持っていないので、そとから渡してあげる必要があります。

```dart
@override
Widget build(BuildContext context) {
  return Stack(
    children: <Widget>[
      PositionedTransition(
        rect: _animationController.drive(
            RelativeRectTween(
              begin: RelativeRect.fromLTRB(0, 0, 100, 100),
              end: RelativeRect.fromLTRB(200, 0, 100, 100),
            ),
          ),
        child: ...,
      )
    ],
  );
}
```

`Animated`系Widgetはこちらで`AnimationController`を用意しなくていい分、より手軽に使うことができます。
一方こちらから`AnimationController`を渡せる`Transition`は、アニメーションを途中で止めたり逆再生したりと
`Animated`系Widgetではできなかったような柔軟な操作もできます。

目的のアニメーションが`Animated`系にも`Transition`系にも用意されている場合は、まず`Animated`系Widgetから検討してみて、
それでもうまくアニメーションが作れなかったときに、`Transition`系Widgetを試してみるのがいいと思います。

どのような`Animated`系Widgetや`Transition`系Widgetが用意されているかは、
[mono](https://twitter.com/_mono)さんの以下の記事を参考にしてください。

> **Flutterのお手軽にアニメーションを扱えるAnimated系Widgetをすべて紹介**
>
> https://link.medium.com/7471nO5Mi2
>
> **FlutterのTransition系アニメーションWidgetをすべて紹介**
>
> https://link.medium.com/qwcmc0aNi2

<br/>

> **15日目: Flutterのアニメーションを理解する(前編)** :
>
> https://itome.team/blog/2019/12/flutter-advent-calendar-day15
>
> **17日目: FlutterのAnimatedWidgetを使いこなす** :
>
> https://itome.team/blog/2019/12/flutter-advent-calendar-day17
