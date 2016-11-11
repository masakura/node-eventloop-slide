```c
// https://github.com/nodejs/node/blob/v6.5.0/deps/uv/src/unix/linux-core.c#L170-L419
void uv__io_poll(uv_loop_t* loop, int timeout) {
// snip ---------------------------------------- snip
  for (;;) {
    /* See the comment for max_safe_timeout for an explanation of why
     * this is necessary.  Executive summary: kernel bug workaround.
     */
// snip ---------------------------------------- snip
      nfds = uv__epoll_wait(loop->backend_fd,    // ここでデータが来るまでポーリング
                            events,              // 処理はブロックされる
                            ARRAY_SIZE(events),  // (データが来る以外でも、I/O 関係)
                            timeout);
// snip ---------------------------------------- snip
        if (w == &loop->signal_io_watcher)
          have_signals = 1;
        else
          w->cb(loop, w, pe->events);  // ここでコールバックを呼び出す
```
