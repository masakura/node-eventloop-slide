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
<!-- .element: style="width: 300px;" -->

公式ドキュメントより <!-- .element: class="ref" -->
