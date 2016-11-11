```c
// https://github.com/nodejs/node/blob/v6.5.0/deps/uv/src/unix/linux-core.c#L170-L419
void uv__io_poll(uv_loop_t* loop, int timeout) {
// snip ---------------------------------------- snip
  while (!QUEUE_EMPTY(&loop->watcher_queue)) {
    q = QUEUE_HEAD(&loop->watcher_queue);  // ここでキューから処理を取り出す
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);

    w = QUEUE_DATA(q, uv__io_t, watcher_queue);
// snip ---------------------------------------- snip
    if (uv__epoll_ctl(loop->backend_fd, op, w->fd, &e)) {  // epoll キューに処理を登録
// snip ---------------------------------------- snip
```
