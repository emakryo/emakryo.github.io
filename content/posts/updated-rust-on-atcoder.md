---
title: "2020/04言語アップデート後のAtcoder Rust環境で使える便利Crates"
date: 2020-04-13T00:00:00+09:00
---

久しぶりにブログというものを書きます。
ここ半年ぐらい競プロを再開したのとRustを使い始めたので、C++から乗り換えようかと思って調べてみた内容をまとめました。

-----

2020/4/12の[Atcoder Beginer Contest 162](https://atcoder.jp/contests/abc162)で
Atcoderの実行環境がアップデートされました。
この記事では、新しいRust実行環境で利用可能になったcrateとその使い道を考えてみます。
筆者は普段、競技プログラミングではC++を使っているのでそれに比べてどうかという観点も含まれています。

読者の対象はRust以外を使っている競技プログラマを想定しているので個々のアルゴリズムやデータ構造の説明は省きます。
また、Rustの入門記事という訳でもないのである程度Rust自体のことも知っている想定です。

もしRustは全くわからないが始めてみよう、というのであればまず次のものを読むといいかと思います。
- [Rust book](https://doc.rust-jp.rs/book/second-edition/ch03-00-common-programming-concepts.html)
- [Rustで精選過去問10選](https://qiita.com/tubo28/items/e6076e9040da57368845)

# 実行環境

ここの[スプレッドシート](https://docs.google.com/spreadsheets/d/1PmsqufkF3wjKN6g1L0STS80yP4a6u-VdGiEv5uOHe0M/edit#gid=1059691052)
に記載があります。利用できるバージョンは1.42 (stable）で,
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

ABCで利用用途のありそうな、次のcrateについてそれぞれ使い道を考えていきます。

- [num](https://rust-num.github.io/num/num/index.html)
- [rand](https://docs.rs/rand/0.7.3/rand/)
- [petgraph](https://docs.rs/petgraph/0.5.0/petgraph/index.html)
- [permutohedron](https://docs.rs/permutohedron/0.2.4/permutohedron/)
- [superslice](https://docs.rs/superslice/1.0.0/superslice/)
- [itertools](https://docs.rs/itertools/0.9.0/itertools/)
- [proconio](https://docs.rs/proconio/0.4.1/proconio/)


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
https://docs.rs/rand/0.7.3/rand/

乱数生成器の実装です。一番よく使うのは一様分布からのサンプリングでしょうか。
`[0, 1)` からの`f64`のサンプリングは簡単です。

```rust
use rand::random;

fn main() {
    for _ in 0..10 {
        println!("{}", random::<f64>());
    }
}
```

整数区間`[a, b)`からの一様分布のサンプリングはもちろんライブラリを使ってもいいですが、`f64`の値から計算する方が楽かもしれないです。
ライブラリを使った例は下のようになります。

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
（ABCではそんな状況はほぼないと思いますが）


# petgraph
https://docs.rs/petgraph/0.5.0/petgraph/

グラフとグラフアルゴリズムが実装されています。
普通は自作ライブラリや毎回手書きするようなアルゴリズムもあるので慣れれば便利かもしれません。
競技プログラミングでよく使うのは次のようなものでしょうか。

- `petgraph::graph::{Graph, UnGraph, DiGraph}` (隣接リスト、有向、無向)
- `petgraph::matrix_graph::MatirxGraph` (隣接行列)
- `petgraph::algo::connected_components`
- `petgraph::algo::tarjan_scc` (強連結成分分解)
- `petgraph::algo::dijkstra`
- `petgraph::algo::bellman_ford`
- `petgraph::algo::toposort`
- `petgraph::algo::min_spanning_tree` (Kruskal's algorithm)
- `petgraph::unionfind::UnionFind`

最小全域木を例にして書いてみます。

```rust
use petgraph::graph::UnGraph;
use petgraph::algo::min_spanning_tree;

fn main() {
    let g: UnGraph<u32, u32> = UnGraph::new_undirected();
    // Example
    // 0 - 1 - 2 - 3 - 4
    //      \ /   /
    //       6 - 5 
    g.extend_with_edges(&[
        (0, 1), (1, 2), (2, 3), (3, 4), (3, 5),
        (1, 6), (2, 6), (5, 6),
    ]);

    let min_span = min_spanning_tree(&g);
    for x in min_span {
        println!("{:?}", x);
    }

    // Node { weight: 0 }
    // Node { weight: 0 }
    // Node { weight: 0 }
    // Node { weight: 0 }
    // Node { weight: 0 }
    // Node { weight: 0 }
    // Node { weight: 0 }
    // Edge { source: 0, target: 1, weight: 0 }
    // Edge { source: 2, target: 3, weight: 0 }
    // Edge { source: 2, target: 6, weight: 0 }
    // Edge { source: 1, target: 6, weight: 0 }
    // Edge { source: 5, target: 6, weight: 0 }
    // Edge { source: 3, target: 4, weight: 0 }

    // Resulting graph
    // 0 - 1 - 2 - 3 - 4
    //      \
    //       6 - 5

}
```

# permutohidron

順列生成のアリゴリズムの実装です。次のように使うことができます。
[ヒープアルゴリズム](https://ja.wikipedia.org/wiki/Heap%E3%81%AE%E3%82%A2%E3%83%AB%E3%82%B4%E3%83%AA%E3%82%BA%E3%83%A0)を使うので高速に動作します。
後述しますが、機能的には上位互換のものが`itertools`にあるので、速度的な懸念がなければそちらを使った方がいいのかもしれません。

```rust
use permutohedron::heap_recursive;

fn main() {
    let mut x = vec![0, 1, 2, 3];
 
     heap_recursive(&mut x, |p| {
         println!("{:?}", p);
     });
 
// [0, 1, 2, 3]
// [1, 0, 2, 3]
// [2, 0, 1, 3]
// [0, 2, 1, 3]
// [1, 2, 0, 3]
// [2, 1, 0, 3]
// [3, 1, 0, 2]
// [1, 3, 0, 2]
// [0, 3, 1, 2]
// [3, 0, 1, 2]
// [1, 0, 3, 2]
// [0, 1, 3, 2]
// [0, 2, 3, 1]
// [2, 0, 3, 1]
// [3, 0, 2, 1]
// [0, 3, 2, 1]
// [2, 3, 0, 1]
// [3, 2, 0, 1]
// [3, 2, 1, 0]
// [2, 3, 1, 0]
// [1, 3, 2, 0]
// [3, 1, 2, 0]
// [2, 1, 3, 0]
// [1, 2, 3, 0]

}
```

# superslice

二分探索が実装されています。`std::slice`にも`binary_search()`はありますが、
目的の要素が複数含まれる場合はいずれか一つのインデックスが返されるという使いづらい仕様のため、
C++の`lower_bound()`や`upper_bound()`に慣れている人は同じインターフェースのこちらの方が使いやすいかもしれません。

```rust
use superslice::*;
 
fn main() {
    let b = [1, 3];
 
    assert_eq!(b.lower_bound(&1), 0);
    assert_eq!(b.upper_bound(&1), 1);
    assert_eq!(b.equal_range(&3), 1..2);
}
```

# itertools

itertoolsではiteratorに関する便利なメソッドが色々提供されています。
競技プログラミングで特に便利そうと思ったのは以下のメソッドです。

- `.dedup()` : 連続する重複した要素を取り除く
- `.unique()` : 重複した要素を取り除く
- `.combinations(usize)` : 指定した個数の組み合わせのiteratorを返す
- `.combinations_with_replacement(usize)` : 指定した個数の重複組み合わせのiteratorを返す
- `.permutations(usize)` : 指定した個数の順列のiteratorを返す
- `.join(&str)` : 区切り文字を要素間に挟んだStringを返す
- `.format(&str)`: 区切り文字を要素間に挟んだFormatを返す

それぞれ例を書いてみます。

```rust
use itertools::Itertools;

fn main() {
    let x = [1, 1, 2, 3, 2, 2, 1];
    println!("{:?}", x.iter().dedup().collect::<Vec<_>>()); // [1, 2, 3, 2, 1]
    println!("{:?}", x.iter().unique().collect::<Vec<_>>()); // [1, 2, 3]
    let y = ["a", "b", "c"];
    println!("{:?}", y.iter().combinations(2).collect::<Vec<_>>()); // [["a", "b"], ["a", "c"], ["b", "c"]]
    println!("{:?}", y.iter().combinations_with_replacement(2).collect::<Vec<_>>());
    // [["a", "a"], ["a", "b"], ["a", "c"], ["b", "b"], ["b", "c"], ["c", "c"]]
    println!("{:?}", y.iter().permutations(2).collect::<Vec<_>>());
    // [["a", "b"], ["a", "c"], ["b", "a"], ["b", "c"], ["c", "a"], ["c", "b"]]
    println!("{:?}", y.iter().join("-")); // "a-b-c"
    println!("{}", x.iter().format("^")); // 1^1^2^3^2^2^1
}
```

# proconio

入力データのパースが簡単になるライブラリです。c++のiostreamに慣れてしまうと他の言語での入力をパースするコードは冗長に感じてしまいますが、
`proconio::input!`マクロを使うとかなり簡単に入力をパースすることができます。

```rust
use proconio::input;

// input
// ------
// 5 3
// 0 1 x
// 1 2 y
// 0 4 foo

fn main() {
    input! {
        n: usize,
        m: usize,
        xs: [(usize, usize, String); m],
    }

    println!("{:?}", &xs[m-1]); // (0, 4, "foo")
}
```

# その他

あまり利用用途は多くないですが特定の問題では役に立ちそうなcrateも簡単に紹介します。

- ndarray: 多次元配列の実装です。
- nalgebra: 低次元の線形代数ライブラリで6次元程度までの線形変換や行列分解などが実装されています。
- indexmap: 挿入順が保存されるハッシュテーブル`IndexMap`の実装です。`std::collection::HashMap`と同じようなAPIを持っています。
挿入順が保存される集合`IndexSet`もあります。
- ordered_float: rustの浮動小数点数は`Ord` (`PartialOrd`) traitを満たしていないので、
そのような値が期待されるようなgenericな関数と組み合わせて使うことができません。
`Ord`を要求する関数を浮動小数点数に対して使いたい時には`order_float::OrderedFloat<f64>` が利用できます。



# まとめ

今回の言語アップデート後のrustで利用可能なcrateをいくつか紹介してきました。
今まではバージョンも古く、stdしか使えない環境で競技プログラミングをするには自作ライブラリを揃えないとなかなか難しいと感じていましたが、
今回紹介したようなcrateは使いこなせば実装量が減り、バグらせにくいコードを書くことができると思うのでぜひマスターしたいところです。

他にも定数倍高速化やコード長の削減、抽象化に役立ちそうなcrateもいくつかあるので興味がある人は調べてみても良いかもしれません。