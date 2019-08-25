---
title: 代码简洁之道 
date: 2019-05-14 15:29:30
tags: 代码规范
---

### **测试代码质量的唯一方式：别人看你代码时说 f \* k 的次数。**

代码质量与其整洁度成正比。干净的代码，既在质量上较为可靠，也为后期维护、升级奠定了良好基础。

本文并不是代码风格指南，而是关于代码的`可读性`、`复用性`、`扩展性`探讨。  

<!-- more -->

我们将从几个方面展开讨论：


## 变量

## 用有意义且常用的单词命名变量

### Bad：

```c
int a = 1000;
//这样用a、b、c等字母做变量名是一个很不好的习惯。
```

### Good:

```c
int nowCurrent = 1000;
//nowCurrent变量名，一看就是用来定义“现在电流”的变量。
```



### 保持统一

可能同一个项目对于获取不同的变量，会有三个不一样的命名。应该保持统一，如果你不知道该如何取名，可以去 [codelf](https://link.juejin.im/?target=https%3A%2F%2Funbug.github.io%2Fcodelf%2F) 搜索，看别人是怎么取名的。

### Bad:

```c
get_p11_I();
//获取P1^1的电流
get_V_p22();
//获取P2^2的电压
```

### Good:

```
get_data_i()
get_data_v()
```





### 每个常量都该命名

### Bad:

```c
// 三个月之后你还能知道 50 是什么吗?能达到什么效果吗？
set_pwm(50);
```

### Good:

```c
int right_moto_go = 50;
//这个变量名，几个月后一看还是可以知道是一个使“左轮前进”的变量所需的pwm。
set_pwm(right_moto_go);
```



### 可描述

通过一个变量生成了一个新变量，也需要为这个新变量命名，也就是说每个变量当你看到他第一眼你就知道他是干什么的。

### Bad:

```c
void run(int lv , int rv){
	...
}
//定义一个电机驱动函数

//以下是错误实例
//给左轮100、右轮0。使小车左轮。
run(100,0);

```

### Good:

```c
void run(int lv , int rv){
	...
}
//定义一个电机驱动函数

//以下是正确示范。一眼就可以看出是是小车左转需要给左轮100的占空比，右轮0的占空比。
int turn_left_lv = 100 , turn_right_lv = 0;

run(turn_left_lv , turn_right_lv);
```



### 避免无意义的前缀

如果创建了一个结构体 car，就没有必要把它的颜色命名为 carColor。

### Bad:

```c
struct car {
  char[] carMake,
  char[] carModel,
  char[] carColor
}

void paintCar(car) {
  car.carColor = 'Red';
}
```

### Good:

```c
struct car {
  char[] make,
  char[] model,
  char[] color
}

void paintCar(car) {
  car.color = 'Red';
}
```





## 函数

函数的命名很有讲究，在Java中一般采用驼峰命名法，而在c语言中一般常采用小写英文字母+"_"的命名规则。如：`get_i_data()`（获取电流数据）`set_pwm()`(设置pwm值)。

### 参数越少越好

如果参数超过两个，在某种特殊的语言中，如 `ES2015/ES6` 的解构语法，不用考虑参数的顺序。

### Bad:

```javascript
function createMenu(title, body, buttonText, cancellable) {
  // ...
}
```

#### Good:

```javascript
function createMenu({ title, body, buttonText, cancellable }) {
  // ...
}

createMenu({
  title: 'Foo',
  body: 'Bar',
  buttonText: 'Baz',
  cancellable: true
});
```



### 只做一件事

这是一条在软件工程领域流传久远的规则。严格遵守这条规则会让你的代码可读性更好，也更容易重构。如果违反这个规则，那么代码会很难被测试或者重用。

### Bad：

```c
void get_data(int *i , int *v){
    *i = ....;
    *v = ....;
    //这个函数传入两个指针，即该函数实现了既能获取电流i、又实现了获取电压v的功能。
}
```

### Good：

```c
int get_i(){
    int i;
    ...
	return i;
}

int get_v(){
    int v;
    ...
	return v;
}
```



### 顾名思意

看函数名就应该知道它是干啥的。

### Bad：

```c
void addToData(int i){
    
    ...
}

addtoData(500);
//很难看出把500添加到什么数据里
```

### Good:

```c
void add_i_to_data(int i){
	...
}

add_i_to_data(500);
//一看可以看出将500添加到电流的数据里
```



### 删除重复代码

很多时候虽然是同一个功能，但由于一两个不同点，让你不得不写两个几乎相同的函数。

要想优化重复代码需要有较强的抽象能力，错误的抽象还不如重复代码。所以在抽象过程中必须要遵循 `SOLID` 原则（`SOLID` 是什么？稍后会详细介绍）。



### 不要传flag参数

通过 flag 的 大小，来判断执行逻辑，违反了一个函数干一件事的原则。

### Bad:

```c
int calculate(int i ,int j ,int flag){
    if(flag == 1)return i+j;
    if(flag == 2)return i-j;
    ....
    
}
```

### Good:

```c
int add(int i , int j ){
	return i+j;
}

int delete(int i , int j ){
    return i-j;
}
```



### 避免副作用

函数接收一个值返回一个新值，除此之外的行为我们都称之为副作用，比如修改全局变量、对文件进行 IO 操作等。

当函数确实需要副作用时，比如对文件进行 IO 操作时，请不要用多个函数/类进行文件操作，有且仅用一个函数/类来处理。也就是说副作用需要在唯一的地方处理。

副作用的三大天坑：随意修改可变数据类型、随意分享没有数据结构的状态、没有在统一地方处理副作用。

### Bad:

```c
int i = 100;

void add_i(){
    i++;
}

int main(){
    printf("%d",i);
    add_i();
    printf("%d",i);
}
```

### Good:

```c
int i = 100;

int add_i(){
	return ++i;
}

int main(){
    int j =0;
    printf("%d",i);
    j = add_i();
    printf("%d",j);
}
```



### 封装条件语句

### Bad:

```c
if(i == 1 && j < 100 && k > 99){
    ...
}
```

### Good:

```c
int check(int i, int j ,int k){
    return (i == 1 && j < 100 && k > 99) ? 1:0;
}

if(check(i,j,k)){
    
    ...
}
```



### 尽量别用"非"条件句

### Bad:

```c
int is_not_int(i){
    ...

}

if(!is_not_int(1)){
    ...
}
```



### Good:

```c
int is_int(i){
    ...

}

if(is_int(1)){
    ...
}
```



### 删除弃用代码

很多时候有些代码已经没有用了，但担心以后会用，舍不得删。

如果你忘了这件事，这些代码就永远存在那里了。

放心删吧，你可以在代码库历史版本中找他它。

### bad:

```c
//int old_get_i();
//int old_get_i_update();
int new_get_i();
```

### Good:

```
int new_get_i();
```



## 代码风格

代码风格是主观的，争论哪种好哪种不好是在浪费生命。市面上有很多自动处理代码风格的工具，选一个喜欢就行了，我们来讨论几个非自动处理的部分。

### 常量大写

### Bad:

```c
int target_i_mA = 70;
//定义目标电流为70mA
```

### Good:

```c
int TARGET_I_MA = 70;
```



## 注释

### 只有业务逻辑需要注释

代码注释不是越多越好。

### Bad:

```c
P1 = 0xFE;
//点亮第一个LED灯
delay(500);
//延时500毫秒
P1 = P1>>1;
//循环点亮第二个灯。
delay(500);
//延时500毫秒
P1 = P1>>1;
//循环点亮第三个灯。
delay(500);
//延时500毫秒
P1 = P1>>1;
//循环点亮第四个灯。
delay(500);
//延时500毫秒
P1 = P1>>1;
//循环点亮第五个灯。
```

### Good:

```c
//循环点亮五盏灯
P1 = 0xFE;
delay(500);
P1 = P1>>1;
delay(500);
P1 = P1>>1;
delay(500);
P1 = P1>>1;
delay(500);
P1 = P1>>1;
```



### 删掉注释的代码

### Bad:

```c
//int old_get_i();
//int old_get_i_update();
int new_get_i();
```

### Good:

```
int new_get_i();
```



### 不要有日记

记住你有 git！ `git log` 可以帮你干这事。

### Bad:

```
/**************
 * 2019-05-01 实现流水灯
 * 2019-05-02 实现数码管
 * 2019-05-07 实现adc
 */
```
![](https://ly-object-1259106193.cos.ap-chengdu.myqcloud.com/00613kQPgy1g30u5w6z7wj31kw18445k.jpg)


# 感谢阅读
![](https://ly-object-1259106193.cos.ap-chengdu.myqcloud.com/whut-electronic.jpg)

---

<div align =right>author:Reyunn</div>
