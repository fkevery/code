# 屏幕点击时间

> 该方法主要用于**获取点击应用图标的时间**

因为用户点击应用图标时，应用完全没有启动，不可能通过 apk 中日志输出等形式拿到时间。

而系统中每一次点击事件，都可以通过 *adb shell getevent* 命令拿到输出，只要将该输出导出到电脑的某个文件，然后监听该文件内容的变化，就可以拿到点击触发的时间。

主要思路：
1. 使用 **adb shell getevent >> ~/Desktop/xx.txt** 将输出导出到电脑中的 xx.txt 文件
   
2. 使用代码死循环读取 xx.txt 文件的长度，只要长度发生变化可以认为发生了点击事件

第二步代码如下：
```c++
std::time_t getTimeStamp()
{
    std::chrono::time_point<std::chrono::system_clock,std::chrono::milliseconds> tp = std::chrono::time_point_cast<std::chrono::milliseconds>(std::chrono::system_clock::now());//获取当前时间点
    std::time_t timestamp =  tp.time_since_epoch().count(); //计算距离1970-1-1,00:00的时间长度
    return timestamp;
};

// main 方法中
long a = 0;
long lastTime = getTimeStamp();
FILE* f = fopen("~/Desktop/outtt.txt", "rw");
while (true) {
    if(f == NULL){
        continue;
    }
    fseek(f, 0, SEEK_END);
    long b = ftell(f);
    long curTime = getTimeStamp();
    // 一次事件会向文件中写两次，这里忽略 1s 内发生的变化
    if(curTime - lastTime > 1000 && b != a){
        lastTime = getTimeStamp();
        cout << lastTime << endl;
    }
    a = b;
}
```