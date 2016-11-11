```c
// https://github.com/nodejs/node/blob/v6.5.0/deps/uv/src/unix/core.c#L343-L357
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
// snip ---------------------------------------- snip
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
// snip ---------------------------------------- snip
```
