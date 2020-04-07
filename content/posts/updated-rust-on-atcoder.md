---
title: "2020/4 言語アップデート後のAtcoderでのRust環境"
date: 2020-04-06T02:03:52+09:00
draft: true
---

2020/4/5にAtcoderで
[Judge system update test contest](https://atcoder.jp/contests/judge-update-202004)が開催され、
そろそろAtcoderの実行環境がアップデートされそうになっています。
この記事では、新しいRust実行環境で利用可能になったcrateとその使い道を考えてみます。

# 実行環境

ここの[スプレッドシート](https://docs.google.com/spreadsheets/d/1PmsqufkF3wjKN6g1L0STS80yP4a6u-VdGiEv5uOHe0M/edit#gid=1059691052)
に記載があります。利用できるバージョンは1.42 (特に記載がないのでstableでしょう）で,
`Cargo.toml`に次のような依存クレートが書いてあると思っていいでしょう。

```toml
[dependencies]
num = "=0.2.1"
num-bigint = "=0.2.6"
num-complex = "=0.2.4"
num-integer = "=0.1.42"
num-iter = "=0.1.40"
num-rational = "=0.2.4"
num-traits = "=0.2.11"
num-derive = "=0.3.0"
ndarray = "=0.13.0"
nalgebra = "=0.20.0"
alga = "=0.9.3"
libm = "=0.2.1"
rand = { version = "=0.7.3", features = ["small_rng"] }
getrandom = "=0.1.14"
rand_chacha = "=0.2.2"
rand_core = "=0.5.1"
rand_hc = "=0.2.0"
rand_pcg = "=0.2.1"
rand_distr = "=0.2.2"
petgraph = "=0.5.0"
indexmap = "=1.3.2"
regex = "=1.3.6"
lazy_static = "=1.4.0"
ordered-float = "=1.0.2"
ascii = "=1.0.0"
permutohedron = "=0.2.4"
superslice = "=1.0.0"
itertools = "=0.9.0"
itertools-num = "=0.1.3"
maplit = "=1.0.2"
either = "=1.5.3"
im-rc = "=14.3.0"
fixedbitset = "=0.2.0"
bitset-fixed = "=0.1.0",
proconio = { version = "=0.3.6", features = ["derive"] }
text_io = "=0.1.8"
whiteread = "=0.5.0"
rustc-hash = "=1.1.0"
smallvec = "=1.2.0"
```

他の依存関係し子孫になっていて直接触れることのないcrateも含まれていると思いますので、
次のcrateについてそれぞれ使い道を考えていきます。

- [num](https://rust-num.github.io/num/num/index.html)
- 

# num
https://rust-num.github.io/num/num/index.html

num crateには数に関する便利な構造体や関数、primativeな数値型の満たすべきtraitなどが実装されています。
競技プログラミングで使う可能性があるのは以下のようなものでしょうか。
- `num::BigInt`
- `num::Complex`
- `num::integer::{gcd, binomial}`

`BigInt` (多倍長整数)はatcoderではなかなか使うことはないような印象がありますが使い方を覚えておいて損はないでしょう。
また `Complex` は二次元幾何の問題で使うと実装が楽になることが多いのでこれも覚えておくと良さそうです。
いずれも基本的な演算は実装されています。
`binomial`は大抵modと組み合わせて使うことが多いのでこれを直接使うことは多くはなさそうですが、簡単な問題では使いどころがありそうです。
`gcd`も自分で実装しても大した実装量ではないですが、任意の整数型で使えるので覚えておくと少し楽ができるかもしれません。

```rust
use num::BigInt;
use num::Complex;
use num::integer::{binomial, gcd};
use std::f64::consts::PI;

fn main() {
    let mut big_int: BigInt = 1.into();
    for n in 1..100 {
        big_int *= n;
    }
    println!("{}", big_int);
    //933262154439441526816992388562667004907159682643816214685929638952175999932299156089414639761565182862536979208272237582511852109168640000000000000000000000 

    let mut x = Complex::new(1.0, 1.0);
    let mut y = Complex::new(-1.0, 2.0);

    println!("{}", x + y); // 0+3i
    println!("{}", x * y); // -3+1i
    println!("{}", x.norm());
    println!("{}", y * Complex::new(0.0, 1.5*PI).exp()); // rotate 1.5 PI

    println!("{}", binomial(20u64, 8));
    println!("{}", gcd(276943533807i64 115i64));
}
```

# rand

乱数生成器の実装です。一番よく使うのは一様分布からのサンプリングでしょうか。
`(0, 1]` からの`f64`のサンプリングは簡単です。

```rust
use rand::random;

fn main() {
    for _ in 0..10 {
        println!("{}", random::<f64>());
    }
}
```

整数の区間`[a, b)`からのサンプリングはもちろんライブラリを使ってもいいですが、`f64`の値から計算する方が楽かもしれないです。

ライブラリ
```rust
use rand::{Rng, SeedableRng};
use rand::distributions::{Distribution, Uniform};

fn main() {
    let between = Uniform::from(10..1000);
    let mut rng = rand::rngs::SmallRng::from_entropy();
    for _ in 0..10 {
        println!("{}", between.sample(&mut rng));
    }
}
```

上の例で出てくる`SmallRng`はデフォルトで使われる乱数生成アルゴリズムより高速なため、
速度的に不十分な場合は検討しても良いかもしれません。
（ただし、atcoderではそんな状況はほぼないと思います）

