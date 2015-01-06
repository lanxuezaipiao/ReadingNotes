##导读##
最近在补看《C++ Primer Plus》第六版，这的确是本好书，其中关于智能指针的章节解析的非常清晰，一解我以前的多处困惑。C++面试过程中，很多面试官都喜欢问智能指针相关的问题，比如你知道哪些智能指针？shared_ptr的设计原理是什么？如果让你自己设计一个智能指针，你如何完成？等等……。而且在看开源的C++项目时，也能随处看到智能指针的影子。这说明智能指针不仅是面试官爱问的题材，更是非常有实用价值。

下面是我在看智能指针时所做的笔记，希望能够解决你对智能指针的一些困扰。

##目录##
1. 智能指针背后的设计思想
2. C++智能指针简单介绍
3. 为什么摒弃auto_ptr？
4. unique_ptr为何优于auto_ptr？
5. 如何选择智能指针？

##正文##
####1. 智能指针背后的设计思想####
我们先来看一个简单的例子：
```cpp
void remodel(std::string & str)
{
    std::string * ps = new std::string(str);
    ...
    if (weird_thing())
        throw exception();
    str = *ps; 
    delete ps;
    return;
}
```
当出现异常时（weird_thing()返回true），delete将不被执行，因此将导致内存泄露。
如何避免这种问题？有人会说，这还不简单，直接在<code>throw exception();</code>之前加上<code>delete ps;</code>不就行了。是的，你本应如此，问题是很多人都会忘记在适当的地方加上delete语句（连上述代码中最后的那句delete语句也会有很多人忘记吧），如果你要对一个庞大的工程进行review，看是否有这种潜在的内存泄露问题，那就是一场灾难！
这时我们会想：当remodel这样的函数终止（不管是正常终止，还是由于出现了异常而终止），本地变量都将自动从栈内存中删除—因此指针ps占据的内存将被释放，**如果ps指向的内存也被自动释放，那该有多好啊。**
我们知道析构函数有这个功能。如果ps有一个析构函数，该析构函数将在ps过期时自动释放它指向的内存。但ps的问题在于，它只是一个常规指针，不是有析构凼数的类对象指针。如果它指向的是对象，则可以在对象过期时，让它的析构函数删除指向的内存。

这正是 auto_ptr、unique_ptr和shared_ptr这几个智能指针背后的设计思想。我简单的总结下就是：**将基本类型指针封装为类对象指针（这个类肯定是个模板，以适应不同基本类型的需求），并在析构函数里编写delete语句删除指针指向的内存空间。**

因此，要转换remodel()函数，应按下面3个步骤进行： 
-  包含头义件memory（智能指针所在的头文件）；
- 将指向string的指针替换为指向string的智能指针对象；
- 删除delete语句。

下面是使用auto_ptr修改该函数的结果：
```cpp
# include <memory>
void remodel (std::string & str)
{
    std::auto_ptr<std::string> ps (new std::string(str))；
    ...
    if (weird_thing ())
        throw exception()； 
    str = *ps； 
    // delete ps； NO LONGER NEEDED
    return;
}
```

####2. C++智能指针简单介绍####
STL一共给我们提供了四种智能指针：auto_ptr、unique_ptr、shared_ptr和weak_ptr（本文章暂不讨论）。
模板auto_ptr是C++98提供的解决方案，C+11已将将其摒弃，并提供了另外两种解决方案。然而，虽然auto_ptr被摒弃，但它已使用了好多年：同时，如果您的编译器不支持其他两种解决力案，auto_ptr将是唯一的选择。

**使用注意点**
- 所有的智能指针类都有一个explicit构造函数，以指针作为参数。比如auto_ptr的类模板原型为：
```cppp
templet<class T>
class auto_ptr {
    explicit auto_ptr(X* p = 0) ; 
    ...
};
```
因此不能自动将指针转换为智能指针对象，必须显式调用：
```cpp
shared_ptr<double> pd; 
double *p_reg = new double;
pd = p_reg;                               // not allowed (implicit conversion)
pd = shared_ptr<double>(p_reg);           // allowed (explicit conversion)
shared_ptr<double> pshared = p_reg;       // not allowed (implicit conversion)
shared_ptr<double> pshared(p_reg);        // allowed (explicit conversion)
```
- 对全部三种智能指针都应避免的一点：
```cpp
string vacation("I wandered lonely as a cloud.");
shared_ptr<string> pvac(&vacation);   // No
```
pvac过期时，程序将把delete运算符用于非堆内存，这是错误的。

**使用举例**
```cpp
#include <iostream>
#include <string>
#include <memory>

class report
{
private:
    std::string str;
public:
 report(const std::string s) : str(s) {
  std::cout << "Object created.\n";
 }
 ~report() {
  std::cout << "Object deleted.\n";
 }
 void comment() const {
  std::cout << str << "\n";
 }
};

int main() {
 {
  std::auto_ptr<report> ps(new report("using auto ptr"));
  ps->comment();
 }

 {
  std::shared_ptr<report> ps(new report("using shared ptr"));
  ps->comment();
 }

 {
  std::unique_ptr<report> ps(new report("using unique ptr"));
  ps->comment();
 }
 return 0;
}
```

####3. 为什么摒弃auto_ptr？####
先来看下面的赋值语句:
```cpp
auto_ptr< string> ps (new string ("I reigned lonely as a cloud.”）;
auto_ptr<string> vocation; 
vocaticn = ps;
```
上述赋值语句将完成什么工作呢？如果ps和vocation是常规指针，则两个指针将指向同一个string对象。这是不能接受的，因为程序将试图删除同一个对象两次——一次是ps过期时，另一次是vocation过期时。要避免这种问题，方法有多种：
- 定义陚值运算符，使之执行深复制。这样两个指针将指向不同的对象，其中的一个对象是另一个对象的副本，缺点是浪费空间，所以智能指针都未采用此方案。
- 建立所有权（ownership）概念。对于特定的对象，只能有一个智能指针可拥有，这样只有拥有对象的智能指针的构造函数会删除该对象。然后让赋值操作转让所有权。这就是用于auto_ptr和uniqiie_ptr 的策略，但unique_ptr的策略更严格。
- 创建智能更高的指针，跟踪引用特定对象的智能指针数。这称为引用计数。例如，赋值时，计数将加1，而指针过期时，计数将减1,。当减为0时才调用delete。这是shared_ptr采用的策略。

当然，同样的策略也适用于复制构造函数。
每种方法都有其用途，但为何说要摒弃auto_ptr呢？
下面举个例子来说明。
```cpp
#include <iostream>
#include <string>
#include <memory>
using namespace std;

int main() {
  auto_ptr<string> films[5] =
 {
  auto_ptr<string> (new string("Fowl Balls")),
  auto_ptr<string> (new string("Duck Walks")),
  auto_ptr<string> (new string("Chicken Runs")),
  auto_ptr<string> (new string("Turkey Errors")),
  auto_ptr<string> (new string("Goose Eggs"))
 };
 auto_ptr<string> pwin;
 pwin = films[2]; // films[2] loses ownership. 将所有权从films[2]转让给pwin，此时films[2]不再引用该字符串从而变成空指针

 cout << "The nominees for best avian baseballl film are\n";
 for(int i = 0; i < 5; ++i)
  cout << *films[i] << endl;
 cout << "The winner is " << *pwin << endl;
 cin.get();

 return 0;
}
```
运行下发现程序崩溃了，原因在上面注释已经说的很清楚，films[2]已经是空指针了，下面输出访问空指针当然会崩溃了。但这里如果把auto_ptr换成shared_ptr或unique_ptr后，程序就不会崩溃，原因如下：
- 使用shared_ptr时运行正常，因为shared_ptr采用引用计数，pwin和films[2]都指向同一块内存，在释放空间时因为事先要判断引用计数值的大小因此不会出现多次删除一个对象的错误。
- 使用unique_ptr时编译出错，与auto_ptr一样，unique_ptr也采用所有权模型，但在使用unique_ptr时，程序不会等到运行阶段崩溃，而在编译器因下述代码行出现错误：
```cpp
unique_ptr<string> pwin;
pwin = films[2]; // films[2] loses ownership.
```
指导你发现潜在的内存错误。

这就是为何要摒弃auto_ptr的原因，一句话总结就是：**避免潜在的内存崩溃问题。**

####4. unique_ptr为何优于auto_ptr？####
可能大家认为前面的例子已经说明了unique_ptr为何优于auto_ptr，也就是安全问题，下面再叙述的清晰一点。
请看下面的语句:
```cpp
auto_ptr<string> p1(new string ("auto") ； //#1
auto_ptr<string> p2;                       //#2
p2 = p1;                                   //#3
```
在语句#3中，p2接管string对象的所有权后，p1的所有权将被剥夺。前面说过，这是好事，可防止p1和p2的析构函数试图刪同—个对象；
但如果程序随后试图使用p1，这将是件坏事，因为p1不再指向有效的数据。

下面来看使用unique_ptr的情况： 
```cpp
unique_ptr<string> p3 (new string ("auto");   //#4
unique_ptr<string> p4；                       //#5
p4 = p3;                                      //#6
```
编译器认为语句#6非法，避免了p3不再指向有效数据的问题。因此，unique_ptr比auto_ptr更安全。

**但unique_ptr还有更聪明的地方。**
有时候，会将一个智能指针赋给另一个并不会留下危险的悬挂指针。假设有如下函数定义： 
```cpp
unique_ptr<string> demo(const char * s)
{
    unique_ptr<string> temp (new string (s))； 
    return temp；
}
```
并假设编写了如下代码：
```cpp
unique_ptr<string> ps;
ps = demo('Uniquely special")；
```
demo()返回一个临时unique_ptr，然后ps接管了原本归返回的unique_ptr所有的对象，而返回时临时的 unique_ptr 被销毁，也就是说没有机会使用 unique_ptr 来访问无效的数据，换句话来说，这种赋值是不会出现任何问题的，即没有理由禁止这种赋值。实际上，编译器确实允许这种赋值，这正是unique_ptr更聪明的地方。

**总之，党程序试图将一个 unique_ptr 赋值给另一个时，如果源 unique_ptr 是个临时右值，编译器允许这么做；如果源 unique_ptr 将存在一段时间，编译器将禁止这么做**，比如：
```cpp
unique_ptr<string> pu1(new string ("hello world"));
unique_ptr<string> pu2;
pu2 = pu1;                                      // #1 not allowed
unique_ptr<string> pu3;
pu3 = unique_ptr<string>(new string ("You"));   // #2 allowed
```
其中#1留下悬挂的unique_ptr(pu1)，这可能导致危害。而#2不会留下悬挂的unique_ptr，因为它调用 unique_ptr 的构造函数，该构造函数创建的临时对象在其所有权让给 pu3 后就会被销毁。**这种随情况而已的行为表明，unique_ptr 优于允许两种赋值的auto_ptr 。**

当然，您可能确实想执行类似于#1的操作，仅当以非智能的方式使用摒弃的智能指针时（如解除引用时），这种赋值才不安全。要安全的重用这种指针，可给它赋新值。C++有一个标准库函数std::move()，让你能够将一个unique_ptr赋给另一个。下面是一个使用前述demo()函数的例子，该函数返回一个unique_ptr<string>对象：
使用move后，原来的指针仍转让所有权变成空指针，可以对其重新赋值。
```cpp
unique_ptr<string> ps1, ps2;
ps1 = demo("hello");
ps2 = move(ps1);
ps1 = demo("alexia");
cout << *ps2 << *ps1 << endl;
```

####5. 如何选择智能指针？####
在掌握了这几种智能指针后，大家可能会想另一个问题：在实际应用中，应使用哪种智能指针呢？
下面给出几个使用指南。

（1）如果程序要使用多个指向同一个对象的指针，应选择shared_ptr。这样的情况包括：
- 有一个指针数组，并使用一些辅助指针来标示特定的元素，如最大的元素和最小的元素；
- 两个对象包含都指向第三个对象的指针；
- STL容器包含指针。很多STL算法都支持复制和赋值操作，这些操作可用于shared_ptr，但不能用于unique_ptr（编译器发出warning）和auto_ptr（行为不确定）。如果你的编译器没有提供shared_ptr，可使用Boost库提供的shared_ptr。

（2）如果程序不需要多个指向同一个对象的指针，则可使用unique_ptr。如果函数使用new分配内存，并返还指向该内存的指针，将其返回类型声明为unique_ptr是不错的选择。这样，所有权转让给接受返回值的unique_ptr，而该智能指针将负责调用delete。可将unique_ptr存储到STL容器在那个，只要不调用将一个unique_ptr复制或赋给另一个算法（如sort()）。例如，可在程序中使用类似于下面的代码段。
```cpp
unique_ptr<int> make_int(int n)
{
    return unique_ptr<int>(new int(n));
}
void show(unique_ptr<int> &p1)
{
    cout << *a << ' ';
}
int main()
{
    ...
    vector<unique_ptr<int> > vp(size);
    for(int i = 0; i < vp.size(); i++)
        vp[i] = make_int(rand() % 1000);              // copy temporary unique_ptr
    vp.push_back(make_int(rand() % 1000));     // ok because arg is temporary
    for_each(vp.begin(), vp.end(), show);           // use for_each()
    ...
}
```
其中push_back调用没有问题，因为它返回一个临时unique_ptr，该unique_ptr被赋给vp中的一个unique_ptr。另外，如果按值而不是按引用给show()传递对象，for_each()将非法，因为这将导致使用一个来自vp的非临时unique_ptr初始化pi，而这是不允许的。前面说过，编译器将发现错误使用unique_ptr的企图。
在unique_ptr为右值时，可将其赋给shared_ptr，这与将一个unique_ptr赋给一个需要满足的条件相同。与前面一样，在下面的代码中，make_int()的返回类型为unique_ptr<int>：
```cpp
unique_ptr<int> pup(make_int(rand() % 1000));   // ok
shared_ptr<int> spp(pup);                       // not allowed, pup as lvalue
shared_ptr<int> spr(make_int(rand() % 1000));   // ok
```
模板shared_ptr包含一个显式构造函数，可用于将右值unique_ptr转换为shared_ptr。shared_ptr将接管原来归unique_ptr所有的对象。
在满足unique_ptr要求的条件时，也可使用auto_ptr，但unique_ptr是更好的选择。如果你的编译器没有unique_ptr，可考虑使用Boost库提供的scoped_ptr，它与unique_ptr类似。