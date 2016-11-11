```javascript
// https://github.com/nodejs/node/blob/v6.5.0/deps/uv/src/unix/timer.c#L62-L90
int uv_timer_start(uv_timer_t* handle,
                   uv_timer_cb cb,
                   uint64_t timeout,
                   uint64_t repeat) {

// snip...

  heap_insert((struct heap*) &handle->loop->timer_heap,
              (struct heap_node*) &handle->heap_node,
              timer_less_than);  // 発火時刻順になるよう挿入
  uv__handle_start(handle);

  return 0;
}
```
