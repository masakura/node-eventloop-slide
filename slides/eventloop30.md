### I/O Callbacks
* 一部でしか使われていないっぽい
* 追いかけてません...

```text
+-----------------+
|     timers      |<-+
+-----------------+  |
        |            |
+-----------------+  |
|  I/O callbacks  |  |
+-----------------+  |
        |            |
+-----------------+  |
|      poll       |  |
+-----------------+  |
        |            |
+-----------------+  |
|     check       |  |
+-----------------+  |
        |            |
+-----------------+  |
| close callbacks |--+
+-----------------+
```
<!-- .element: style="width: 300px;" class="left" -->
