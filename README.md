# Node.js イベントループを調べてみた
東京 Node 学園祭用スライド。


## 案
### 冒頭
調べることになったきっかけについて書く。

* JavaScript の非同期の話を他の人に説明するときに単純なイベントループの絵を書く
* 本当に合ってのこれ?
  - Linux には非同期しステームコールがないらしい
  - (日本語で) 解説している記事をほとんど見ない
  - setImmediate とか、明らかに優先権がありそう
* ブラウザーのコードを読むのは大変そうなので、Node.js のコードを読むことに

## 解説
時間切れになる前に資料とか結果を紹介

### 参考資料
* [公式ドキュメント: The Node.js Event Loop, Timers, and process.nextTick()](https://github.com/nodejs/node/blob/master/doc/topics/event-loop-timers-and-nexttick.md) ここに大体書いてあった (後から気がついた...)
* [Software Design 10 月号: Web サーバーはなぜ動くのか? Node.js](http://gihyo.jp/magazine/SD/archive/2016/201610) by Yosuke FURUKAWA
* [Node Interactive Europe 2016 発表資料: Event Loop, yay](https://drive.google.com/file/d/0B1ENiZwmJ_J2a09DUmZROV9oSGc/view) by Bert Belder
  - [参加レポート: Node Interactive Europe 2016 に参加しました。](http://yosuke-furukawa.hatenablog.com/entry/2016/09/30/123942) by Yosuke FURUKAWA


### 結果
思ってたのと全然違う

* イベントループは V8 側にあると思ったら libuv 側だった
* キューを何個も持つ、かなり複雑な構造をしていた
* それでも、単純なイベントループに見えるようになっているのがすごいと思った
* 今回調べたのは Linux 用、Mac や Windows はまたちょっと違う


### イベントループの構造
イベントループの呼び出しは src/node.cc 内にあった。

```javascript
// https://github.com/nodejs/node/blob/v6.5.0/src/node.cc
    {
      SealHandleScope seal(isolate);
      bool more;
      do {
        v8_platform.PumpMessageLoop(isolate);
        more = uv_run(env->event_loop(), UV_RUN_ONCE);

        if (more == false) {
          v8_platform.PumpMessageLoop(isolate);
          EmitBeforeExit(env);

          // Emit `beforeExit` if the loop became alive either after emitting
          // event, or after running some callbacks.
          more = uv_loop_alive(env->event_loop());
          if (uv_run(env->event_loop(), UV_RUN_NOWAIT) != 0)
            more = true;
        }
      } while (more == true);
}
```

* `uv_run()` 関数が イベントループを実行する関数
* V8 のイベントループだと思われるものも呼び出してはいる
* でも、使われている感じがしない

イベントループそのものは、libuv 側で定義されている。

```javascript
// https://github.com/nodejs/node/blob/v6.5.0/deps/uv/src/unix/core.c#L343-L357
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
// snip...
  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    uv__run_timers(loop);
    ran_pending = uv__run_pending(loop);
    uv__run_idle(loop);
    uv__run_prepare(loop);

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    uv__io_poll(loop, timeout);
    uv__run_check(loop);
    uv__run_closing_handles(loop);
// snip...
```

公式ドキュメントから抜粋するけど、フェーズに分かれていて、一つ一つ役割が違う。

```
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```

#### timers
`setTimeout()` / `setInterval()` 関数用、他でも使われているけど、`setImmediate()` 関数では使われていない

```javascript
// https://github.com/nodejs/node/blob/v6.5.0/deps/uv/src/unix/timer.c#L150-L167
void uv__run_timers(uv_loop_t* loop) {
  struct heap_node* heap_node;
  uv_timer_t* handle;

  for (;;) {
    heap_node = heap_min((struct heap*) &loop->timer_heap);
    if (heap_node == NULL)
      break;

    handle = container_of(heap_node, uv_timer_t, heap_node);
    if (handle->timeout > loop->time)
      break;

    uv_timer_stop(handle);
    uv_timer_again(handle);
    handle->timer_cb(handle);
  }
}
```

発火時刻になったら実行待ちキューに登録されるのではなく、キューの中から発火時刻を過ぎたものが実行されるようになっている。

また、このキューは発火時刻が直近のものから並んでいるので、キューを全て検査しなくてもいいようになっている。

```javascript
// https://github.com/nodejs/node/blob/v6.5.0/deps/uv/src/unix/timer.c#L62-L90
int uv_timer_start(uv_timer_t* handle,
                   uv_timer_cb cb,
                   uint64_t timeout,
                   uint64_t repeat) {

// snip...

  heap_insert((struct heap*) &handle->loop->timer_heap,
              (struct heap_node*) &handle->heap_node,
              timer_less_than);
  uv__handle_start(handle);

  return 0;
}
```

タイマーハンドラーのキューに登録するときに、発火時刻順に並ぶようになっている。

というわけで、タイマー関数は必ず発火順に実行されます。

```javascript
setTimeout(() => { console.log('1000A'); }, 1000); // (2)
setTimeout(() => { console.log('500'); }, 500); // (1)
setTimeout(() => { console.log('1000B'); }, 1000); // (3)

// 1500 ms 待つ
const last = Date.now() + 1500;
while (Date.now() < last) {}
```


#### I/O Callbacks
公式ドキュメントより引用。

> This phase executes callbacks for some system operations such as
> types of TCP errors. For example if a TCP socket receives
> ECONNREFUSED when attempting to connect, some *nix systems want to
> wait to report the error. This will be queued to execute in the I/O
> callbacks phase.

コールバックが呼び出されるときに使われるようだけど、あまり大事そうじゃなかかったので、調べてません。


#### idle, prepare
公式ドキュメントより引用。というかこれしか書いていない。

> only used internally

内部使用なので調べてません。


#### poll
Node.js が最も Node.js らしい部分! でも、あんまり追いかけられていない。

```javascript
// https://github.com/nodejs/node/blob/v6.5.0/deps/uv/src/unix/linux-core.c#L170-L419
void uv__io_poll(uv_loop_t* loop, int timeout) {

// snip...

  while (!QUEUE_EMPTY(&loop->watcher_queue)) {
    q = QUEUE_HEAD(&loop->watcher_queue);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);

    w = QUEUE_DATA(q, uv__io_t, watcher_queue);

// snip...

    if (uv__epoll_ctl(loop->backend_fd, op, w->fd, &e)) {

// snip...
```

I/O 処理を Linux のシステムコールの epoll のキューに全て登録している。

```
// https://github.com/nodejs/node/blob/v6.5.0/deps/uv/src/unix/linux-core.c#L170-L419
void uv__io_poll(uv_loop_t* loop, int timeout) {

// snip...

  for (;;) {
    /* See the comment for max_safe_timeout for an explanation of why
     * this is necessary.  Executive summary: kernel bug workaround.
     */

// snip...

      nfds = uv__epoll_wait(loop->backend_fd,
                            events,
                            ARRAY_SIZE(events),
                            timeout);

// snip...

        if (w == &loop->signal_io_watcher)
          have_signals = 1;
        else
          w->cb(loop, w, pe->events);
```

`epoll_wait` システムコールを呼び出して、ポーリングを開始している。データが読み取れるなどの状態になるまでひたすら待機する。準備が整ったら、それを処理するコールバックを呼び出して、処理してもらう感じ。

ところで、ひたすら待機するんだったら、タイマー関数はどうやって実行するの? ってことなんですが、`uv__io_poll()` 関数を呼び出す側で、直近に発火するタイマー関数の発火時刻までしか待機しないように `timeout` 引数を渡しています。

```javascript
// https://github.com/nodejs/node/blob/v6.5.0/deps/uv/src/unix/core.c#L350-L356
    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    uv__io_poll(loop, timeout);
    uv__run_check(loop);
    uv__run_closing_handles(loop);
```

epoll で待機する時間を、いろんな情報を元に決めている

* 1 以上 - タイマーハンドラーがあるとき、直近の発火時刻までの時間
* 0 - キューが空とか、実行しなければいけない処理があるとき (おそらく `setImmediate()` ハンドラーがあるときとか)
* -1 - 無限に待つ


#### check
公式ドキュメントによると、`setImmediate()` 関数用らしい。つまり、poll フェーズで実行されたコールバックで登録された `setImmediate()` のハンドラーは次のフェーズで実行され、タイマーハンドラーより先に動作することが保証されている。

```javascript
const fs = require('fs');

fs.readFile(__dirname + '/immediate.js', () => {
  setTimeout(() => { console.log('timeout'); }, 100); // (2)

  // 200ms 待つ
  const last = Date.now() + 200;
  while (Date.now() < last) {}

  setImmediate(() => { console.log('immediate'); }); // (1)
});
```

詳しくは、[公式ドキュメント](https://github.com/nodejs/node/blob/v6.5.0/doc/topics/the-event-loop-timers-and-nexttick.md#setimmediate-vs-settimeout) に書いてあります。


#### close callbacks
公式ドキュメントより。

> If a socket or handle is closed abruptly (e.g. socket.destroy()),
> the 'close' event will be emitted in this phase. Otherwise it will
> be emitted via process.nextTick().

そんなに大事そうじゃないので、追いかけてません。でも、`process.nextTick()` 関数で使われているのは要注意かもしれません。


### おまけ
#### ファイル構造
| 場所 | 役割
| ---- | ----
| src | Node.js のソースコード (C++)
| lib | Node.js オブジェクト (JavaScript)
| deps/ | 依存ライブラリ
| deps/uv | libuv
| deps/uv/src/queue.h | キューマクロ
| deps/uv/unix | libuv Unix 用ソースコード
| deps/uv/src/unix/core.c | libuv イベントループ等
| deps/uv/win | libuv Windows 用ソースコード
| deps/v8 | v8
| deps/... | 他


#### queue.h
https://github.com/nodejs/node/blob/v6.5.0/deps/uv/src/queue.h

`libuv` で使われている、汎用のキューマクロ。心底読みにくい。しかも、あまり読まなくても良かったというオチが...

構造は、双方向リンク + リングのリスト。

```
+---------------------------------+
|                                 |
+-[QUEUE]--[E1]--[E2]--[E3]--[E4]-+
```

キューの最初のエントリに挿入するのも、最後に挿入するのもそれほどコストがかからないので素晴らしいと思います。


### 最後に
* fs.* は Thread Pool を使っているらしいけど、見つけられなかった
* ウェブブラウザーのイベントループはまた違いそう
* ECMA-262 でどこまで定義されているのかな?
* このあたり知っている人がいたら教えてください!
