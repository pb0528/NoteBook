# C++ Knowledge



## pointer(指针)

```C++
int main()
{
    int a[4][2] = { 7,6,5,4,3,2,1,0 };
    int* p1 = a[2];
    int m = 10;
    
    
    int* p2 = &a[0][0]; //*p2 = 7;
    for (int i = 0; i < 8; i++)
    {
        cout << p2++ << endl;
    }
    int* j = (int*) (&a + 1);//&a 相当于 int (*)[4][2];

    int(*p3)[2] = &a[1]; //*p3 = 5;

    cout << *(j - 1) << endl;

    //cout <<  << endl;
}
```



###  将数值上调至x的倍数函数

```c++
size_t ROUND_UP(size_t bytes){
	return (bytes + x - 1) & ~(x - 1);
}
```



### C++明确的求值顺序

![image-20210911210709249](C:\Users\PB\Desktop\Record Note\C++\PIC\C++_Expression.png)





### 求值顺序、优先级、结合律

​	运算对象的求值顺序与优先级和结合律无关，在一条形如f() + g() * h() + j()的表达式中：

* 优先级规定，g()的返回值和h()的返回值相乘
* 结合律规定，f()的返回值先与g() 和 h()乘积相加，所得结果再与j()返回值相加
* 对于这些函数的调用顺序没有明确的规定

如果f、g、h和j是无关函数，它们既不会改变同一对象的状态也不执行IO任务，那么函数的调用顺序不受限制。反之，如果其中某几个函数影响同一对象，则它是一条错误的表达式，将产生未定义的行为。

建议：

1. 拿不准的时候最好用括号来强制让表达式的组合关系符合程序逻辑要求
2. 如果改变了某个运算对象的值，在表达式的其它地方不要再使用这个运算对象

例外：*++iter中，递增运算符改变iter的值，iter的值又是解引用运算符运算对象



. 运算符 优先级 高于 *（解引用）运算符



![image-20210912114348700](C:\Users\PB\Desktop\Record Note\C++\PIC\image-20210912114348700.png)



 ![image-20210918221403625](C:\Users\PB\Desktop\Record Note\C++\PIC\image-20210918221403625.png)

**std::thread** **std::ifstream** **std::unique_str** 都是移动的，但是不可拷贝



