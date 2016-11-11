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
    if (handle->timeout > loop->time)  // ここで発火時刻を過ぎているか検査
      break;

    uv_timer_stop(handle);
    uv_timer_again(handle);
    handle->timer_cb(handle);  // ここでコールバック
  }
}
```
