## ArkTS_P1

## 《鸿蒙 HarmonyOS 应用开发基础》

## 知识目标

1. 熟悉 ArkTS 的概念，能够说出什么是 ArkTS，以及 ArkTS 与 JavaScript、TypeScript 的关系
2. 掌握调试输出，能够使用`console.log()`语句输出信息
3. 掌握注释的使用方法，能够合理运用单行注释、多行注释增强代码可读性
4. 掌握变量、常量和数据类型，能够使用变量、常量存储数据
5. 掌握运算符，能够灵活运用运算符完成运算
6. 掌握选择结构语句，能够根据实际需求选择合适的分支语句
7. 掌握循环语句，能够根据实际需求选择合适的循环结构
8. 掌握跳转语句，能够灵活使用`continue`/`break`实现流程跳转
9. 掌握数组和枚举，能够使用数组、枚举存储批量 / 预设数据
10. 熟悉函数概念，理解函数作用
11. 掌握内置函数、自定义函数编写与调用
12. 掌握函数作为值传递（变量、参数、返回值、数组元素）
13. 掌握箭头函数定义与调用
14. 熟悉变量作用域、闭包原理并规范使用

## 章节概述

ArkTS 是鸿蒙应用开发主力语言，基于 TypeScript 扩展优化：保留 TS 基础语法，强化静态类型检查，减少动态类型开销，提升应用 性能、降低移动端功耗。本章完整讲解 ArkTS 基础语法。

------

# 2.1 初识 ArkTS

## 2.1 核心关系：JS → TS → ArkTS

ArkTS 并非全新语言，华为在 TypeScript 基础上改造，有 JS/TS 基础可快速上手鸿蒙开发。

![](hm-pic\1782040604707.png)

目标：熟悉ArkTS的概念，能够说出什么是ArkTS，以及ArkTS与JavaScript、TypeScript的关系

### 1. JavaScript

- 定位：Web 前端原生脚本语言，标准由 Ecma 国际制定，规范文档 ECMA-262，语言标准称为 ECMAScript，JS 是 ECMAScript 的实现扩展。
- 用途：网页交互（轮播、表单）、后端、桌面、移动端开发。
- 版本：ES6 (2015) 为里程碑，每年更新新版本（如 ES2024）。

### 2. TypeScript（JS 超集）

由微软开源推出，**完全包含 JS 所有特性**，新增核心能力：

- 静态类型检查：编码阶段捕获错误，减少 异常
- 接口、泛型、严格语法约束
- TS 代码可编译为 JS，兼容所有 JS  环境
- 适合大型项目，提升代码可维护性、可读性

### 3. ArkTS（鸿蒙专属语言）

面向移动设备优化，解决 JS/TS  慢、功耗高问题：

1. 兼容绝大部分 TS 语法，TS 开发者无缝迁移
2. 严格限制动态类型，**禁用 any 类型**，消除 时类型判断开销
3. 代码提前编译优化，应用启动更快、设备功耗更低

------

# 2.2 调试输出和注释

## 2.1 调试输出 console.log ()

### 作用

打印日志，在 DevEco Studio 底部「日志」面板查看，用于代码调试。

### 语法

```typescript
console.log(参数1, 参数2, ...);
```

- 参数：多个内容用英文逗号分隔，第一个必须是字符串，其余类型自动转字符串
- 行尾分号可省略（换行分隔语句）

## 2.2 注释

注释仅用于说明代码逻辑，程序执行时会被忽略，分两种：

1. **单行注释 //**

```typescript
console.log('你好'); // 打印提示文本
```

1. **多行注释 /\* \*/**

```typescript
/*
调试输出示例代码
作者：xxx
*/
console.log('你好');
```

------

# 2.3 变量、常量和数据类型

## 2.3.1 变量 let

### 概念

内存中开辟的可变存储空间，存储程序临时数据，使用前必须声明。

### 基础语法

```typescript
// 声明变量（指定类型）
let 变量名: 类型;
// 声明并赋值（初始化）
let 变量名: 类型 = 值;
// 省略类型，ArkTS自动类型推断
let 变量名 = 值;
// 联合类型：支持多种类型
let a: string | number;
```

> 注意：ArkTS**不支持 any 任意类型**，强制静态类型约束。

### 变量命名规则

#### 强制规则

1. 不能数字开头，不能包含`+-*/`等运算符
2. 严格区分大小写 `age` ≠ `Age`
3. 禁止使用 ArkTS 关键字（let、if、while 等）

#### 推荐规范

1. 仅使用字母、数字、下划线`_`、美元符`$`
2. 语义化命名（age 年龄、score 分数）
3. 多单词：下划线命名`user_name` / 小驼峰`userName`

### 多变量声明

```typescript
// 一行声明多个变量
let a: string, b: number;
// 一行声明并赋值
let name: string = "小明", age: number = 18;
```

### 示例

```typescript
let student01: string = '小明';
let student02 = '小智';
console.log(student01); // 小明
console.log(student02); // 小智
```

```typescript
aboutToAppear()  {

      console.log("测试打印，能看到这条就说明日志面板没问题");

      // 1:完整写法：指定类型+赋值
    let student1:string='小明';

    // 2：简单写法:自动推断类型+赋值
    let student2='小红';

    // 3：先声明变量，后单独赋值
    let student3:string;
    student3='小草'

    //4：一行声明多个变量，只声明，不赋值
    let num:number,num2:number;
    num=1;
    num2=2;

    //5:一行声明多个变量并同时赋值
    let scoreMath:number=90,scoreChinese:number=87;

    // 使用console.log()打印所有的变量
    console.log('student1是:',student1)
    console.log('student2是:',student2)
    console.log('student3是:',student3)
    console.log('num=',num)
    console.log('num2=',num2)
    console.log('scoreMath=',scoreMath)
    console.log('scoreChinese=',scoreChinese)

  }
```





## 2.3.2 常量 const

常量分为两类：

1. **字面量常量**：源码固定值（`10`、`"abc"`、`true`、`[1,2]`、`{}`），不可修改

2. **const 关键字常量**：不可重新赋值，习惯全大写命名

3. **常量两个硬性规则**

   ###### 规则 1：声明时必须直接赋值，不能分开写

   ```
   // 正确
   const PI: number = 3.1415926;
   
   // 错误，直接爆红报错
   const num: number;
   num = 100;
   ```

   ###### 规则 2：赋值后绝不允许二次修改

   ```
   const name: string = "小草";
   name = "小明"; // 报错！常量禁止重新赋值
   ```

   ##### 3. 和 let 直观对比

   ```
   // let 变量，可以随便改
   let score: number = 80;
   score = 95;
   
   // const 常量，固定死
   const CLASS_NAME: string = "一班";
   CLASS_NAME = "二班"; // 代码报错
   ```

   ##### 4. 小补充（ArkTS 特有小细节）

   如果 `const` 存的是数组、对象，**里面内容能改，但是变量本身不能重新赋值**

   ```
   const arr: number[] = [1,2,3];
   arr.push(4); // 允许，数组内部内容可变
   arr = [6,6,6]; // 报错，不能给常量整体换新数组
   ```

   ##### 极简总结

   ArkTS 里 `const` 就是常量，用法、限制和 JS/TS 完全统一，不会变的数据就用它。

   1. > 特殊说明：const 保存数组 / 对象时，**不能重新赋值变量本身，但可以修改内部元素 / 属性**。

## 2.3.3 常用基础数据类型

### 1. string 字符串

三种引号包裹，支持模板字符串`${变量}`拼接：

```
let stu1: string = '小明';
let stu2: string = "小智";
let introduce: string = `${stu1}和${stu2}是好朋友`;
console.log(introduce); // 小明和小智是好朋友
```

#### 常用转义字符

| 转义符 | 含义               |
| :----- | :----------------- |
| '      | 单引号             |
| "      | 双引号             |
| `      | 反引号             |
| \n     | 换行               |
| \t     | 制表符             |
| \      | 反斜杠             |
| \u597d | Unicode 字符（好） |

### 2. number 数字

包含整数、浮点数、科学计数法、特殊值

1. 进制整数



```
let bin: number = 0b11010; // 二进制 26
let oct: number = 0o32;    // 八进制 26
let dec: number = 26;      // 十进制 26
let hex: number = 0x1a;    // 十六进制 26
```

1. 浮点数 & 科学计数法

```
let f1: number = -3.12;
let f2: number = 3.14E5; // 3.14 * 10^5
let f3: number = 7.35E-5;
```

1. 特殊值

- `Infinity` 无穷大
- `-Infinity` 无穷小
- `NaN` 非数字（非法运算结果）

### 3. boolean 布尔

仅两个值 `true`(真) / `false`(假)，用于逻辑判断：

typescript

```
let flag: boolean = true;
```

### 4. null 空

仅有`null`一个值，表示无对象引用：

typescript

```
let empty: null = null;
```

### 5. undefined 未定义

变量声明未赋值时默认值：

```
let num: undefined;
let num2: undefined = undefined;
```

### 6. void 空

主要用于标记**无返回值函数**。

### 7. object 对象（引用类型）

内存保存引用地址，多个变量可共用同一对象，包含：字面量对象、数组、函数、类实例、枚举等。

------

# 2.4 运算符

## 2.4.1 算术运算符

表格

| 运算符 | 作用         | 示例                       |
| :----- | :----------- | :------------------------- |
| `+`    | 加           | 3+3=6                      |
| `-`    | 减           | 6-3=3                      |
| `*`    | 乘           | 3*5=15                     |
| `/`    | 除           | 8/2=4                      |
| `%`    | 取模（余数） | 5%7=5                      |
| `**`   | 幂运算       | 4**2=16                    |
| `++`   | 自增         | 前置先加后用，后置先用后加 |
| `--`   | 自减         | 前置先减后用，后置先用后减 |

### 自增自减示例

```typescript
let a = 2, b = 2;
console.log(++a); // 3 前置：先自增再返回
console.log(b++); // 2 后置：先返回再自增
```

### 注意事项

1. 四则运算遵循「先乘除后加减」，小括号提升优先级
2. 取模正负由**左侧数字**决定：`-8 % 7 = -1`、`8 % -7 = 1` 
3. 浮点数运算存在精度误差，建议转整数计算后还原

## 2.4.2 字符串运算符 `+`

```
console.log('小智' + '，' + 18); // 小智，18
```

## 2.4.3 赋值运算符

```
=`、`+=`、`-=`、`*=`、`/=`、`%=`、`**=`、`<<=`、`>>=`、`&=`、`^=`、`|=
```

```
let num = 5;
num += 3; // num = num + 3 → 8
num *= 2; // num = num * 2 → 16
```

## 2.4.4 比较运算符

返回`true/false`，ArkTS 禁止不同类型使用`==`（直接报错）

表格

| 运算符            | 说明                               |
| :---------------- | :--------------------------------- |
| `>` `<` `>=` `<=` | 大小比较                           |
| `==` `!=`         | 相等 / 不等（类型不同直接报错）    |
| `===` `!==`       | 全等 / 不全等（值 + 类型同时对比） |

## 2.4.5 逻辑运算符

### 规则

以下值判定为**假**：`false、0、''、null、undefined、NaN`；其余为真

表格

| 运算符  | 作用                                     |      |                                          |
| :------ | :--------------------------------------- | :--- | :--------------------------------------- |
| `&&` 与 | 左边为假直接短路，返回左边；否则返回右边 |      |                                          |
| `       |                                          | ` 或 | 左边为真直接短路，返回左边；否则返回右边 |
| `!` 非  | 取反                                     |      |                                          |

### 短路示例

```
let a = 1;
false && a++; // 短路，a不会自增
true || a++;  // 短路，a不会自增
console.log(a); // 1
```

## 2.4.6 三元运算符 `条件 ? 表达式1 : 表达式2`

条件为 true 执行表达式 1，false 执行表达式 2：

```
let age = 18;
let res = age >= 18 ? "成年" : "未成年";
```

## 2.4.7 类型检测 typeof

返回数据类型字符串：

```
typeof 23; // "number"
typeof "abc"; // "string"
typeof []; // "object"
```

## 2.4.8 运算符优先级

1. `()` 括号优先级最高

2. 单目运算符（`! ++ -- typeof`）

3. 幂运算 `**`

4. 乘除取模 `* / %`

5. 加减 `+ -`

6. 比较 `< > <= >=`

7. 相等 `=== !== == !=`

8. 逻辑与`&&`、逻辑或`||`

9. 三元 `?:`

10. 赋值 

    ```
    = +=
    ```

    等同优先级：左结合（从左往右算），赋值 / 三元为右结合

------

# 2.5 流程控制

## 2.5.1 选择（分支）结构

### 1. if 单分支

```
if(条件){
  代码块;
}
```

### 2. if...else 双分支

```
if(条件){
  条件成立执行
}else{
  条件不成立执行
}
```

### 3. if...else if...else 多分支

```
let score = 78;
if(score >=90){
  console.log("优秀");
}else if(score >=80){
  console.log("良好");
}else if(score >=70){
  console.log("中等");
}else if(score >=60){
  console.log("及格");
}else{
  console.log("不及格");
}
```

### 4. switch 等值多分支

仅匹配**固定等值**，`break`防止穿透，`default`兜底

```
switch(表达式){
  case 值1:
    逻辑;
    break;
  case 值2:
    逻辑;
    break;
  default:
    默认逻辑;
}
```

## 2.5.2 循环结构

### 1. for 循环（已知循环次数首选）

```
// 初始化;循环条件;自增
for(let i=1; i<=100; i++){
  console.log(i);
}
```

### 2. while 循环（未知循环次数）

先判断，条件成立再执行循环体

```
let i = 1;
while(i <= 100){
  console.log(i);
  i++;
}
```

### 3. do...while 循环

**先执行一次循环体，再判断条件**，至少  1 次

```
let j = 1;
do{
  console.log(j);
  j++;
}while(j <= 100);
```

## 2.5.3 跳转语句

### 1. continue 跳出本次循环

跳过当前剩余代码，直接进入下一轮循环

```
for(let i=1;i<=6;i++){
  if(i === 2) continue;
  console.log(`第${i}个桃子`);
}
```

### 2. break 跳出整个循环

终止全部循环；支持标签跳出多层嵌套循环

```
outer: // 标签
for(let i=0;i<10;i++){
  for(let j=0;j<5;j++){
    if(i===5) break outer; // 直接跳出外层循环
  }
}
```

------

# 2.6 数组和枚举

## 2.6.1 数组 Array

### 概念

存储一组同类型数据，索引从 0 开始，格式 `元素类型[]` / `Array<类型>`

### 1. 一维数组

```
// 定义
let fruits: string[] = ["苹果","香蕉"];
let scores: Array<number> = [98,95,100];
// 访问/修改
console.log(fruits[0]); // 苹果
fruits[2] = "草莓"; // 新增元素
fruits[1] = "菠萝"; // 修改元素
```

### 2. 二维数组（数组嵌套数组）

适合表格、矩阵数据，双层索引访问`arr[i][j]`

```
let arr: number[][] = [[89,98],[95,83]];
console.log(arr[0][0]); // 89
```

## 2.6.2 枚举 enum

预定义一组固定常量，统一管理固定选项（颜色、状态、星期）

### 语法

```
// 自动赋值：0,1,2
enum Color{Red,Green,Blue}
// 手动赋值
enum ColorSet{
  Red = "#f00",
  Green = "#0f0",
  Blue = "#00f"
}
// 使用
console.log(ColorSet.Red); // #f00
```

------

# 2.7 函数

## 2.7.1 函数作用

封装重复逻辑，提高代码复用、降低维护成本；分为**内置函数**、**自定义函数**。

## 2.7.2 自定义函数

### 定义语法



```
function 函数名(参数1:类型, 参数2?:类型, 参数3:类型=默认值):返回值类型{
  // 函数体
  return 返回数据;
}
```

- 可选参数 `参数?` / 默认参数 `参数=xxx`，必须放末尾
- 无返回值标记 `:void`

### 示例

```
// 求和函数
function sum(num1:number, num2:number):number{
  return num1 + num2;
}
// 调用
console.log(sum(1,2)); // 3
// 接收返回值
let res = sum(10,20);
```

## 2.7.3 函数作为值使用

函数属于对象，可赋值变量、传参、返回、存入数组；用`type`简化函数类型

```
// 定义函数类型别名
type AddFunc = (a:number,b:number)=>number;
function sum(a:number,b:number){return a+b;}
// 赋值变量
let fn: AddFunc = sum;
console.log(fn(1,2));
// 函数作为参数（回调函数）
function calc(func:AddFunc){
  return func(10,20);
}
console.log(calc(sum));
// 函数存入数组
let arr = [sum];
console.log(arr[0](5,6));
```

## 2.7.4 箭头函数（简化函数）

### 基础写法

```
// 完整写法
let sum = (a:number,b:number)=>{
  return a + b;
};
// 单行省略return、大括号
let sum2 = (a:number,b:number)=>a+b;
// 自执行箭头函数
((a:number,b:number)=>{
  console.log(a+b);
})(1,2);
```

## 2.7.5 常用内置函数

1. `parseInt()`：提取整数，舍弃小数，自动识别进制

```
parseInt("100.56"); // 100
parseInt("0xF"); // 15
```

1. `parseFloat()`：提取浮点数

```
parseFloat("3.14e-2"); // 0.0314
```

1. 定时器

- `setTimeout(回调,毫秒)`：延迟执行一次
- `setInterval(回调,毫秒)`：周期性重复执行
- `clearTimeout(标识)` / `clearInterval(标识)`：清除定时器

```
let timer = setInterval(()=>{
  console.log("定时输出");
  clearInterval(timer); // 执行一次后销毁
},1000);
```

------

# 2.8 变量的作用域和闭包

## 2.8.1 变量作用域

`let`声明变量拥有**块级作用域**（`{}`内有效：if、for、函数）

1. 内部可访问外部变量，外部不能访问内部变量

```
let outer = 10;
function test(){
  let inner = 20;
  console.log(outer); // 可访问外部
}
test();
console.log(inner); // 报错，外部无法访问内部
```

1. 作用域向内传递，多层嵌套均可读取外层变量

## 2.8.2 闭包 Closure

### 定义

内层函数访问外层函数的变量 / 函数，整体构成闭包；内层称为闭包函数。

### 特性

外层函数执行完毕后，变量不会销毁，持续保存在内存

### 示例（计数器）

```
function createCount(){
  let num = 0;
  // 返回闭包函数
  return ()=>{
    num++;
    console.log(`第${num}次调用`);
  }
}
let count = createCount();
count(); // 第1次
count(); // 第2次
count(); // 第3次
```

------

# 2.9 阶段案例：统计每个学生总成绩

## 需求

学生成绩二维数组：

- 小明：语文 90，数学 99，英语 95

- 小智：语文 98，数学 100，英语 85

- 小红：语文 95，数学 96，英语 97

  

  遍历二维数组，计算每位学生总分。

## 本章小结

本章覆盖 ArkTS 全部基础语法：

1. ArkTS、JS、TS 三者关系

2. 日志输出、代码注释

3. 变量、常量、7 大基础数据类型

4. 全部运算符与优先级

5. 分支、循环、跳转流程控制

6. 一维 / 二维数组、枚举

7. 自定义函数、箭头函数、函数传参、内置定时器 / 转换函数

8. 变量块级作用域、闭包原理

   





开发文档：
https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/foreword

# 一、修改app图标 以及 app名称

~~~ typescript
// icon: app图标
// label app名字
// startWindowIcon  窗口启动图标
"abilities": [
      {
        "icon": "$media:layered_image",
        "label": "$string:EntryAbility_label",
        "startWindowIcon": "$media:startIcon",
      }
    ],
~~~


# 二、透明度对照表

十六进制值	透明度（约）	示例
#00	完全透明	#002d2b29
#40	25%	#402d2b29
#80	50%	#802d2b29
#C0	75%	#C02d2b29
#FF	完全不透明（默认）	#FF2d2b29






#   ???   配置导航栏菜单
~~~ typescript
import { Layout } from './Layout';
import { Start } from './Start';
import { User } from './User';

interface ITabs  {
  title: string
  imgUrl: string
}
let tabList: ITabs[] = [
  {
    title: '推荐',
    imgUrl: 'app.media.tuijian_0',
  },
  {
    title: '发现',
    imgUrl: 'app.media.faxian_0',
  },
  {
    title: '漫游',
    imgUrl: 'app.media.manyou_0',
  },
  {
    title: '我的',
    imgUrl: 'app.media.wode_0',
  }
]

@Entry
@Component
struct NavigationExample {
  @Provide('pageInfos') pageInfos: NavPathStack = new NavPathStack()
  @State tabsClickIndex: number = 0

  build() {
    Navigation(this.pageInfos) {
      Tabs({ barPosition: BarPosition.End }) {
        ForEach(tabList, (item:ITabs, index: number ) => {
          TabContent() {
            if (index === 0) {
              Layout()
            } else if (index === 1) {
              Start()
            } else {
              User()
            }
          }.tabBar(
            this.buildTabItem(item.title, item.imgUrl, index)
          )
        })
      }
      .barWidth('100%')
      .barHeight(70)
      .barMode(BarMode.Fixed)  // 固定模式，不滚动
      .onChange((index: number) => {
        // 存储当前点击的索引
        this.tabsClickIndex = index
      })
    }
    .title('我的应用')
    // .edgeSwipe(false)
    .hideBackButton(true)
  }
  
//   自定义tabContent栏
  @Builder
  buildTabItem(text: string, icon: string, index: number) {
    Column() {
      if (index === this.tabsClickIndex) {
        Image($r(`${icon}2`))
          .width(24)
          .backgroundColor('#d81e06')
          .height(24)
          .borderRadius(12)
          .padding(2)
        Text(text)
          .fontSize(12)
          .margin({ top: 5 })
          .fontColor('#d81e06')
      } else {
        Image($r(`${icon}1`))
          .width(24)
          .height(24)
        Text(text)
          .fontSize(12)
          .margin({ top: 5 })
      }

    }
    .width('100%')
    .height('100%')
    .justifyContent(FlexAlign.Center)
  }
}
~~~



