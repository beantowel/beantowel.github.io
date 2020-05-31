---
layout: post
title:  "go语言闭包问题一例——循环变量引用捕获"
date:   2020-05-31 13:43:00 +0800
categories: golang
---
刚学习go语言时，其官方文档中就介绍了一种常见的迷惑现象，即 [Using goroutines on loop iterator variables](https://github.com/golang/go/wiki/CommonMistakes#using-goroutines-on-loop-iterator-variables):

```go
for i := 0; i < 5; i++ {
    go func() {
        fmt.Printf("i=%v\n", i)
    }()
}
time.Sleep(100 * time.Millisecond)
```

以上程序会输出如下结果而不是第一眼看上去应该得到的 `i=0，1，2，3，4`

```plain
» go run ./test.go
i=5
i=5
i=5
i=5
i=5
```

最显然的一个原因当然是go语言的闭包是通过引用捕获 (capture by reference) 自由变量的，即匿名函数中的`i`是对循环变量`i`的引用，所以当循环结束，程序在`time.Sleep`调度协程运行时，5个协程都会得到同一个值`i=5`。但是这个锅真的只是引用捕获来背吗？我认为另一个造成迷惑的原因是 GC，因为有了 GC，引用捕获悄悄的延长了循环变量`i`的生存时间/scope。上述代码实际上是在循环之外使用了循环变量，而没有任何错误提示！因此 lexical scope + capture by reference 造成了 dynamic scope 的错觉。

为了更好地论证这个观点，下面拿 C++ 举几个例子，毕竟 C++ 的闭包是可以显式指定 capture by reference 还是 capture by value 的

使用 capture by value `[=]` 时，输出和预期一样。

```cpp
std::vector<std::future<void>> futures;
for(int i=0; i<5; i++) {
    auto f = [=](){
        std::cout << "i=" << i << std::endl;
    };
    futures.push_back(
        std::async(
            std::launch::deferred,
            f));
}
for(auto &fut: futures){
    fut.get();
}
```

```plain
i=0
i=1
i=2
i=3
i=4
```

使用 capture by reference `[&]` 时

```cpp
std::vector<std::future<void>> futures;
for(int i=0; i<5; i++) {
    auto f = [&](){
        std::cout << "i=" << i << std::endl;
    };
    futures.push_back(
        std::async(
            std::launch::deferred,
            f));
}
for(auto &fut: futures){
    fut.get();
}
```

输出...

```plain
i=5
i=5
i=5
i=5
i=5
```

嗯？居然和 go 的问题一样，没有报错？稍微修改一下调用方式：

```cpp
void capRefDangle(std::vector<std::future<void>> &futures)
{
    for(int i=0; i<5; i++) {
        auto f = [&](){
            std::cout << "i=" << i << std::endl;
        };
        futures.push_back(
            std::async(
                std::launch::deferred,
                f));
    }
    return;
}

std::vector<std::future<void>> futures;
capRefDangle(futures);
for(auto &fut: futures){
    fut.get();
}
```

输出...

```plain
i=32766
i=32766
i=32766
i=32766
i=32766
```

得到了更奇怪的结果，我本来想说如果是 C++ 就不会存在这种问题，对 dangling reference 一定会报错，然鹅根据实验结果和查询得知这是一种未定义行为，既能得到 go 中一样的结果，也可能输出奇怪的东西，不过总的来说我认为 go 这个锅就是让 capture by reference + GC 背。
