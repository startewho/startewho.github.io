
c++中对基类指针的问题，代码如下

```cpp
#include <iostream>
#include <string>

using namespace std;

class Base
{  
//有虚函数，因此是多态基类
public:
    virtual ~Base() {}
};

class Base2
{
    public:
    virtual ~Base2() {}
};

class Derived : public Base, public Base2 { };

int main()
{
    Base2* b2;
    Base* b;
    Derived d;
    Derived* pd;

    printf("d  :%p\n",&d);
    
    b2 = dynamic_cast <Base2*>(&d);
   
    printf("b2 :%p\n",&b2);
      
    b = dynamic_cast <Base*> (b2);
    printf("b  :%p\n",b);
    pd = dynamic_cast <Derived*> (b2);  //安全的转换
    printf("bd :%p\n",pd);
    printf("&bd:%p\n",&pd);
    return 0;
}
```
运行结果：

  d  :0x7ffcfa8f33c0
  b2 :0x7ffcfa8f33d0
  b  :0x7ffcfa8f33c0
  bd :0x7ffcfa8f33c0
  &bd:0x7ffcfa8f33b8

可以看到编译器进行了转换，保证了访问的正确