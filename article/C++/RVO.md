## C++ 返回值优化（RVO）

> 对于C++中函数返回临时对象，通常观点是会产生临时对象，有额外开销，这是真的吗？

首先构造一个类用于测试，测试环境信息：`Linux`, `GCC 9.3.0`

```c++
class RvoClass {
public:
    explicit RvoClass(int a) : a(a) { cout << "constructor" << endl; }

    RvoClass(const RvoClass& rvo)
    {
        cout << "copy constructor" << endl;
        a = rvo.a;
    }

    RvoClass(RvoClass&& rvo) noexcept
    {
        cout << "move constructor" << endl;
        a = rvo.a;
    }

    RvoClass& operator=(const RvoClass& rvo)
    {
        cout << "copy assignment" << endl;
        a = rvo.a;
        return *this;
    }

    RvoClass& operator=(RvoClass&& rvo) noexcept
    {
        cout << "move assignment" << endl;
        a = rvo.a;
        return *this;
    }

    ~RvoClass() { cout << "destructor" << endl; }

private:
    int a;
};
```

## 测试一：直接返回临时对象

```c++
RvoClass func2()
{
    RvoClass rvo(10);
    return rvo;
}

int main(int argc, char** argv)
{
    {
        RvoClass r2 = func2();
    }
    return 0;
}
```

测试结果如下：

```txt
constructor
destructor
```

出乎意料之外，并没有额外开销，那其中到底发生了什么？让我们看看汇编。

```bash
objdump -M intel -S -C -d rvo
```

```assembly
RvoClass func2()
{
    1270:       f3 0f 1e fa             endbr64 
    1274:       55                      push   rbp
    1275:       48 89 e5                mov    rbp,rsp
    1278:       48 83 ec 20             sub    rsp,0x20
    127c:       48 89 7d e8             mov    QWORD PTR [rbp-0x18],rdi
    1280:       64 48 8b 04 25 28 00    mov    rax,QWORD PTR fs:0x28
    1287:       00 00 
    1289:       48 89 45 f8             mov    QWORD PTR [rbp-0x8],rax
    128d:       31 c0                   xor    eax,eax
    RvoClass rvo(10);
    128f:       48 8b 45 e8             mov    rax,QWORD PTR [rbp-0x18]
    1293:       be 0a 00 00 00          mov    esi,0xa
    1298:       48 89 c7                mov    rdi,rax
    129b:       e8 fe 00 00 00          call   139e <RvoClass::RvoClass(int)>
    return rvo;
    12a0:       90                      nop
}
    12a1:       48 8b 45 f8             mov    rax,QWORD PTR [rbp-0x8]
    12a5:       64 48 33 04 25 28 00    xor    rax,QWORD PTR fs:0x28
    12ac:       00 00 
    12ae:       74 05                   je     12b5 <func2()+0x45>
    12b0:       e8 0b fe ff ff          call   10c0 <__stack_chk_fail@plt>
    12b5:       48 8b 45 e8             mov    rax,QWORD PTR [rbp-0x18]
    12b9:       c9                      leave  
    12ba:       c3                      ret    
    
    

00000000000012e2 <main>:

int main(int argc, char** argv)
{
    12e2:       f3 0f 1e fa             endbr64 
    12e6:       55                      push   rbp
    12e7:       48 89 e5                mov    rbp,rsp
    12ea:       48 83 ec 20             sub    rsp,0x20
    12ee:       89 7d ec                mov    DWORD PTR [rbp-0x14],edi
    12f1:       48 89 75 e0             mov    QWORD PTR [rbp-0x20],rsi
    12f5:       64 48 8b 04 25 28 00    mov    rax,QWORD PTR fs:0x28
    12fc:       00 00 
    12fe:       48 89 45 f8             mov    QWORD PTR [rbp-0x8],rax
    1302:       31 c0                   xor    eax,eax
    {
        RvoClass r2 = func2();
    1304:       48 8d 45 f4             lea    rax,[rbp-0xc]
    1308:       48 89 c7                mov    rdi,rax
    130b:       e8 60 ff ff ff          call   1270 <func2()>
    1310:       48 8d 45 f4             lea    rax,[rbp-0xc]
    1314:       48 89 c7                mov    rdi,rax
    1317:       e8 16 01 00 00          call   1432 <RvoClass::~RvoClass()>
    }
    return 0;
    131c:       b8 00 00 00 00          mov    eax,0x0
}
    1321:       48 8b 55 f8             mov    rdx,QWORD PTR [rbp-0x8]
    1325:       64 48 33 14 25 28 00    xor    rdx,QWORD PTR fs:0x28
    132c:       00 00 
    132e:       74 05                   je     1335 <main+0x53>
    1330:       e8 8b fd ff ff          call   10c0 <__stack_chk_fail@plt>
    1335:       c9                      leave  
    1336:       c3                      ret    
```

通过汇编可以看出在`func2`函数中创建了临时对象，然后将这个临时对象的地址返回。

这是因为编译器做了返回值优化`RVO`，从C++11开始支持此特性。

下面直接返回临时对象，也只执行一次构造。

```cpp
RvoClass func3()
{
    return RvoClass(8);
}
```

## 测试二：运行时决定返回临时对象

```cpp
RvoClass func1(int a)
{
    if (a > 10) {
        RvoClass rvo(a - 10);
        return rvo;
    } else {
        RvoClass rvo(a);
        return rvo;
    }
}

int main(int argc, char** argv)
{
    {
        RvoClass r1 = func1(11);
    }
    return 0;
}
```

测试结果如下：

```txt
constructor
move constructor
destructor
destructor
```

从测试结果看，编译器并没有做`RVO`优化。但由于我们定义了`移动构造函数`，编译器做了`拷贝优化`，如果注释掉`移动构造函数`，你会发现结果是调用了`拷贝构造函数`。使用`移动构造函数`可以减少调用`拷贝构造函数`所带来的开销。

## 结论

首先如果明确要求避免拷贝构造，那应该将函数外定义，通过引用传参(或指针传参)的方式来避免额外的拷贝操作。

如下方式：

```cpp
void func(RvoClass& rvo)
{
    ...
}

RvoClass r(0);
func(r);
```

对于运行期才能决定如何返回临时变量的函数，不会有`RVO`，但可能有`拷贝优化`。

对于编译期就能知道如何返回临时变量的函数，会有`RVO`(其实需要看编译器是否支持)。

对于函数返回值的类型做了转换的，不会有`RVO`。

对于函数返回`右值引用`，不会有`RVO`，但可能有`拷贝优化`;

