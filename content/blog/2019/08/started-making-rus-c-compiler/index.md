---
title: "Rust製のC言語コンパイラを作り始めました"
date: 2019-08-11T01:59:46+09:00
draft: false
---

## モチベーション
ずっと自分でプログラミング言語を作りたいと思っていたものの、他のやりたいことに押されて優先度下がっていって結局これまで手を付けられていなかったのですが、知り合いに誘われたのをきっかけに勉強を始めました。
勧められるままに[https://www.sigbus.info/compilerbook](低レイヤを知りたい人のためのCコンパイラ作成入門)をやっているのですが、イチから説明してくれているので、今の所スラスラと進められています。

## Rustを選んだ理由
もとのサイトはC言語でC言語のコンパイラを実装するサンプルコードで進んでいるのですが、せっかくなので別の言語でやってみたいと思い、最近ハマっているRustで実装しています。
Rustは最近ようやくちゃんと触り始めたのですが、かなりいい言語だと思います。
具体的にRustの好きなところは書き始めたところで長くなりそうだったので別のブログに分けます。

## 進捗
とりあえずカッコ付き四則演算ができるようになりました。
```text
try 0 0
try 42 42
try 90 '(12 + 3) * 6'
try 8 '4 + 8 / 2'
try 47 '5+6*7'
try 15 '5*(9-6)'
try 4 '(3+5)/2'
try 5 '+5'
try 5 '12+(-7)'
try 5 '20+(-3*5)'
```

まだ四則演算のみなので、プログラミング言語というよりも電卓です。

- 文字列を受け取って各要素(トークン)に分解 
- 分解したトークンを再帰下降構文解析で木構造に変換
- 木構造を探索しながらアセンブリを標準出力
- 標準出力をシェルでテキストファイルにパイプ
- 作ったアセンブリを `gcc` コマンドでコンパイル

ということをやっています。
再帰下降構文解析は初めてやったのですが、BNFによる生成規則と実装がきれいに結びついていて感動しました。

#### 足し算と引き算
```text
expr  = mul ("+" mul | "-" mul)*
```
```rust
fn expr(tokens: &mut Vec<Token>) -> Self {
    let mut node = Node::mul(tokens);

    loop {
        if tokens.len() == 0 {
            break;
        }
        let token = &tokens[0];
        match token.operator {
            Some('+') => {
                tokens.remove(0);
                let rhs = Node::mul(tokens);
                node = Node::operator('+', node, rhs);
            }
            Some('-') => {
                tokens.remove(0);
                let rhs = Node::mul(tokens);
                node = Node::operator('-', node, rhs);
            }
            _ => {
                break;
            }
        }
    }
    return node;
}
```

#### 掛け算と割り算
```text
mul   = unary ("*" unary | "/" unary)*
```
```rust
fn mul(tokens: &mut Vec<Token>) -> Self {
    let mut node = Node::unary(tokens);

    loop {
        if tokens.len() == 0 {
            break;
        }
        let token = &tokens[0];
        match token.operator {
            Some('*') => {
                tokens.remove(0);
                let rhs = Node::unary(tokens);
                node = Node::operator('*', node, rhs);
            }
            Some('/') => {
                tokens.remove(0);
                let rhs = Node::unary(tokens);
                node = Node::operator('/', node, rhs);
            }
            _ => {
                break;
            }
        }
    }
    return node;
}
```

#### 単項演算子
```text
unary = ("+" | "-")? term
```
```rust
fn unary(tokens: &mut Vec<Token>) -> Self {
    let token = &tokens[0];
    match token.operator {
        Some('+') => {
            tokens.remove(0);
            return Node::term(tokens);
        }
        Some('-') => {
            tokens.remove(0);
            return Node::operator('-', Node::number(0), Node::term(tokens));
        }
        _ => {
            return Node::term(tokens);
        }
    }
}
```

#### 最小単位の各項(各項がカッコに囲まれた式であった場合は、再帰的に評価する)
```text
term  = num | "(" expr ")"
```
```rust
fn term(tokens: &mut Vec<Token>) -> Self {
    if tokens[0].operator == Some('(') {
        let close_index = tokens
            .iter()
            .position(|token| token.operator == Some(')'))
            .unwrap();
        let mut exp = tokens[1..close_index].to_vec();
        tokens.drain(0..(close_index + 1));
        return Node::expr(&mut exp);
    } else {
        let num = tokens[0].value.unwrap();
        tokens.remove(0);
        return Node::number(num);
    };
}
```

もとのサイトに下のようなことが書かれているのですが、

>
> 自作のコンパイラが作者である自分を超える知性を持っているように感じることすらあります。
>

再帰の階層が人間に追いきれないようになるくらいに深くなったあたりから、少しだけこのことを感じるようになりました。

とりあえずの目標としてフィボナッチ数列が再帰で解けるようになるところまでやっていきたいです。
作っているコンパイラのリポジトリは[https://github.com/itome/nine-cc](こちら)

