# 七、接口系统

不引入 `interface` / `impl X : Y` 关键字，直接使用标准 C++ 抽象基类 + 纯虚
函数表达"需要多态分发"的接口；`comp bool` 函数（第六节说明）只用于泛型约束场景，
不用于表达运行时接口。

## 定义接口：抽象基类

```cpp
class Writer {
public:
    virtual size_t write(const uint8_t* data, size_t size) = 0;
    virtual void flush() = 0;
    virtual ~Writer() = default;
};
```

## 实现接口：标准继承 + `override`

```cpp
class File : public Writer {
public:
    size_t write(const uint8_t* data, size_t size) override {
        // ...
    }

    void flush() override {
        // ...
    }
};
```

## 接口指针

```cpp
// 栈上/借用
void process(Writer& w) {
    w.write(cast<const uint8_t*>("Hello").value(), 5);
    w.flush();
}

// 堆上：标准 unique_ptr
unique_ptr<Writer> writer = make_unique<File>("out.txt");
```

## 接口与反射结合

```cpp
comp {
    // 查找当前模块内所有派生自 Writer 的类型
    for (auto T : types_deriving_from(^^Writer)) {
        register_writer(T);
    }
}
```
