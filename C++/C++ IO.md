# C++ I/O

![](C:\Users\PB\Desktop\Record Note\C++\PIC\Istream.PNG)

输入流可以读取并解释字符串序列。标准输入流对象*cin*是此类型的对象之一。

## 关键成员函数

**Formatted input**:

operator>>        从有序输入流中提取对象

**Unformatted input**:

gcount             获取字符数量

get				    获取字符

getline			 获取行字符

ignore              提取并丢弃字符串

peek                 获取下一个字符

read				  读取数据块

**Positioning**:

tellg                    从输入序列中

seekg                 设置输入序列输入位置



![](C:\Users\PB\Desktop\Record Note\C++\PIC\Ostream.PNG)



## C语言IO

获取文件大小 

相关函数：ftell、fseek、rewind

```c
 	FILE *file;
    char buffer[] = "hello world";
    file = fopen("myfile.bin", "rb");
    //fwrite(buffer, sizeof(char), sizeof(buffer), file);
    if(file == NULL) 
    {
        fputs("File error!\n", stderr);
        exit(-1);
    }

    //obtain file sizeupt
    fseek(file, 0, SEEK_END);
    int size = ftell(file);
    rewind(file);

    printf("the size of file is %d\n", size);
    fclose(file);
    return 0;
```



## 基本输入输出

此处主要显式主要的输入输出 

**getline**

```c++
function 

std::getline(string)

1. istream& getline(istream& is, string& str, char delim);

   istream& getline(istream&& is, string& str, char delim);

2. istream& getline(istream& is, string& str);

   istream& getline(istream&& is, string& str);
```

功能描述：从is流中提取字符，并存储进str中，遇到分隔符delim截至（或者换行符'\n')

如果is流是文件流，遇到文件截至符号或者输入操作出现错误都会出现截至

如果， delimiter 分隔符出现，它会被提取出来并且被丢弃。值得注意的是，在调用getline函数之前str中的所有字符都会被新提取出来的字符所取代。如果被提取的字符加入进字符类似push_back()

```c++
#include<iostream>
#include <string>

using namespace std;

int main()
{
	string str("hello world");
	cout << "before call getline: " << str.size() << " str contend: " << str << endl; //此时str "hello world"

	getline(cin, str); //键入 “ni hao"

	cout << "after call getline: " << str.size() << "str contend: " << str << endl; //此时str "ni hao"
	return 0;
}
```

以上代码可以发现 在调用getline()函数之后，可以检测到str之前的内容被完全抛弃，不管能否超过str原来的大小



**cin.get**

```c++
function

std::istream::get
	single character int get();
					 istream& get(char& c);
			c-string(2) istream& get(char* s, streamsize n);
						istream& get(char* s, streamsize n, char delim);
		stream buffer	istream& get(steambuf& sb);
						istream& get(streambuf& sb, char delim);
```

描述：从流中提取字符串， 作为非格式化输入

1. single character(单个字符串):

   从流中提取字符串

2. 字符串类

   从流中提取字符串并且存储于字符串中直至 提取到n-1个字符或者碰到分隔符；*delimiter character*可能是分隔符或者换行符其中之一。

3. 字符串缓冲

   与c类型字符串类似，但是取消了字符数量的限制。

测试代码

```c++
//测试cin.get 函数
char c = 'A'; char b = 'B';
cout << "before call cin.get： " << c  << " " << b << endl; //输入 c[enter]
cin.get(c);
cin.get(b);
cout << "after call cin.get: " << c << " " << b << endl; //输出 c \n
```

**说明：cin.get()并不会丢弃换行符**

**>>** ：格式化输入流操作

测试代码：

```c++
//测试cin >> 运算符
	string str;
	cout << "before call \'>>\' " << str.size() << " str contend: " << str << endl; // str 为空
	int a = 0;
	cin >> a; //输入 14 加上换行符'\n'
	getline(cin, str); //此时观察到str 仍然为空
	cout << "after call \'>>\' " << str.size() << " str contend: " << str << endl; //str 仍然为空
```

**测试代码说明cin 标准输入输出会把'\n' 截断并且丢弃**