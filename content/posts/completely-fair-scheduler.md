---
title: "Completely Fair Schedulerについて"
date: 2015-12-10T00:00:00+09:00
draft: false
---

この記事はLinux advent calendar 10日目の記事です。

以前までLinux kernel はO(1)スケジューラというものを使っていました。これは「詳解Linux カーネル第３版」に詳しく説明があると思います。
この記事ではその後、v2.6.23から導入されたCompletely Fair Scheduler(CFS)についてまとめます。多分に間違いがあるかもしれないのでまさかりを投げてください。

## スケジューリングとは

あまりLinuxカーネルに馴染みのない人のためにざっくりスケジューリングについて説明します。

LinuxやWindowsといったOSでは普通、プロセスは複数同時に動いているように見えます。
しかし、実は短時間でプロセスを切り替えることで人間からは同時に動いているように見えているだけなのです。
（追記：最近ではマルチコアCPUが一般的ですが、一つのコアに関しては同じことが言えます）
このとき順番にプロセスを切り替えるというのが最も単純なスケジューラですが、もちろんすべてのプロセスは対等ではありませんし、
それぞれのプロセスに状態があり、デバイスの読み書きのような時間のかかる処理の完了を待っていたり、他プロセスからのシグナルを待っていたりして、実行可能ではないプロセスも存在します。
そのため、Linuxカーネルはそこそこ複雑なスケジューリングの機構を用意することで無駄の少なく、応答性の高いシステムを実現しています。

## スケジューリングクラス

Linuxはスケジューリングクラスという概念を導入しています。
これはそれぞれのプロセスが異なるスケジューリングポリシーを持つことができるというもので、v4.xでは

* stop_sched_class
* dl_sched_class
* rt_sched_class
* fair_sched_class
* idle_sched_class

の5つがあります。
これらのスケジューリングクラスは上記の順で連結リストになっていて、
上の方が優先度が高くなっています。

この中のfair_sched_class がCFSスケジューラに相当するものです。

## Completely Fair Scheduler (CFS) 

ここからが本題のCFSに関する説明です。`kernel/sched/fair.c`に実装があります。

CFSではプロセスを赤黒木で管理します。赤黒木は平衡２分木のひとつで値の挿入、削除、探索が $\displaystyle O(\log n)$
でできるというデータ構造です。また、木の一番左（値が最小）の要素はキャッシュされているので
$\displaystyle O(1)$ で取得できます。

この赤黒木は `sched_entity`を要素、その中の`vruntime`をキーとしています。
`sched_entity`はかならずしもプロセスに対応するものではありませんが、
とりあえず、プロセスに対して１つ存在すると思ってもらって大丈夫です。（Group Scheduling が絡んでいます）

赤黒木のキーとなるvruntimeですが、これはプロセスの仮想実行時間を表しています。
仮想実行時間とはその名の通りプロセスが実行された合計時間（のようなもの）です。
この仮想実行時間が短いプロセスから選んでスケジューリングすることで
すべてのプロセスに平等なスケジューリングを実現しようとしています。

### データ構造

スケジューリングに関する構造体は次のようなものがあります。

* [task_struct](http://lxr.free-electrons.com/source/include/linux/sched.h?v=4.0#L1277)
* [sched_class](http://lxr.free-electrons.com/source/kernel/sched/sched.h?v=4.0#L1143)
* [sched_entity](http://lxr.free-electrons.com/source/include/linux/sched.h?v=4.0#L1164)
* [rq](http://lxr.free-electrons.com/source/kernel/sched/sched.h?v=4.0#L538)
* [cfs_rq](http://lxr.free-electrons.com/source/kernel/sched/sched.h?v=4.0#L336)
* [load_weight](http://lxr.free-electrons.com/source/include/linux/sched.h?v=4.0#L1111)


順に説明していきます。

### task_struct

とにかく長いです。400行以上あります。
プロセスに関する情報はすべてこの構造体に収められているのでしょうがないです。
特に大事そうなフィールドだけ紹介します。

#### state

プロセスの状態を簡潔に表します。次のような値をとるようです。[`include/linux/sched.h`](http://lxr.free-electrons.com/source/include/linux/sched.h?v=4.0#L203)
```c
#define TASK_RUNNING		0
#define TASK_INTERRUPTIBLE	1
#define TASK_UNINTERRUPTIBLE	2
#define __TASK_STOPPED		4
#define __TASK_TRACED		8
/* in tsk->exit_state */
#define EXIT_DEAD		16
#define EXIT_ZOMBIE		32
#define EXIT_TRACE		(EXIT_ZOMBIE | EXIT_DEAD)
/* in tsk->state again */
#define TASK_DEAD		64
#define TASK_WAKEKILL		128
#define TASK_WAKING		256
#define TASK_PARKED		512
#define TASK_STATE_MAX		1024
```

#### on_rq

rqにそのプロセスが入っているかどうかを表します。
rqは後述します。

#### prio, static_prio, normal_prio

プロセスの優先度を表します。
優先度は0から140までの値をとり、値が小さい方が優先度が高くなります。
0から99まではリアルタイム優先度、100から139までがCFSで扱うプロセスの優先度となります。
このプロセスの優先度はnice値によって決められ、nice値の -20 ~ 19が優先度の100 ~ 139にマップされてます。
また、何もしないidleプロセスは140と決められています。


#### sched_class

プロセスの属するスケジューリングクラスを表します。

#### sched_entity

プロセスに対応する`sched_entity`を表します。

### sched_class

スケジューリングクラスを表す構造体です。
```c
struct sched_class {
	const struct sched_class *next;

	void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
	void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
	void (*yield_task) (struct rq *rq);
	bool (*yield_to_task) (struct rq *rq, struct task_struct *p, bool preempt);

	void (*check_preempt_curr) (struct rq *rq, struct task_struct *p, int flags);

	/*
	 * It is the responsibility of the pick_next_task() method that will
	 * return the next task to call put_prev_task() on the @prev task or
	 * something equivalent.
	 *
	 * May return RETRY_TASK when it finds a higher prio class has runnable
	 * tasks.
	 */
	struct task_struct * (*pick_next_task) (struct rq *rq,
						struct task_struct *prev);
	void (*put_prev_task) (struct rq *rq, struct task_struct *p);

/* 中略 */

	void (*set_curr_task) (struct rq *rq);

/* 中略 */
};
```

関数ポインタがたくさんあり、スケジューリングクラスごとに異なる操作を統一して行えるようにしています。
上に書いたスケジューリングクラスがそれぞれどこかで宣言されています。


### sched_entity

上で説明した赤黒木のノードに対応する構造体です。
```c
struct sched_entity {
	struct load_weight	load;		/* for load-balancing */
	struct rb_node		run_node;
	struct list_head	group_node;
	unsigned int		on_rq;

	u64			exec_start;
	u64			sum_exec_runtime;
	u64			vruntime;
	u64			prev_sum_exec_runtime;

/* 略 */
};
```
この中に`vruntime`や実際の赤黒木のノードか収められています。
`load`は優先度に基づいた重みが格納されます。

### rq

これも結構長いです。
`rq`はrunqueueの略で実行可能なプロセス(の`sched_entity`)を管理する構造体です。
よく使いそうなフィールドだけ紹介します。

- `nr_running`: 実行可能なプロセスの数
- `cfs`: CFSに属するプロセスの管理をする構造体
- `dl`, `rt`: それぞれデッドラインスケジューラ、リアルタイムスケジューラで使われる構造体
- `cfs_rq`: CFSに関連するデータを含む構造体

`cfs_rq`構造体の定義は次のようになっています。

```c
struct cfs_rq {
	struct load_weight load;
	unsigned int nr_running, h_nr_running;

	u64 exec_clock;
	u64 min_vruntime;

	struct rb_root tasks_timeline;
	struct rb_node *rb_leftmost;

	/*
	 * 'curr' points to currently running entity on this cfs_rq.
	 * It is set to NULL otherwise (i.e when none are currently running).
	 */
	struct sched_entity *curr, *next, *last, *skip;
/* 略 */
};
```
赤黒木の根や、その最も左のノードへのポインタなどもこの中に含まれています。

### load_weight

`load_weight` 構造体の中身はこれだけです。`sched_entity`や`cfs_rq`の中に含まれています。

```c
struct load_weight {
	unsigned long weight;
	u32 inv_weight;
};
```
重みとその逆数だけです。
何の重みかについては後述します。

## 処理

ここからはプロセスの一生を通じてどのようにスケジューリングされるかおおまかに追っていきます。

まずプロセスが生成されると、スケジューリング関連の値は[`sched_fork()`](http://lxr.free-electrons.com/source/kernel/sched/core.c#L2161)の中で初期化されます。
そのあと[`task_fork_fair()`](http://lxr.free-electrons.com/source/kernel/sched/fair.c?v=4.0#L7733)の中で`sched_entity`の`vruntime`の値は親のvruntimeの値になります。
しかしその後[`place_entity()`](http://lxr.free-electrons.com/source/kernel/sched/fair.c#L2940) が呼ばれ、`vruntime`の値はそれまでの`vruntime`の値に
１回実行権が与えられたときの実行時間が足されます。
これにより、次々とforkが起こる事で新たに作られたプロセスばかりが実行されないようになります。
(`task_fork_fair()`の中で`vruntime`から`cfs_rq.min_vruntime`の値が引かれていますが、これはまた`enqueue_entity()`のなかで足されて戻ります。
おそらく眠りから覚めたプロセスと扱いを揃えるため？)

また、`sched_entity`中の`load.weight`がプロセスの`static_prio`に基づいて[`set_load_weight()`](http://lxr.free-electrons.com/source/kernel/sched/core.c?v=4.0#L767)で初期化されます。
実際の重みは優先度ごとにあらかじめ値が決められています。([`prio_to_weight[]`](http://lxr.free-electrons.com/source/kernel/sched/sched.h?v=4.0#L1101))
重みは優先度が高いほど大きく、優先度が低いほど小さく定められています。式で書くと
`weight = 1.25^(-nice) * 1024`ぐらいみたいです。
最後に`do_fork()`から`wake_up_new_task()`が呼ばれ、プロセスが赤黒木に入ります。

その後、`schedule()`の中で[`pick_next_task()`](http://lxr.free-electrons.com/source/kernel/sched/core.c#L2983)が呼ばれることで次に実行するプロセスを選択します。
プロセスの選択はまず、すべての実行可能なプロセスがCFSクラスに属するとき、直接CFSクラスの`pick_next_task()`を、
そうでないときは、すべてのスケジューリングクラスの`pick_next_task()`を順に呼び出しています。
CFSでは次に赤黒木の最も左の要素(次に実行されるプロセスの`sched_entity`)を削除し、それまで走っていたプロセスの`sched_entity`を木に挿入するという操作をします。

また、`pick_next_task_fair()`の中の[`update_curr()`](http://lxr.free-electrons.com/source/kernel/sched/fair.c#L700)で、
それまで走っていたプロセスの`vruntime`に実行時間が重みで割られた値が足されます。これにより優先度が高いプロセスはほとんど実行されてないとみなされ、
優先的に実行されやすくなります。一方優先度の低いプロセスは長い間実行されたとみなされるため、次に実行されるのはしばらくあとになる可能性が高くなります。

これを繰り返すことで、常に仮想実行時間のもっとも小さいプロセスが選択されて、平等に（感じられるように）スケジューリングが進んでいきます。

プロセスが死ぬときは、`do_exit()`の中で`tsk->state = TASK_DEAD`と設定することで、`__schedule()`の中で
`prev->on_rq`が0になり、`pick_next_task()`の中で`put_prev_entity()`が呼ばれても、そのプロセスを赤黒木に挿入しなくなります。


以上で終わりです。


あまり時間がなくてこの程度のことしか調べられませんでしたが、実際のところはこれに加えて
SMPにおけるロードバランシングや、グループスケジューリングによりもっと複雑なことをしています。

ほんとはもっとちゃんとまとめる形にしたかったのですが、
こんな乱文になってしまい無念..


## 参考文献

* 詳解Linux カーネル第３版
* https://www.ibm.com/developerworks/jp/linux/library/l-completely-fair-scheduler/
* http://amiq11.tumblr.com/post/41152252069/linux%E3%82%B9%E3%82%B1%E3%82%B8%E3%83%A5%E3%83%BC%E3%83%A9-%E6%A6%82%E8%A6%81
