# STL
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
回到主目录点[这里](https://www.cnblogs.com/abnb250/articles/20384791)  

<br>

<br>

## map
***
map及其函数：map<key, value>提供一对一的数据处理能力，第一个值被称为关键字key，每个关键字只能在map中出现一次，第二个值被称为关键字的值value，可以重复  

代码模板如下：  
```cpp
// 定义，以 string, int 对为例
#include <map>
map<string, int> mp;             // 底层是红黑树，元素可比较大小，key 的结构体要重载 < 运算

// 2- 插入
mp.insert(make_pair("zhs", 18));       // 插入一个键值对，若原先存在该 key，则无法插入和覆盖
mp.insert(pair<string,int>("zhs", 18)); // 插入一个键值对，若原先存在该 key，则无法插入和覆盖
mp["zhs"] = 18;                        // 甚至可以类似数组下标去访问 key 对应的 value，若原先不存在该 key，则创建，若原先存在，则覆盖以前该关键字对应的值
mp.emplace("zhs", 18);                 // 插入效果跟 insert 效果完全一样

// 3- 删除
mp.erase(key);        // 删除值为 key 的元素
mp.erase(iter);       // 删除迭代器 iter 指向的元素，例如
mp.erase(mp.begin());
mp.erase(iter1, iter2); // 删除区间 [iter1, iter2) 的所有元素，例如 mp.erase(mp.begin(), mp.end());
mp.clear();           // 清空集合

// 3- 求大小
int siz = mp.size();         // 求集合大小
bool flag = mp.empty();      // 集合判空

// 4-查询
if(mp.find(key) != mp.end())          // find 函数返回一个指向被找到元素的迭代器
    cout << mp[key] << endl;
if(mp.count(key))                     // count 返回某个值元素的个数
    cout << mp[key] << endl;
auto iter = mp.lower_bound(key);      // 求 key 的下界，返回指向大于等于某值的第一个元素的迭代器
auto iter = mp.upper_bound(key);      // 求 key 的上界，返回大于某个值元素的迭代器

// 5-遍历
map<string, int>::iterator iter;      // 正向遍历
for(iter=mp.begin(); iter!=mp.end(); iter++)
{
    cout << iter->first << " " << iter->second << endl;
    // 或者
    cout << (*iter).first << " " << (*iter).second << endl;
}
map<int>::reverse_iterator riter;     // 反向遍历
for(riter=mp.rbegin(); riter!=mp.rend(); riter++)
{
    // 遍历的同时修改
    iter->second += 10;
    cout << iter->first << " " << iter->second << endl;
}
for (auto x : mp){                    // C++ 11 auto 遍历
    cout << x.first << " " << x.second << endl;
}
for (auto& x : mp){                   // C++ 11 auto 遍历
    x.second += 10;                   // 遍历并修改
    cout << x.first << " " << x.second << endl;
}

// 6- 求最值
map<string, int>::iterator it = mp.begin();     // 最小值
cout << *it << endl;
map<string, int>::iterator it = mp.end();       // 最大值
cout << *(--it) << endl;

/*
1. map 和 set 一样，其中的元素必须支持 < 运算符
2. 在插入时，使用 insert 和 用数组方式去插入，在原先存在要插入的键值
时是有区别的，insert不会插入，用数组方式插入则会直接覆盖原先数据
3. map 的迭代器支持 ++、--、==、!=、(*iter).first、
(*iter).second、iter->first、 iter->second 等操作
4. map 的迭代器不支持 +、-、 +=、 -= 等算术操作符，也不支持 >、<、
>=、<= 等比较操作符
*/
```

<br>

<br>

## set
***
Set及其函数：set与map有点像，只是set只有一个值，同样自动排序。所有的操作都严格在O(logn)时间内完成。set插入的元素不能相同，而multiset可以重复。  

代码模板如下：  
```cpp
// 1- 定义
#include <set>
set<int> s;       // 元素必须可比较大小，元素类型必须要支持 < 运算，结构体需要重载 <

// 2- 插入
s.insert(key);    // 插入

// 3- 删除
s.erase(key);     // 删除值为 key 的元素
s.erase(iter);    // 删除迭代器 iter 指向的元素，例如 s.erase(s.begin());
s.erase(iter1, iter2); // 删除区间 [iter1, iter2) 的所有元素，例如 s.erase(s.begin(), s.end());
s.clear();        // 清空集合

// 4- 求大小
int siz = s.size();       // 求集合大小
bool flag = s.empty();    // 集合判空

// 5-查询
if(s.find(key) != s.end())        // find 函数返回一个指向被查找到元素的迭代器
    cout << "exist" << endl;
if(s.count(key) == 1)             // count 返回某个值元素的个数
    cout << "exist" << endl;

set<int>::iterator iter = s.lower_bound(key);  // 求 key 的下界，返回指向大于等于某值的第一个元素的迭代器
set<int>::iterator iter = s.upper_bound(key);  // 求 key 的上界，返回大于某个值元素的迭代器

// auto 类型推断关键字 在NOI 系列比赛中也可以使用了
auto iter = s.lower_bound(key);  // 求 key 的下界，返回指向大于等于某值的第一个元素的迭代器
auto iter = s.upper_bound(key);  // 求 key 的上界，返回大于某个值元素的迭代器

// 6-遍历
set<int>::iterator iter;          // 正向遍历
for(iter=s.begin(); iter!=s.end(); iter++)
{
    cout<<*iter<<endl;
}
set<int>::reverse_iterator riter; // 反向遍历，不重要
for(riter=s.rbegin(); riter!=s.rend(); riter++)
{
    cout<<*riter<<endl;
}

// 7- 求最值
set<int>::iterator it = s.begin();  // 最小值
cout << *it << endl;
set<int>::iterator it = s.end();    // 最大值
cout << *(--it) << endl;

/*
注意：
1. set 和优先队列一样，其中的元素必须支持 < 运算符
2. set 的迭代器支持 ++、--、==、!=、*iter 等操作
3. set 的迭代器不支持 +、-、 +=、 -= 等算术操作符，也不支持 >、<、>=、<= 等比较操作符
*/
```


<br>

<br>

## 优先队列
***
priority_queue优先队列：堆的作用主要是用来获取最大值和最小值，因此堆有个别名叫优先队列。  

常见代码模板如下：  
```cpp
#include <queue>
using namespace std;
// using std::priority_queue;
priority_queue<int> q;       // 定义一个空的优先队列，默认是大根堆

void test()
{

    int x = 5;
    q.push(x);           // 将 5 插入堆，这时堆顶是 5
    q.push(3);           // 将 3 插入堆，这时堆顶是 5
    q.push(2*x);         // 将 10 插入堆，这时堆顶是 10
    int k = q.top();     // 取堆顶元素，应该是 10
    q.pop();             // 删除堆顶元素，删除了 10，这时堆顶为 5
    int s = q.size();    // 求堆中元素个数，应为 2
    // 清空优先队列的一种方法
    while(!q.empty())
        q.pop();
    // 清空优先队列的第二种方法
    q = priority_queue<int>();
}
```

实现小根堆：  
```cpp
// 大根堆改成小根堆，其中 vector<Type> 为内部存储容器，
// greater<Type> 为比较方式
priority_queue<Type,vector<Type>,greater<Type>> q;

// 常为 int 类型
priority_queue<int,vector<int>,greater<int>> q;
```

优先队列与结构体重载运算符：  
```cpp
// 只能用于大根堆的结构体
struct node{
    int val;
    string info;
    ...
    // 大根堆，重载小于运算符
    bool operator <(const node& other) const {
        return val < other.val;
    }
};
// 定义一个大根堆（默认）
priority_queue<node> q;

// 只能用于小根堆的结构体
struct node{
    int val;
    string info;
    ...
    // 小根堆，重载大于运算符
    bool operator >(const node& other) const {
        return val > other.val;
    }
};
// 定义一个小根堆（默认）
priority_queue<node,vector<node>,greater<node>> q;
```

<br>

<br>

## pair
***
pair：将两个变量成对组合打包在一起的数据类型，常见的应用就是保存坐标。支持比较运算符，先比较第一个元素，再比较第二个元素  
```cpp
pair<int,int> pos;
pos.first = 1;
pos.second = 1;
a = make_pair(1,1);
a = pair<int,int>(1,1);
```

<br>

<br>

## vector
***
vector及其函数

| 问题 | 普通数组 | vector 数组 |
| :--- | :--- | :--- |
| 定义数组 | `int a[N];`<br>`int a[N] = {0}; // 全初始为 0` | `vector<int> a;`<br>`vector<int> a(N);`<br>`vector<int> a(N, 1);// 全赋值为 1` |
| 输入元素 | `cin >> a[i];` | `cin>>t; a.push_back(t);` |
| 比较两数组相等 | `for(int i = 0; i < n; i++){ if(a[i] == b[i]) ...... }` | `if(a == b)` |
| 删除最后一个元素 | `len--;` | `a.pop_back();` |
| 删除下标 i 位置的元素 | `for(int j = i+1; j < n; j++){ a[j-1] = a[j]; }` | `a.erase(a.begin()+i, a.begin()+i+1);` |
| 删除下标 t1~t2 的所有元素 | `。。。。。。` | `a.erase(a.begin()+t1, a.begin()+t2+1);` |
| 在下标 i 处插入一个元素 | `for(int j = n+1; j >= i+1; j--){ a[j] = a[j-1]; }`<br>`a[i] = t;` | `a.insert(a.begin()+i, t);` |
| 翻转数组 | `reverse(a, a+n);` | `reverse(a.begin(), a.end());` |
| 排序 | `sort(a, a+n);` | `sort(a.begin(), a.end());` |

<br>

<br>

## 字符串
***
字符串：常见字符串函数与reverse  
```cpp
// 最常用的操作
str.size();//返回字符串长度
str.length();//返回字符串长度
str.empty();//检查 str 是否为空，为空返回 1，否则返回 0
str[n];//存取 str 第 n + 1 个字符
str.at(n);//存取 str 第 n + 1 个字符 (如果溢出会抛出异常)

// 反转
reverse(str.begin(), str.end());

// 查找
str.find("ab");//返回字符串 ab 在 str 的位置
str.find("ab", 2);//在 str[2]~str[n-1] 范围内查找并返回字符串 ab 在 str 的位置
str.rfind("ab", 2);//在 str[0]~str[2] 范围内查找并返回字符串 ab 在 str 的位置
if(str.find("ab")!=string::npos)
{   cout << "下标为:" << str.find("ab"); }

// 子串
str.substr(3); // 返回 [3] 及以后的子串
str.substr(2, 4); // 返回 str[2]~str[2+(4-1)] 子串(即从[2]开始4个字符组成的字符串)
str.substring(5, 10); // 返回 str[5]~str[9] 子串(不包含结尾)

// 插入
str.insert(2, "sz");//从 [2] 位置开始添加字符串 "sz"，并返回形成的新字符串
str.insert(2, "abcd", 3);//从 [2] 位置开始添加字符串 "abcd" 的前 3 个字符，并返回形成的新字符串
str.insert(2, "abcd", 1, 3);//从 [2] 位置开始添加字符串 "abcd" 的前 [2]~[2+(3-1)] 个字符，并返回形成的新字符串

// 删除
str.erase(3);//删除 [3] 及以后的字符，并返回新字符串
str.erase(3, 5);//删除从 [3] 开始的 5 个字符，并返回新字符串

// 替换
str.replace(2, 4, "sz");//返回把 [2]~[2+(4-1)] 的内容替换为 "sz" 后的新字符串
str.replace(2, 4, "abcd", 3);//返回把 [2]~[2+(4-1)] 的内容替换为 "abcd" 的前3个字符后的新字符串

// 追加
str = str + "abc";
str.push_back('a');//在 str 末尾添加字符'a'
str.append("abc");//在 str 末尾添加字符串"abc"

```

ASCII码值、字母大小写转换、‘0’~‘9’  
```cpp
// 数字转字符: 'A'(65)  'a'(97)  '0'(48)
char A = char(65);
char a = char(97);
char c = 'a' + 2;        // 'c' = 'a' + 2
char seven = '0' + 7;    // '7' = '0' + 7

// 字符转数字
int a = 'a';       // a: 97
int A = 'A';       // A: 65
int t = 't' - 'a';       // 计算字母间差值
int seven = '7' - '0';
```

stringstream：主要用于字符串分割，可以先用clear（）以及str（）将字符串读入并进行分割，再用>>把内容输出  
```cpp
string s;
stringstream ss;
int a, b, c;
getline(cin, s);
ss.clear();
ss.str(s);
ss >> a >> b >> c;
```

字典序：字典序就是以ASCII码排序，字符串是逐位比较：ab  > a ,abcd > abbb。使用sort可以对字符串进行字典序排序，字符按ASCII码由小到大排列
```cpp
string str = "agfdvdsgds";
sort(str.begin(),str.end());
```

<br>

<br>

## 结构体
***
结构体：结构体是自定义类型，它就像一个能装不同数据类型的“包裹”。当你声明一个结构体后，需要赋予这个结构体一个名字（即结构体名）。而定义具体变量时，也可以定义结构体数组。  

1.结构体的声明  
```cpp
struct 结构体类型名{       //其中，struct是关键字，在c++中表示结构体
    数据类型 成员变量1;    //多个成员变量可以具有不同的数据类型
    数据类型 成员变量2;
    ......
};

结构体类型名 变量名; （struct 可省略）
```

下面是一个例子  
```cpp
struct student{   //student表示结构体的类型名
    string name;
    char sex;
    int age;
    float height,score;
};

student zhangsan;//声明一个student的结构体变量叫zhangsan

student a[1001];//创建结构体数组a
//a数组中的每一个元素都是一个结构体类型student的变量
```

我们也能同时定义结构体和变量  
```cpp
struct student{
    string name;
    char sex;
    int age;
    float height,score;
}a[1001];
```
2.结构体的使用  
将结构体变量视为一个整体进行操作  
```cpp
swap(a[1], a[2]);
```

使用符号 . 对结构体变量的成员进行操作  
```cpp
结构体名.成员变量名
```
下面是一个例子  
```cpp
student a;//李四
cin >> a.name;//输入赋值李四的名字
a.age = 23;//直接赋值给李四的年龄为23
```

3.结构体初始化  
```cpp
student a = {"tuotuo",'F',12,168,100};

student a;
cin >> a.name >> a.sex;
a.age = 12;
```

4.结构体中的sort函数  
```cpp
struct student {
    string name;
    int age;
    float height;
}a[10];
bool cmp(student a, student b){
    return a.age < b.age;
}
int main(){
    ...//输出
    sort(a, a + n, cmp);//按照年龄从小到大排序
    ...//输出
}
```

数组标记和数组连续性  
```cpp
// 标记数组
int flag[N];  // flag[i] 用于标记状态或者计数

flag[i] = 1;  // 数组标记状态，选择 / 不选择

flag[i]++;    // 数组标记计数，记录i出现的次数
```

数组连续性  
```cpp
// 连续最长相同、最长上升或下降：相邻两项比较，并讨论即可
int a[N], s = 0;
int ans = 0;

for(int i = 1; i <= n; i++) {
    if(a[i] == a[i-1]) {  // a[i] < a[i-1] 或 a[i] > a[i-1]
        s++;
        ans = max(ans, s);
    } else {
        s = 1;
    }
}
cout << ans;
```

<br>

<br>





