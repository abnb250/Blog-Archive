# LaTex语法
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  

<br>

## 前言
***
本篇内容主要是整理LaTex语法的数学公式内容，  
其余LaTex语法内容本篇并未涉及，  
想了解其余内容可以通过其余渠道获得。  

<br>

<br>

## 01.数学模式概览
***
LaTex有两种数学模式：  
- 行内数学模式：随文本排版，公式不换行。  
- 行间数学模式：公式单独成行，居中，可以自动编号.

**行内数学模式**：  
写法：``` \( ... \) ``` 或 ``` $ ... $ ```。  
效果例如：$ E=mc^2 $ 。  
**行间数学模式**：  
写法：  
```
$$
...
$$
```
（不编号）或    ``` \begin{equation} ... \end{equation} ```（自动编号）  
效果例如:  
$$
E=mc^2 
$$   
**注：** 数学模式中的普通字母会被当做变量（斜体），数字和标点符号保持不变。空格会被忽略，需要用专门的间距命令（后述）。所有数学命令只在数学模式中有效。  

<br>

<br>

## 02.基本符号与结构
***
### 2.1 上下标
上标：```^```；下标：```_```。  
如果上下标的内容多于一个字符，则必须用花括号括起来：  
``` a_{ij} ```，``` 2^{n+1} ```，``` a_{b_{c}} ```（嵌套）。  
默认会贴在符号的右下/左上角，但在行间公式、大型运算符（如求和）中，会放在正上下标。  

<br>

### 2.2 分式
``` \frac{分子}{分母} ```：标准分式。  
在行内公式中，分式会缩小，可以用``` \tfrac ```强制行内大小。

<br>

### 2.3 根式
平方根：``` \sqrt{表达式} ```。    
n次根：``` \sqrt[n]{表达式} ```。  

<br>

### 2.4 希腊字母
|小写|命令|大写|命令|
|:---:|:---:|:---:|:---:|
|α|```\alpha```|Γ|```\Gamma```|
|β|```\beta```|Δ|```\Delta```|
|γ|```\gamma```|Θ|```\Theta```|
|δ|```\delta```|Λ|```\Lambda```|
|ε|```\epsilon``` 或 ```\varepsilon```|Ξ|```\Xi```|
|ζ|```\zeta```|Π|```\Pi```|
|η|```\eta```|Σ|```\Sigma```|
|θ|```\theta``` 或 ```\vartheta```|Υ|```\Upsilon```|
|ι|```\iota```|Φ|```\Phi```|
|κ|```\kappa```|Ψ|```\Psi```|
|λ|```\lambda```|Ω|```\Omega```|
|μ|```\mu```|		
|ν|```\nu```|	
|ξ|```\xi```|		
|π|```\pi``` 或 ```\varpi```|		
|ρ|```\rho``` 或 ```\varrho```|		
|σ|```\sigma``` 或 ```\varsigma```|		
|τ|```\tau```|		
|φ|```\phi``` 或 ```\varphi```|		
|χ|```\chi```|		
|ψ|```\psi```|		
|ω|```\omega```|

**注：** 大写希腊字母中，只有那些与拉丁字母不同形状的才有命令（如 ```\Pi```），像 A、B 等直接用拉丁字母即可。  

<br>

### 2.5 二元运算符与关系符
#### 2.5.1 加减乘除  
|**符号**|**代码**|
|:---:|:---:|
|$+$ | `+`|
|$- $ | `-`|
|$\times$ | `\times`|
|$\div$ | `\div`|
|$\cdot$ | `\cdot`|
|$\pm$ | `\pm`|
|$\mp$ | `\mp`|
#### 2.5.2 关系  
|**符号**|**代码**|
|:---:|:---:|
|$=$ | `=`|
|$\ne$ | `\ne`|
|$\approx$ | `\approx`|
|$\sim$ | `\sim`|
|$\simeq$ | `\simeq`|
|$\cong$ | `\cong`|
|$\equiv$ | `\equiv`|
#### 2.5.3 大小比较  
|**符号**|**代码**|
|:---:|:---:|
|$<$ | `<`|
|$>$ | `>`|
|$\le$ | `\le`|
|$\ge$ | `\ge`|
|$\ll$ | `\ll`|
|$\gg$ | `\gg`|
#### 2.5.4 集合
|**符号**|**代码**|
|:---:|:---:|
|$\in$ | `\in`|
|$\ni$ | `\ni`|
|$\subset$ | `\subset`|
|$\subseteq$ | `\subseteq`|
|$\supset$ | `\supset`|
|$\supseteq$ | `\supseteq`|
|$\cup$ | `\cup`|
|$\cap$ | `\cap`|
|$\emptyset$ | `\emptyset`|
|$\varnothing$ | `\varnothing`|
#### 2.5.5 逻辑
|**符号**|**代码**|
|:---:|:---:|
|$\forall$|`\forall`|
|$\exists$|`\exists`|  
|$\neg$|`\neg`|
|$\lor$|`\lor`|  
|$\land$|`\land`|  
|$\implies$|`\implies`|  
|$\iff$|`\iff`|  

<br>

### 2.6 箭头与点  
#### 2.6.1 单箭头
|**符号**|**代码**|
|:---:|:---:|
|$\leftarrow$|`\leftarrow`|  
|$\rightarrow$|`\rightarrow`|  
|$\uparrow$|`\uparrow`|  
|$\downarrow$|`\downarrow`|  
#### 2.6.2 双箭头
|**符号**|**代码**|
|:---:|:---:|
|$\Leftarrow$|`\Leftarrow`|
|$\Rightarrow$|`\Rightarrow`| 
|$\Uparrow$|`\Uparrow`|
|$\Downarrow$|`\Downarrow`|
#### 2.6.3 长箭头
|**符号**|**代码**|
|:---:|:---:|
|$\longleftarrow$|`\longleftarrow`|
|$\longrightarrow$|`\longrightarrow`|
#### 2.6.4 点
|**符号**|**代码**|
|:---:|:---:|
|$\dots$|`\dots`|
|$\cdots$|`\cdots`|
|$\vdots$|`\vdots`| 
|$\ddots$|`\ddots`|  

在数学中通常 `a_1,\dots,a_n` 用 `\dots` 会自动判断位置，但也可手动用 `\dotsc`（后跟逗号）等。  

<br>

<br>

## 03.运算符与大型运算符
***
### 3.1 常用大型运算符
|**名称**|**符号**|**代码**|
|:---:|:---:|:---:|
|求和|$\sum$|`\sum`  
|积分|$\int$|`\int`  
|乘积|$\prod$|`\prod`  
|并集|$\bigcup$|`\bigcup`  
|交集|$\bigcap$|`\bigcap`  

在行内公式中，默认在右上下角，行间公式中默认在正上下方。  
可以使用`\limits`强制在正方，`\nolimits`强制在角上。  

<br>

### 3.2 多重积分
|**名称**|**符号**|**代码**|
|:---:|:---:|:---:|
|二重|$\iint$|`\iint`|
|三重|$\iiint$|`\iiint`|  

例如：$\int_0^1 f(x)\,dx$ `\int_0^1 f(x)\,dx`  

<br>

### 3.3 在运算符号上下方加限制条件
使用`\substack`可以写多行条件：  
`\sum_{\substack{1\le i\le n \\ 1\le j\le m}} a_{ij}`  
$\sum_{\substack{1\le i\le n \\ 1\le j\le m}} a_{ij}$  
对于求和符号的特殊角标，可以用`\sideset`:（好像有错）  
`\sideset{_{a}^{b}}{_{c}^{d}}\sum`  
$\sideset{_{a}^{b}}{_{c}^{d}}\sum$  

<br>

<br>


## 分割线
下面的内容需要加载amsmath宏包，这对于有些markdown阅读器来说是没有的，会渲染不出来（包括博客园），如果有想要看完整版包括渲染后效果的，可以进文章开头的QQ群，群文件里面有pdf文档。  

<br>

<br>

## 04.多行公式与环境
***
需要加载amsmath宏包。  
vscode中的Markdown Preview Enhanced 插件默认集成了MathJax引擎，其已内置加载了amsmath扩展。  
### 4.1 gather环境
每行公式居中，独立编号，想要不编号就用gather*。  
代码：
```
$$
\begin{gather}
E = mc^2 \\
a^2 + b^2 = c^2
\end{gather}
$$
```
效果：
$$
\begin{gather}
E = mc^2 \\
a^2 + b^2 = c^2
\end{gather}
$$
在行末用`\\`换行，可以用`\notag`取消某行的编号。  

### 4.2 align环境
用`&`指定对齐点（通常是等号前面），实现多列对齐。  
代码：  
```
$$
\begin{align}
x + y &= z \notag \\
a + b &= c \notag
\end{align}
$$
```
效果：  
$$
\begin{align}
x + y &= z \notag \\
a + b &= c \notag
\end{align}
$$
一行内可以多个对齐列：  
代码：  
```
$$
\begin{align}
x &= 1 & y &= 2 \notag   \\
u &= 3 & v &= 4 \notag
\end{align}
$$
```
效果：  
$$
\begin{align}
x &= 1 & y &= 2 \notag   \\
u &= 3 & v &= 4 \notag
\end{align}
$$
这里`&`既用来对齐，又用来分隔列（就像表格的列分隔符）

### 4.3 split环境
用于在一个编号公式内部换行，必须放在equation环境内。  
代码：  
```
$$
\begin{equation}
\begin{split}
f(x) &= (x+1)^2 \\
&= x^2 + 2x + 1
\end{split}
\end{equation}
$$
```
效果：  
$$
\begin{equation}
\begin{split}
f(x) &= (x+1)^2 \\
&= x^2 + 2x + 1 \notag
\end{split}
\end{equation}
$$

### 4.4 手动编号与标签
`\tag{自定义编号}`可以覆盖自动编号，  
如`\tag{*}`会加一个型号，`\tag{1.2}·可指定任意编号。  
`\label{...}` 和 `\ref{...}` 可以交叉引用公式编号。  

<br>

<br>

## 05.矩阵与分段函数
***
### 5.1 矩阵环境
无括号：    `matrix`  
圆括号：( ) `pmatrix`   
方括号：[ ] `bmatrix`  
花括号：{ } `Bmatrix`  
单竖线：| | `vmatrix`  
双竖线：|| || `Vmatrix`  
以使用方括号为例：  
代码：  
```
$$
\begin{bmatrix}
a & b & c \\
d & e & f \\
g & h & i
\end{bmatrix}
$$
```
效果：  
$$
\begin{bmatrix}
a & b & c \\
d & e & f \\
g & h & i
\end{bmatrix}
$$
用`&`分隔列，`\\`换行。  

### 5.2 cases环境（分段函数）
代码：  
```
$$
f(x) = 
\begin{cases}
x^2, & \text{if } x \ge 0 \\
-x, & \text{otherwise}
\end{cases}
$$
```
效果：  
$$
f(x) = 
\begin{cases}
x^2, & \text{if } x \ge 0 \\
-x, & \text{otherwise}
\end{cases}
$$

<br>

<br>

## 06.数字符号的修饰
***
### 6.1 上下标修饰（帽子、横线、点等）
上横线：$\overline{abc}$ `\overline{abc}`  
下横线：$\underline{abc}$ `\underline{abc}`  
上花括号：$\overbrace{abc}^{label}$ `\overbrace{abc}^{label}`  
下花括号：$\underbrace{abc}_{label}$ `\underbrace{abc}_{label}`  
单字母向量：$\vec{v}$ `\vec{v}`  
多字母向量： $\overrightarrow{AB}$ `\overrightarrow{AB}`  
窄帽子：$\hat{a}$ `\hat{a}`  
宽帽子：$\widehat{abc}$ `\widehat{abc}`  
波浪线单：$\tilde{a}$ `\tilde{a}`  
波浪线多：$\widetilde{abc}$ `\widetilde{abc}`  
点：  
一阶导：$\dot{a}$ `\dot{a}`  
二阶导：$\ddot{a}$ `\ddot{a}`  

对符号加粗：`\boldsymbol{\alpha}`（amsmath 推荐）或 `\mathbf` 只能对拉丁字母生效。

### 6.2 数学字体
|**字体**|**代码**|**效果**|
|:---:|:---:|:---:|
|黑板粗体（空心）|`\mathbb{R}`|$\mathbb{R}$|
|粗体罗马|`\mathbf{A}`|$\mathbf{A}$|
|粗体希腊/符号|`\boldsymbol{\alpha}`|$\boldsymbol{\alpha}$|
|手写体（花体）|`\mathcal{ABC}`|$\mathcal{ABC}$|
|哥特体|`\mathfrak{ABC}`|$\mathfrak{ABC}$|
|花体（另一种）|`\mathscr{ABC}`|$\mathscr{ABC}$|
|无衬线|`\mathsf{ABC}`|$\mathsf{ABC}$|
|打字机体|`\mathtt{ABC}`|$\mathtt{ABC}$|

### 6.3 在数学公式中插入普通文本
`\text{文字}` 能自动处理当前字体和间距，比 `\mbox` 更好。  
例如：`f(x) = \text{如果 } x>0 \text{ 则 } x^2`  
$f(x) = \text{如果 } x>0 \text{ 则 } x^2$

### 6.4 调整间距
|**符号**|**代码**|
|:---:|:---:|
|`\,`|（小间距 3/18 em）|
|`\:` 或 `\>`|（中间距 4/18）|
|`\;`|（大间距 5/18）|
|`\!`|（负间距 -3/18）|
|`\quad`|（1 em）|
|`\qquad`|（2 em）|

例如：`\int_0^1 x^2 \, dx` 中的 `\,` 使 dx 稍微远离被积函数。  
$\int_0^1 x^2 \, dx$

<br>

<br>


## 07.实战提示与常见错误
***
- 忘记切换数学模式：普通模式下的`\alpha`会报错  
- 花括号不匹配：使用{}时务必确保成对出现  
- 数学模式中的空格无效：必须使用间距命令  
- 多行公式对齐：记住`&`的位置，每行中`&`的个数应一致  
- 大型上下标乱跑：用`\limits`或`\nolimits`控制  
- amsmath必须加载：几乎所有高级数学功能都需要它  
- 空行在数学模式中禁止：空行在数学模式内部会报错，但可以在行间公式外部使用空行分隔段落。  

<br>

<br>

 