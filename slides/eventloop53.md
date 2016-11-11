```c
// https://github.com/nodejs/node/blob/v6.5.0/deps/uv/src/unix/core.c#L350-L356
    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);  // 待機する時刻を計算

    uv__io_poll(loop, timeout);  // 指定時間だけポーリングする
    uv__run_check(loop);
    uv__run_closing_handles(loop);
```

| 値 | 説明
| ---- | ----
| 1 以上 | タイマーハンドラーがあるとき。直近の発火事故くまでの時間。
| 0 | キューが空だったり、`setImmediate()` ハンドラーがある時だったり。
| -1 | 無限に待つ。
<!-- .element: style="font-size: 75%;" -->
