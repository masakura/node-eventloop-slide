### [deps/uv/src/queue.h](https://github.com/nodejs/node/blob/v6.5.0/deps/uv/src/queue.h)

libuv で使われているキューマクロ。かなり読みにくい。

構造は`双方向リンク + リング`のリスト。

```text
+---------------------------------+
|                                 |
+-[QUEUE]--[E1]--[E2]--[E3]--[E4]-+
```
<!-- .element: style="width: 400px;" -->
