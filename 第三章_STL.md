# 第三章 

## 3.1 STL

STL ：Standard Template Library 标准模板库

STL的作用不仅是提供了string、vector、map、set等容器，更重要的是STL封装了许多复杂的数据结构算法和操作，vector封装了数组、list封装了链表、map和set封装了二叉树等等，还提供了插入、删除、查找、排序等操作。



## 3.2 string（不重要）

#include \<string\>

### 1.string类的实现

其实就是一个 char *m_data，封装了一个C语言的char数组和相关的操作。

strlen()计算char字符串长度时，不把'\0'计入其中，strcpy()复制char字符串时将'\0'也复制过去。

### 2.string的声明方式

skip

### 3.C++字符串与C字符串间的转换

C++字符串转C字符串：data()、c_str()或者copy()

只看c_str()方法：

c_str()方法得到一个指向string字符串的const指针，缺点是如果string字符串被改变的话，读取指针指向的字符串自然会改变。

```C++
int main(){
    string str="Hello";
    const char *cstr=str.c_str();
    cout<<cstr;//Hello
    str="World";
    cout<<cstr;//World
}
```

为了使转换得到的char字符数组保持不变，可以将内容复制出来

```C++
int main(){
    string str="Hello";
    char *cstr=new char[20];
    strncpy(cstr,str.c_str(),str.size());
}
```

### 4.string和int的转换

snprintf(), strtol()等方法太麻烦，不如用sstream

```C++
#include <iostream>
#include <sstream>
using namespace std;
int main()
{
    //string转int
	string str="10";
	int num;
	stringstream ss;
	ss<<str;
	ss>>num;
	cout<<num+1;
 
    ss.clear();//将读写状态重置
    ss.str("");//将内容置空
    
    //int转string
    int num2=100;
	string str2;
	ss<<num2;
	ss>>str2;
	str2[0]='d';
	cout<<str2;
}
```



## 3.3 vector

### 3.3.1 vector

#include \<vector\>

vector元素存储在一块连续的存储空间中，可以用迭代器或者下标方式访问元素。

vector::size()返回容器大小，vector::capacity()返回容器容量。

容器的大小和容量是不同的，大小是指元素的个数，容量是指给容器分配的内存大小。容量多于容器大小的部分用于将来容纳新插入的元素。

### 3.3.2 vector 增删查

#### 1.vector初始化和遍历

```C++
vector<T> v1; //创建空vector
vector<T> v2=v1; //创建一个跟v1相同的vector
int a[3] = {1,2,3};
vector<int> v(a,a+3);//直接给元素的初始化值
vector<int> v={0,1,2};//这样初始化也是可以的
```

不建议创建空vector然后一个个元素地push_back进去，会导致vector多次重新分配内存来扩大容器容量，应该先用reserve指定容量，然后再push_back。

遍历方式：

```C++
for(int i=0;i<v.size();i++) cout<<v[i];

vector<int>::iterator iter;
for(iter=v.begin();iter!=v.end();iter++) cout<<*iter;
```

vector是一个模板类，可以放任何类型的对象，放结构体的情况下，可以通过sort函数，按照自己定义的排序方式排序。

```C++
#include <algorithm>
#include <vector>
struct Rect{
    int len,width;
};
bool cmp(Rect a, Rect b){
    if(a.len!=b.len) return a.len<b.len;
    else return a.width<b.width;
}
vector<Rect> v={...};
sort(v.begin(),v.end(),cmp);
//此外也可以在结构体内重载操作符<，然后sort(v.begin(),v.end());
```



#### 2.vector查找

```C++
#include <algorithm>
vector<int>::iterator iter=find(v.begin(),v.end(),3);//查找值为3的元素
if(iter!=v.end()) cout<<"Found";
```



#### 3.vector删除

erase删除指定值或者指定位置的元素，pop_back删除最后一个元素。

erase函数原型有两个：

```C++
iterator erase(iterator position)
iterator erase(iterator begin,iterator end)
```

删除vector中值为3的元素的一个经典错误：

```C++
vector<int> v={1,2,3,4};
for(vector<int>::iterator iter=v.begin();iter!=v.end();iter++){
    if(*iter==3){
        v.erase(iter);
    }
}
```

错误的地方在于，当 v.erase(iter)执行完后iter就变成了野指针，无法进行iter++，无法继续遍历下去，正确做法是

```C++
int a[3] = {1,2,3};
vector<int> v(a,a+3);
for(vector<int>::iterator iter=v.begin();iter!=v.end();){
    if(*iter==3){
        iter=v.erase(iter);//返回值是更新后的容器的被删除元素的后一个元素的迭代器
    }
    else{
        iter++;
    }
}
```

#### 4.vector增加

insert将元素插入到某个位置，push_back在最后面添加一个元素。

insert(pos, value)是将value插入到pos位置，原本pos位置的元素向后移一位。

```C++
int a[3] = {0,1,2};
vector<int> v(a,a+3);
v.insert(v.begin()+1,5);//{0,5,1,2}
v.insert(v.end(),6);//把6插入到最后一个元素的下一个位置，相当于push_back()  {0,5,1,2,6}
```



### 3.3.3 vector的内容管理与效率

#### 1.使用reserve()提前设定容量大小

vector使用动态数组来实现。增加元素后如果动态数组的内存（容器容量）不够用，就会重新分配一个2倍大小的内存，然后把原数组内容复制过去，这个过程会使vector性能下降。因此最好提前用reserve()函数设定容量大小，就不会因为多次容量扩充而导致效率下降。

相关函数：

1. size() 获取容器中有多少个元素；
2. capacity() 获取容器已经分配的内存总共能容纳多少个元素；就是说capacity()-size()就是容器还能容纳的元素个数；
3. resize(n) 强制改为容纳n个元素，后面的截断；
4. reserve(n) 强制将容量改为能容纳不小于n个元素。

以下代码就不会有容器内存重新分配

```C++
vector<int> v;
v.reserve(1000);
for(int i=0;i<1000;i++){
    v.push_back(i);
}
```

容器内存的重新分配会导致旧的迭代器、指针、引用失效（因为内容被复制到新的内存位置去了）

因为每次容器扩容不是在原有空间上扩充，而是新选一块更大的连续内存，把容器元素逐个复制过去，最后销毁旧的内存，此时指向旧内存空间的迭代器已经失效，需要更新迭代器。

#### 2.修整过剩空间

一句话将vector容量收缩到合适：

```C++
vector<int>(v).swap(v)
```

先创建一个临时容器把v中的元素拷贝过去（临时容器的容量刚刚合适），然后交换两个容器的内容，最后销毁临时容器，这样v的容量就收缩到合适了。



### 3.3.4 vector类的实现

最主要的是vector类的成员变量

```C++
private:
	T* array;//指针，用来分配内存开辟动态数组
	unsigned int theSize;//大小
	unsigned int theCapacity;//容量
```

vector的push_back, pop_back和下标随机访问效率很高，insert, erase和容量扩容效率很低。



## 3.4 map

### 3.4.1 map是什么

map内部自建一个红黑树，对数据自动排序。

map<key_type, value_type> m;

对于key的类型 ，有一个约束是必须支持 < 操作符，不然自动排序时会出错。

map根据key在n个结点中快速查询记录，复杂度为log(n)

### 3.4.2map的增删查

#### 1.map插入

```C++
map<int,string> m;
//方式一：用insert函数插入pair数据
insert_pair=m.insert(pair<int,string>(1,"student1"));
if(insert_pair.second==true){cout<<"insert successfully";}//判断是否插入成功
//方式二：用数组方式插入数据
m[2]="student2";
```

当key值已经存在于map中时，想要再插入相同key值的键值对，此时insert操作无法插入数据，而数组方式则会覆盖掉原有键值对。

#### 2.map的遍历

跟vector的迭代器遍历一样

```C++
//正向遍历
for(map<int,string>::iterator iter=m.begin();iter!=m.end();iter++){
    cout<<iter->first<<iter->second<<endl;
}
//反向遍历
for(map<int,string>::reverse_iterator iter=m.rbegin();iter!=m.rend();iter++){
    cout<<iter->first<<iter->second<<endl;
}
```

#### 4.map的查找

```C++
int cnt=m.count(key);//为0说明不存在key，为1说明存在key
map<int,string> iter=m.find(key);//iter!=m.end()说明存在key
```

#### 5.map的删除

```C++
m.erase(key);
m.erase(iter);//最常用，但也要注意删除时不能让iter变成野指针
m.erase(iter_begin,iter_end);

for(map<int,string>::iterator iter=m.begin();iter!=m.end();){
    if(iter->second=="student2"){
        m.erase(iter++);
    }
    else{
        iter++;
    }
}
```

#### 6.map的排序

map的自动排序默认按key值从小到大排序。

函数对象：调用操作符的类生成的对象，下面的less\<Key\>就是一个函数对象。

```C++
map<string,int,less<string>> m;
map<string,int,greater<string>> m;//从大到小
```

自己写一个函数对象，按自己想要的顺序来排序。

```C++
struct CmpByKeyLength{
    bool operation()(const string& k1, const string& k2){
        return k1.length()<k2.length();
    }
};
map<string,int,CmpByKeyLength> m;
```

如果key的类型是结构体，则必须重载 < 操作符，否则insert时无法进行排序操作（编译错误）。

```C++
struct Stu{
    int id;
    bool operation < (Stu const& s)const{
        return id<r.id;
    }
}
```

上面的方法都是借助map内部的红黑树，按照key排序的。如果想按value排序，就会想到用sort()函数，但他只能对序列容器（vector、list、deque）进行排序。所以可以将map的pair键值对全部放到vector中，然后用sort()排序。pair本身重载了 < 操作符，是先按key排序，key相同的情况下再按value排序。要想让pair按value排序，可以写比较函数或者函数对象。

```C++
//比较函数
bool cmp(pair<string,int> &p1,pair<string,int> &p2){
    return p1.second<p2.second;
}
vector<pair<string,int>> v(m.begin(),m.end());
sort(v.begin().v.end(),cmp);
for(int i=0;i<v.size();i++){
    cout<<v[i].first<<v[i].second;
}
```

但是借助vector来排序的话，排序之后容器已经换成vector了。

### 3.4.3 map原理_红黑树

map和set用的都是红黑树（RBTree）。红黑树是结点有颜色的二叉查找树。

**二叉查找树/有序二叉树**：每个结点都满足：    左子树所有结点<根节点<右子树所有结点

理想情况下，包含n个结点的二叉查找树高度为log(n)，极端情况下有可能退化成一条线性链，复杂度上升到O(n)

**平衡二叉树**（AVL）：每个结点的左右子树高度之差小于等于1，且左右子树都是平衡二叉树。

map之所以不用二叉查找树，是因为它的性能不稳定，容易出现O(n)，不用平衡二叉树，是因为它的要求太严格，插入结点导致过多的旋转操作，开销较大（AVL适合查找操作多而插入和删除操作少的情况）。而红黑树性能为O(log(n))，且旋转操作少一些。

***

**红黑树的五个特性（最重要）：**

1. 每个结点要么是红色，要么是黑色的。
2. 根结点必须是黑色的。
3. 每个叶子结点是黑色的（叶子结点是指为NULL的叶子结点）。
4. 红色结点的子结点必须是黑色的。
5. 一个结点到其子孙结点的所有路径上包含相同数目的黑色结点。

***

特性3和特性4使得每条路径上红色结点数目<=黑色结点数目，结合特性5，使得能够确保没有一条路径会是另一条路径的两倍长。可以证明得到一个含有n个结点的红黑树的高度最多为 **2log(n+1)** ，因此红黑树复杂度诶O(log(n))。（证明比较复杂，有需要的时候再看）

向红黑树中插入或者删除结点，可能会破坏红黑树的性质，需要利用颜色重涂和旋转来恢复红黑性质。（情况很多且操作复杂，有需要时再看）

新插入的结点是红色的，因为如果是黑色，则肯定破坏特性5，因此让新结点为红色会更好，有可能没有破坏特性，直接插入成功。



## 3.5 set

set用来存储同一类型的数据，每个元素的值唯一，且插入时自动排序。

map和set都是用红黑树实现的，避免了vector动态数组的一些缺点。

比如set插入和删除操作都是高效的，不需要移动后面元素，除被删除元素外，其他iterator也不会失效（没有内存移动），元素增多时查找和插入的速度受影响很小。这些都是树的优点。（vector序列好处是随机访问速度快）

**set的增删查：**

```C++
//创建空的set对象
set<int> s;
//创建set对象
int a[3]={0,1,2};
set<int> s(a,a+3);
//插入新元素
s.insert(value);//不能插入重复元素
//删除元素
s.erase(value);
s.erase(iter);//iter=s.begin()之类的
//查找元素
s.count(value);//为0或1
iter=s.find(value);//不为end()就是找到了
//清空
s.clear();

```

