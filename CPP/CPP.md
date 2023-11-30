# C++11模板类型推导规则
##### 在编译过程中编译器会用expr推断两种类型：一个是T，一个是arg。而这两种类型往往是不一样的，因为type通常会包含修饰符，比如_const_或者引用（type :T， T&,  T&&）
```cpp
	template<typename T>
	void fun(type arg);//模板参数为类型
	fun(expr);
	
```
1.  形参类型为T的普通值类型    
      + 忽略传入的expr实参的任何引用、const属性等任何属性后，作为T的推导类型   
      + 将T的推导类型代入Type，即可得到arg的推导类型
	```cpp
	template<typename T>
	void fun(T arg);
	```

    ```cpp
	//调用处：
	int a=0;
	int& b=a;
	int &&d = 0;
	const int& c=0;
	const int* const d=&a;
	fun(a);//T->int,arg->int
	fun(b);//T->int,arg->int
	fun(d);//T->int,arg->int
	fun(c);//T->int,arg->int
	** fun(d);//T->const int*,arg->const int* **
	fun(2);//T->int,arg->int	
	```
2. 形参类型 为T的左值引用或指针
	-   忽略传入的expr实参的任何引用属性作为T的推导类型
	-   将T的推导类型代入Type，即可得到arg的推导类型
	-   **不接受右值**
	-   若与组合类型有重叠，则忽略（cv）
	```cpp
	template<typename T>
	void fun(T& arg);
    ```
     
	```cpp
	//调用处：
	int a=0;
	int& b=a;
	const int c=0;
	const int& d=a;
	fun(a);//T->int, arg->int&
	fun(b);//T->int,arg->int&
	fun(c);//T->const int,arg->const int&
	fun(d);//T->const int,arg->const int&
	fun(2);//不接受右值！
	```
	当传入只是数组时，一般退化成指针，但是若是T&，则会推到为数组(可以获取大小)

	```cpp
		template<typename T, std::size_t N>
		constexpr std::size_t arraySize(T (&)[N]) noexcept{
			return N;
		}
		//在编译期间返回一个数组大小的常量值
		int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 };
		int mappedVals[arraySize(keyVals)];
	```

1. 形参类型为T的万能引用
	+ expr为左值时，将T推导为T&；当expr为右值时，将T推导为T（值类型）
	+ 将T的推导类型代入Type，即可得到arg的推导类型（引用折叠)
	+ 如果Type类型表达式中含有任何其他饰词（const等）、其他表达形式，都不算做万能引用，即此规则的第1条不再成立（按照“形参类型为左值引用的第1条规则来推导”）
	```cpp
	template<typename T>
	void fun(T&& arg);
	fun(expr);
	```
	注意是 expr 而不是 值的类型；
	```cpp
	int a=0, &&b = 4, &c = a;
	fun(a);//T->int&,arg->int& && -> 引用折叠 -> int&
	fun(2);//T->int,arg->int&&
	fun(b);//T->int&
	func(c）；//T->int& 
	//因为abc都是左值
	
	template<typename T>
	void fun1(const T&& arg);//不算万能引用！（按左值引用的规则来推导）
	template<typename T>
	void fun2(std::vector<T>&& arg);//不算万能引用！（按左值引用的规则来推导）
	template<typename T>
	class A{
	    void fun3(T&& arg);//不算万能引用！（按左值引用的规则来推导）
	};
	```


# 传入函数的形参都是左值，字符串是左值
```cpp
void fuc(int& param)//左值引用{
	cout << "lvalue reference" << endl;
}
void fuc(int&& param)//右值引用{
	//param 此时是左值
	cout << "rvalue reference" << endl;
}
    int a = 1;
    int &b = a;
    int &&c = 3;
    //要看表达式的类型去确定调用哪一个函数，虽然 c 是 int&& ， 但c是左值。
    //函数判断时，看的是（表达式左右）， 而不是（int，string。。。）
    PerfectFoward(a);
    PerfectFoward(b);
    PerfectFoward(c);//lva/ue
	PerfectFoward(100);//rvalue
```

# 引用折叠
+ 只有 T&& && 与 T&& 变成右值引用，其余只要有左值引用，则变成左值引用。





# 左值与右值

左：可以取地址，右：不可以地址。

## 拷贝构造函数
```cpp
    Stu(const Stu &s){
        std::cout << "copy" << std::endl;
    };
    Stu& operator= (const Stu &){
        std::cout << "copy assign" << std::endl;
        return *this;
    };
```

其中要有：
    const    因为必须传引用，但是右值不可赋值给左值引用，但（const 左值引用）是万能引用，可以用右值赋值，所以加const。
拷贝构造函数调用场景：
	（1）用类的一个对象去初始化另一个对象时
	（2）当函数的形参是类的对象时（也就是值传递时），如果是引用传递则不会调用
	（3）当函数的返回值是类的对象或引用时
	
## 移动构造函数（可以减少对象的复制）
```cpp
class A {
    public:
        A() {}
        A(const A&) { std::cout << "copy" << std::endl; }
        A& operator=(const A & a ) 
	        { std::cout  << "copy ass" << std::endl; return *this; }
        A(A&&) { std::cout << "move" << std::endl; }
        A& operator=(A && a) 
	        { std::cout << "move ass"  << std::endl; return *this; }
};
A getA( A a) {return a; }
A getAA(){
    A a;
    return getA(a);
}
int main() {
    A b = getAA();
    return 0;
}

```
若没有 移动构造函数上面就会有四次拷贝
	 （=b），（ret getAA），（getA（）），（ret getA）
若有则只有一次copy，三次move。


完美转发：函数模板可以将自己的参数“完美”地转发给内部调用的其它函数。所谓完美，即不仅能准确地转发参数的值，还能保证被转发参数的左、右值属性不变。

```cpp
Stu test(Stu&& a){
  std::cout << "&&" <<std::endl;
  return a;
}
Stu test(Stu& a){
  std::cout << "&" <<std::endl;
  return a;
}
template<typename T>
void tt(T &&a){
    // test(std::forward<T>(a));
    test(a);
}
  Stu a;
  tt(a);
  tt(Stu());
```
我们想当 tt传入左值调用 “&”， 右值调用 “&&”， 但是因为函数实参都是左值，所以都会调用 “&”，
并且会有copy construtor 调用。
此时可用完美转发实现，并且不用copy。

就是提高程序性能（A->B->C）不想再B中调用copy 。

		forward<T> （value）;
××若T是value的左值引用，则表达式为左引用，否则是右引用---
		
```cpp
	t = forward<int> (a); // t是int &&
	t = forward<int&> (a)// t是int &
```

如下定义不报错，但是若调用报错，因为传入值之间会转化
```cpp
void test1(int a){ }
void test1(int &&a){ }
void test1(int &a){ }
```



# decltype
## 推导规则 T = decltype(e)
+   e未加括号并且是标识符表达式（不含+，-....保留字）或者 是未加括号的类成员访问则T =
e类型
+ 若e是函数（仿）调用，则 T = 函数返回值
+ 若e是左值，则T = e的左引用
+ 若e是右值，则T = e （a+b，i++）
+ 若e是将亡值，则T = e的右引用（函数返回）
+ 否则 T = e的类型
若有CV限定则，一般可以同步e的限定，但
```cpp
struct A{int a;};
const A* a = new A();
decltype(a->x) // int
decltype((a->x)) // const int &
```

# auto推导规则
+ 若auto中没有(&，× )则去掉 CV
+ auto 会去掉 &
+ 类似万能引用（模板 T&&），会将左变成左引用，右变成值
+ 数组，函数变成指针  
+ auto类型推导通常和模板类型推导相同，但是auto类型推导假定花括号初始化代表std::initializer_list，而模板类型推导不这样做
```cpp
const int i = 5;
auto j = i; // j -- int
auto &j = i; // j -- const int

int j1 = 1;
int &j2 = j1;
auto j = j2// j -- int != int &

int i = 5;
auto && j = i; // j -- int &
auto && j = 5; // j -- int
```

---

# 列表初始化

```cpp
struct C {  
	C(std::initializer_list<std::string> a)  {  
		//is pointer
		for (const std::string* item = a.begin(); item != a.end();  ++item) {  
			std::cout << item << " ";  
		}  
	}  
};  
C c{ "hello", "c++", "world" };
```
 
隐式缩窄转换问题
    从浮点类型转换整数类型
    从long double转换到double或float，或从double转换  
到float.
    从整数类型或非强枚举类型转换到浮点类型
    从整数类型或非强枚举类型转换到不能代表所有原始类型值的  
整数类型，除非源是一个常量表达式，其值在转换之后能够适合目标  
类型

列表初始化的优先级
```cpp
std::vector<int> x1(5, 5);  
std::vector<int> x2{ 5, 5 };
//如果有一  个类同时拥有满足列表初始化的构造函数，且其中一个是以  
//std::initializer_list为参数，那么编译器将优先以 std::initializer_ list为参数构造函数
```




---


声明任何构造函数都会抑制默认构造函数的添加。
```cpp
class City {  
	string name;
	public:  
		City(const char *n) : name(n) {}  
};  
int main()  {  
	City a("wuhan");  
	City b; // 编译失败，自定义构造函数抑制了默认构造函数  
}
```

---


为了让联合类型更加实用，在C++11标准中解除了大部分限制，联 合类型的成员可以是除了引用类型外的所有类型。
```cpp
//在C++11中如果有联合类型中存在非平凡类型，那么这个联合类型的特殊成员函数将被隐式删除
//我们必须自己至少提供联合类型的构造和析构函数
union U  {  
U() {} // 存在非平凡类型成员，必须提供构造函数  
~U() {} // 存在非平凡类型成员，必须提供析构函数  
int x1;  
float x2;  
std::string x3;  
std::vector<int> x4;  
}; 
U u;  
new(&u.x3) std::string("hello world");  
std::cout << u.x3 << std::endl;  
u.x3.~basic_string();  
//在使用完对象后手动调用对象的析构函数。通过这样的方法保证了联合类型使用的灵活性和正确性。
```


---

## 委托构造函数
```cpp
class X  
{  
public:  
X() : X(0, 0.) {}  
X(int a) : X(a, 0.) {}  
X(double b) : X(0, b) {}  
X(int a, double b) : a_(a), b_(b) { CommonInit(); }  
private:  
void CommonInit() {}  
int a_;  
double b_;  
};
//如果一个构造函数为委托构造函数，那么其初始化列表里就不能对数据成员和基类进行初始化
X() : a_(0), b_(0) { CommonInit(); }  
X(int a) : X(), a_(a) {} // 编译错误，委托构造函数不能在初始化列表初始化成员变量
```

## 委托模板构造函数
```cpp
class X {  
	template<class T> 
		X(T first, T last) : l_(first, last) { }  
	std::list<int> l_;  
	public:
	X(std::vector<short>& v) : X(v.begin(), v.end()) { }  
	X(std::deque<int>& v) : X(v.begin(), v.end()) { } 
};  
```
---


#  基于范围循环的陷阱

基于范围的for循环非常方便，甚至可以遍历临时对象。但是要注意的是，如果要遍历临时对象的话，需要遍历的临时对象必须是 **右值表达式** ，而且也要注意表达式中间产生的其他临时对象是在循环开始前就会被销毁的，只有表达式返回的最后的临时对象才会被“存”起来。
```cpp
struct MyClass{
    string text = "MyClass";
    string& getText(){
        return text;
    }
};
    for (auto ch : MyClass().text){
        cout << ch;
    }
    for (auto ch : MyClass().getText()){   //错误
        cout << ch;
    }
	string &t = MyClass().getText();
	//非(const左值引用)和(右值引用)不会延长临时对象(MyClass())生命周期,所以t是野引用。
```

原始的例子中，range_expression是 "MyClass().text"，MyClass()是临时对象，同时 "MyClass()" 这个表达式是右值。所以，"MyClass().text" 这个表达式也是右值，"MyClass().text" 这个对象是临时对象中的一部分。所以，在 "auto && __range = range_expression;" 这个语句中，auto会被推导为 "std::string"。初始化右值引用为临时对象的一部分时，可以延长整个临时对象的生存期，在引用被销毁时临时对象才会被销毁。所以for循环可以正常执行。

但是在修改过后，range_expression是 "MyClass().getText()"。同样地，MyClass()是临时对象，"MyClass()" 这个表达式是右值。但是 "getText()" 的返回类型为 "string&"，所以，"MyClass().getText()" 这个表达式是左值。所以，在 "auto && __range = range_expression;" 这个语句中，auto会被推导为 "string &"，语句等价于 "string & __range = range_expression;" 。虽然"MyClass().getText()" 这个对象是临时对象中的一部分，但是在初始化非const的左值引用时，不会延长临时对象的生存期，所以在这个初始化语句结束的同时MyClass()这个临时对象就被销毁了，__range成为了野引用，所以后面的循环语句可能会出现内存错误。


---

# noexcept
noexcept是一个与异常相关的关键字，它既是一个说明符，也是一个运算符.

C++标准委员会又赋予了noexcept作为运算符的特性。noexcept运算符接受表达式参数并返回true或false。因为该过程
是在编译阶段进行，所以表达式本身并不会被执行。而表达式的结果取决于编译器是否在表达式中找到潜在异常。
noexcept运算符能够准确地判断 **函数是否有声明不会抛出异常**

**noexcept对于移动语义，swap，内存释放函数和析构函数非常有用**
```cpp
int foo() noexcept{
	return 42;
}
int foo1(){
	return 42;
}
int foo2() throw(){
	//如果函数不会抛出任何异常，那么`( )`中什么也不写
	return 42;
}
noexcept(foo1()) //true
noexcept(foo1()) //false
noexcept(foo2()) //true
```    
    
实现若T满足move constructor不抛出异常调用move constructor， 抛异常调用copy constructor
```CPP
template<typename T>
void
__swap(T &a, T &b, std::integral_constant<bool, true>)
noexcept
{
    T temp(std::move(a));
    a = std::move(b);
    b = std::move(temp);
}

template<typename T>
void
__swap(T &a, T &b, std::integral_constant<bool, false>)
{
    T temp(a);
    a = b;
    b = temp;
}

typedef integral_constant<bool, true> true_type;
typedef integral_constant<bool, false> false_type;

template<typename T>
void
swap(T &a, T &b)
noexcept( 
    noexcept(
        __swap(a, b, std::integral_constant<bool, 
            noexcept(T(std::move(a))) && noexcept(a.operator=(std::move(b)))> ()
        ) 
    )
)
{
    __swap(a, b,
        std::integral_constant<bool,
            noexcept(T(std::move(a))) && noexcept(a.operator=(std::move(b)))> ()
    );
}
// 外层是说明符， 内层是运算符(说明符只接受常量， 所以不可以省略上述内层noexcept)
```
### 
```cpp
#define APP_VERSION0 0x234
struct GetAppVersion : public std::integral_constant<unsigned int, APP_VERSION> {};
auto x = GetAppVersion();
```


## 初始化
花括号初始化又叫统一初始化，在C++中这三种方式都被看做是初始化表达式，但是只有花括号任何地方都能被使用。
括号表达式还有一个少见的特性，即它不允许内置类型间隐式的变窄转换（narrowing conversion）

如果有一个或者多个构造函数的声明包含一个std::initializer_list形参，那么使用花括号初始化语法的调用更倾向于选择带std::initializer_list的那个构造函数
但是空参会调用默认构造函数
```cpp
class Widget { 
public:  
    Widget(int i, bool b);      //同上
    Widget(int i, double d);    //同上
    Widget(std::initializer_list<long double> il);      //新添加的
    …
}; 
Widget w1(10, true);    //调用第一个构造函数
Widget w2{10, true};    //使用花括号初始化，但是现在
                        //调用带std::initializer_list的构造函数
                        //(10 和 true 转化为long double)

Widget w3(10, 5.0);     //调用第二个构造函数 
Widget w4{10, 5.0};     //使用花括号初始化，但是现在
                        //调用带std::initializer_list的构造函数
                        //(10 和 5.0 转化为long double)
Widget w6{w4};                  //使用花括号，调用std::initializer_list构造
                                //函数（w4转换为float，float转换为double）
Widget w8{std::move(w4)};       //使用花括号，调用std::initializer_list构造
                                //函数（与w6相同原因

```

## enum
限域enum避免命名空间污染而且不接受荒谬的隐式类型转换
enum class 可指定底层类型，且可以限定作用域
```cpp
enum class Status: std::uint32_t; 

//用于
enum class UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo;                         //同之前一样
auto val =
    std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>
        (uInfo);
//可以用函数代替强转，moden c++

```

#引用限定符
```cpp
    void doWork() &;    //只有*this为左值的时候才能被调用
    void doWork() &&;   //只有*this为右值的时候才能被调用

	//应用
class Widget {
public:
    using DataType = std::vector<double>;
    DataType& data() &              //对于左值Widgets,
    { return values; }              //返回左值
    DataType data() &&              //对于右值Widgets,
    { return std::move(values); }   //返回右值
private:
    DataType values;
};
auto vals1 = w.data();              //调用左值重载版本的Widget::data，
                                   //拷贝构造vals1
auto vals2 = makeWidget().data();   //调用右值重载版本的Widget::data, 
                                    //移动构造vals2

```

## const ， constexpr
constexpt 在编译期可知， const 不行

constexpr函数：
- constexpr函数可以用于需求编译期常量的上下文。如果你传给constexpr函数的实参在编译期可知，那么结果将在编译期计算。如果实参的值在编译期不知道，你的代码就会被拒绝。
- 当一个constexpr函数被一个或者多个编译期不可知值调用时，它就像普通函数一样，运行时计算它的结果。这意味着你不需要两个函数，一个用于编译期计算，一个用于运行时计算。constexpr全做了。

**也即，constexpr函数根据实参类型，产出不通期变量： 实参是编译器常量产出编译期常量，否则产出运行期变量。




## move， forward

都是强制转换
forward = static_cast<T&&>(); -- 若T是引用(int&),则转为引用(左值)

```cpp
class Annotation {
public:
    explicit Annotation(const std::string text)
    ：value(std::move(text))    //“移动”text到value里；这段代码执行起来
    { … }                       //并不是看起来那样
    
    …

private:
    std::string value;
};
//不一定调用，移动构造，因为有const， 所以转换为 拷贝构造

class string {                  //std::string事实上是
public:                         //std::basic_string<char>的类型别名
    …
    string(const string& rhs);  //拷贝构造函数
    string(string&& rhs);       //移动构造函数
    …
};

```
- 不要在你希望能移动对象的时候，声明他们为const。对const对象的移动请求会悄无声息的被转化为拷贝操作。
- std::move不仅不移动任何东西，而且它也不保证它执行转换的对象可以被移动。关于std::move，你能确保的唯一一件事就是将得到一个右值。

- std::move和std::forward在运行期什么也不做。
- **对右值引用使用std::move，对通用引用使用std::forward**
- **使用通用引用的函数在C++中是最贪婪的函数。它们几乎可以精确匹配任何类型的实参,因此避免在通用引用上重载**


用于提升效率
```cpp
template<typename T>
void logAndAdd(T&& name){
    names.emplace(std::forward<T>(name));
}

std::string petName("Darla");
logAndAdd(petName);                     //拷贝左值到multiset
logAndAdd(std::string("Persephone"));	//移动右值而不是拷贝它
logAndAdd("Patty Dog");                 //在multiset直接创建std::string
                                        //而不是拷贝一个临时std::string

```

一种 同时使用重载与通用引用的解决方法 -- tag dispatch
```cpp
template<typename T>
void logAndAdd(T&& name){
    logAndAddImpl(
        std::forward<T>(name),
        std::is_integral<typename std::remove_reference<T>::type>()
    );
}

template<typename T>                            //非整型实参：添加到全局数据结构中
void logAndAddImpl(T&& name, std::false_type){
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}
std::string nameFromIdx(int idx);
void logAndAddImpl(int idx, std::true_type) {
  logAndAdd(nameFromIdx(idx)); 
}

```
二, 使用 std::enable_if 来选择性禁止 限定模板的使用：
```cpp
class Person {
public:
    template<
        typename T,
        typename = typename std::enable_if<
                       !std::is_base_of<Person,typename std::decay<T>::type>::value
						&&
					   !std::is_integral<std::remove_reference_t<T>>::value //可以组合使用
                   >::type
    >
    explicit Person(T&& n);
    …
};

```

## 完美转发失败
- 当模板类型推导失败或者推导出错误类型，完美转发会失败。
- 导致完美转发失败的实参种类有花括号初始化，作为空指针的0或者NULL，仅有声明的整型static const数据成员，模板和重载函数的名字，位域。


# 智能指针
unique_ptr
- 工厂方法一般用 unique_ptr
- Pimpl惯用法是std::unique_ptr的最常见的使用情况之一
    - 使用时要加上,否则无法通过编译
    ```cpp
        Widget::~Widget() = default; 
    ```



- share_ptr创建控制块的情况：
    - std::make_shared
    - 当从独占指针（即std::unique_ptr或者std::auto_ptr）上构造出std::shared_ptr
    - 当从原始指针上构造出std::shared_ptr时会创建控制块
  **因此若从上述来源中，构建了两个share_ptr则可能会析构两次**

  - std::shared_ptr不能处理数组。和std::unique_ptr不同的是，std::shared_ptr的API设计之初就是针对单个对象的，没有办法std::shared_ptr<T[]>


- std::weak_ptr不能解引用，也不能测试是否为空值，std::weak_ptr通常从std::shared_ptr上创建
- 用std::weak_ptr替代可能会悬空的std::shared_ptr。
- std::weak_ptr的潜在使用场景包括：缓存、观察者列表、打破std::shared_ptr环状结构。

```cpp
processWidget(std::shared_ptr<Widget>(new Widget),  //潜在的资源泄漏！
              computePriority());
执行顺序 new Widget - com... -- shared_ptr
在 com。。中异常则 widget泄露
```

# 智能指针 -- insert share_ptr注意insert是构建新值， 引用计数为0，不要用insert。

```cpp
    std::vector<std::shared_ptr<int>> ti = {std::make_shared<int>(5), std::make_shared<int>(5),std::make_shared<int>(5),std::make_shared<int>(5),};
    ti.insert(end(ti), begin(ti), end(ti));
    for(int i=0; i<ti.size(); ++i){
        std::cout << ti[i].use_count() << " ";
    }
```

# lower_buond返回第一个 cmp 为false的it
```cpp
  //不变的是第二个值 key
  auto iter = std::lower_bound(array_, array_ + GetSize(), key,
                               [&comparator](const auto &pair1, auto key) { return comparator(pair1.first, key) < 0; });
```