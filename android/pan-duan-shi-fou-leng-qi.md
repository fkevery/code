# 判断是否冷启

以<mark style="color:red;">是否启动 MainActivity 当作判断条件</mark>

当一个应用在启动时，主线程的 MessageQueue 中已经存储有要启动的 activity 的 Intent。因此<mark style="color:red;">可以反射遍历 messageQueue，拿到相应的 obj 判断是不是 MainActivity</mark>。
