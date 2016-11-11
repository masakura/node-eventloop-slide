```javascript
// https://github.com/nodejs/node/blob/v6.5.0/src/node.cc#L4589-L4603
      do {
        v8_platform.PumpMessageLoop(isolate);
        more = uv_run(env->event_loop(), UV_RUN_ONCE); // <- ここ

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
