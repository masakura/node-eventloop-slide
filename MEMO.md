イベントループの開始は src/node.cc:4589:StartNodeInstance 関数の中

* V8 のイベントループ (MessageLoop) も呼び出している
  - あまりこちらに言及した資料を見ないので、V8 のイベントループは使われていないのかも
  - ちょっとびっくりした
* uv_run 関数でイベントループを呼び出している
  - UV_RUN_ONCE と UV_RUN_NOWAIT の違いは分からない
  * ここはそれほど複雑じゃなかった

イベントループの実装は deps/uv/src/unix/core.c

* 名前の通り、Windows 用の実装は別
* 単純なキューではなく、工程に分かれている
* libuv 側がイベントループの詳細を用意している
* コードで追っかけると複雑
  - update_time
  - run_timers
  - run_pending
  - run_idle
  - run_prepare
  - backend_timeout
  - io_poll
  - run_check
  - run_closing_handles
  - update_time [RUN_ONCE ONLY]
  - run_timers [RUN_ONCE ONLY]
  - loop_alive


docs/topics/the-event-loop-timers-and-nexttick.md このドキュメントが神

* フェーズって考え方を持っている
  - timers - スケジュールされた setTimeout/setInterval が実行される?
  - I/O callbacks - よくあるコールバック (除く、タイマー、setImmediate、close)
    + ちょいとよくわからん
  - idle, prepare - 内部仕様なので気にしない方向で
  - poll - ???
  - check - setImeddiate
  - close callbacks socket.on('close', ...)

* update_time
  - イベントループで使われる時刻 loop->time を更新
* run_times
  - break が使われているので、タイマーハンドラーはソート済みっぽい
  - setTimeout(() => {}, 1000); setTimeout(() => {}, 0); 後者がキューの先頭にある


### io-poll
Linux の場合、epoll で監視している。

* `epoll_wait` で I/O 処理一つでも完了するまで無期限に監視をする (ブロッキングされる)
  - ただし、setImmediate とか Timer が設定されていたら、それらが実行される時間あたりで監視をやめ、次のフェーズに移動するようにする


### Timer
ここでタイマーハンドラーを登録する?
deps/uv/src/fs-poll.c:  err = uv_timer_init(loop, &ctx->timer_handle);
uv_fs_poll_start
StatWatcher::Start

uv_timer_start ここかな?
* 近いところから実行されるよう、ヒープに追加されてるのが分かる

lib/timers.js:
src/timer_wrap.cc:TimerWrap::Start()
uv_timer_start

実際に実行されるのは...


### setImmediate
Timer とは別!

lib/timers.js:
node.cc:


### fs-poll?
ファイルシステムの状態変更監視用っぽい。
deps/uv/src/fs-poll.c:poll_cb で、uv_timer_start を呼び出している。ポーリング用?
uv_fs_poll_start:StatWatcher::Start


### FileSystem
epoll 使ってポーリング (= ブロッキング I/O) だと思うんだけど、https://drive.google.com/file/d/0B1ENiZwmJ_J2a09DUmZROV9oSGc/view こっちで見ると、fs.* Thread pool ってなってて、内部でスレッドが使われている? よく分からなかった。
