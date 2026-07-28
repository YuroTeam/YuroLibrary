## ArkTS_P2

《鸿蒙 HarmonyOS 应用开发基础》

## 一、课程导入

ArkTS 是鸿蒙专属强类型语言，上一章学习基础语法，本章聚焦**面向对象核心进阶**。大型项目单纯使用面向过程代码会出现复用差、维护难、耦合度高的问题，面向对象通过封装、继承、多态拆分业务，实现高内聚低耦合，本章完整覆盖面向对象全套语法、内置工具、模块化、异常、API 异步开发，最后通过倒计时实战落地时间处理能力。

## 二、本章学习目标

1. 区分面向过程、面向对象两种编程思想，理解优缺点
2. 掌握两种创建对象方式：对象字面量、class 类
3. 分清实例成员、静态成员定义与访问规则
4. 吃透类核心语法：构造方法、继承 super、访问修饰符、instanceof
5. 掌握接口定义、类实现接口、接口继承
6. 掌握泛型函数 / 泛型类 / 泛型接口、泛型约束
7. 熟练使用 Math/Number/Date/Array/String/JSON 内置对象 API
8. 掌握模块化`export/import`导出导入语法
9. 使用`try/catch/throw`捕获、主动抛出运行错误
10. 了解 ArkTS API 导入与异步 Promise、async/await 写法
11. 独立完成「计算时间差」倒计时综合案例

## 三、本章重点

1. 面向对象核心概念：对象、属性、方法、this、super
2. 类的完整语法：构造方法、继承、三大访问修饰符
3. 接口与类的配合实现，接口继承
4. 泛型基础定义与约束写法
5. Date、Array、String 高频内置方法
6. ES 模块化导出导入（默认导出 / 命名导出）
7. 异步处理：Promise、async/await

## 四、本章难点

1. this 指向、super 调用父类构造 / 父方法规则
2. public/protected/private 权限区分，封装思想落地
3. 泛型约束`T extends 接口`理解与使用
4. Date 月份 0~11 特殊规则、毫秒时间差换算
5. Promise 异步流程、async/await 异常捕获
6. 对象引用传递、数组操作原数组 / 新数组区分

## 五、分模块知识点 + 完整带注释代码笔记

### 3.1 面向过程 vs 面向对象

#### 知识点梳理

1. **面向过程**：以函数流程为核心，步骤驱动；优点无额外性能开销，适合极小项目；缺点难复用、难扩展，逻辑复杂牵一发而动全身。
2. **面向对象**：以对象为核心，对象包含**属性（特征）**、**方法（行为）**；代码易维护、复用、扩展，适合大型业务，有少量性能开销。
3. 对象本质：键值对集合，值为普通类型是属性，值为函数是方法。
4. 引用传递：对象 / 数组 / 函数赋值给新变量，仅传递内存地址，修改会互相影响。

#### 引用传递示例代码

```typescript
// 定义学生接口约束对象结构
interface Student {
  name: string;
}
// 创建字面量对象
let stu: Student = { name: "张三" };
// 引用传递：stu1和stu指向同一个内存地址
let stu1 = stu;
stu1.name = "李四";
console.log(stu.name); // 输出李四，两个变量共用对象
```

### 3.2 创建对象两种方式

#### 3.2.1 对象字面量 + 接口约束

##### 知识点

1. 字面量语法 `{key:value, ...}`，空对象`{}`
2. `interface`描述对象结构，编译做类型校验，不匹配直接报错
3. 可选属性使用`属性?:类型`

##### 完整注释代码

```typescript
// 1. 定义接口，规范学生对象必须包含的结构
interface Student {
  name: string; // 必填字符串姓名
  age: number; // 必填数字年龄
  gender?: string; // 可选性别
  study: (subject: string) => string; // 学习方法，入参科目，返回字符串
}

// 2. 字面量创建对象，类型指定为Student接口
let stu: Student = {
  name: "张三",
  age: 20,
  study(subject: string) {
    return `正在学习${subject}`;
  },
};

// 调用对象方法
console.log(stu.study("语文")); // 正在学习语文
```

#### 3.2.2 class 类创建对象

##### 知识点

1. 类是对象模板，`class`关键字，类名大驼峰
2. `new 类名()`实例化生成对象，每个实例独立属性
3. 相比字面量：多个实例共享同一份方法，节省内存

##### 完整注释代码

```typescript
// 定义学生类
class Student {
  // 实例属性，默认空字符串
  name: string = "";
  // 实例方法
  study(subject: string) {
    return `${this.name}正在学习${subject}`;
  }
}

// 实例化对象
let stu = new Student();
stu.name = "小明";
console.log(stu.study("数学")); // 小明正在学习数学
```

### 3.3 实例成员 & 静态成员

#### 3.3.1 实例成员

##### 知识点

1. 无`static`修饰的属性 / 方法为实例成员，**必须 new 实例后访问**
2. 实例内通过`this.成员`访问自身属性方法
3. 外部访问：`实例对象.成员名()`

##### 完整注释代码

```typescript
class Student {
  name: string = "小明"; // 实例属性
  study(subject: string) {
    // this代表当前实例对象
    return `${this.name}学习${subject}`;
  }
}
// 创建实例
let stu = new Student();
// 外部访问实例属性
console.log(stu.name);
// 调用实例方法
console.log(stu.study("英语"));
```

##### 拓展：Record 解决特殊键名访问

```typescript
// Record<string, 类型>支持[]下标访问对象
let obj = { "a-b": 100 } as Record<string, number>;
console.log(obj["a-b"]); // 100
```

#### 3.3.2 静态成员 static

##### 知识点

1. `static`修饰属性 / 方法，属于类本身，**无需实例，直接类名。访问**
2. 静态方法内不能使用 this（无实例），只能`类名.静态成员`

##### 完整注释代码

```typescript
class Student {
  static title: string = "学生身份"; // 静态属性
  // 静态方法
  static introduce() {
    // 静态内访问静态成员：类名.xxx
    return `标识：${Student.title}`;
  }
}
// 直接通过类调用，不需要new
console.log(Student.title); // 学生身份
console.log(Student.introduce()); // 标识：学生身份
```

### 3.4 类与接口语法细节

#### 3.4.1 构造方法 constructor

##### 知识点

1. `constructor`类专属构造函数，new 实例时自动执行，用于初始化属性
2. 构造传参简化属性赋值，可省略属性默认值
3. 链式调用：方法返回`this`，支持连续`.`调用

##### 完整注释代码

```typescript
class Student {
  name: string;
  // 构造方法，实例化自动执行
  constructor(name: string) {
    this.name = name; // 将传入参数赋值给实例属性
  }
  // 返回this，支持链式调用
  doHomework() {
    console.log(`${this.name}写作业`);
    return this;
  }
}
// 实例化传入参数，自动走构造
let stu = new Student("小红");
console.log(stu.name); // 小红
// 链式调用
stu.doHomework().doHomework();
```

#### 3.4.2 类的继承 extends

##### 知识点

1. `class 子类 extends 父类`，子类自动拥有父类所有非私有成员
2. `instanceof` 判断对象是否属于某个类（包含继承链）

##### 完整注释代码

```typescript
// 父类
class Father {
  money() {
    return "存款10万";
  }
}
// 子类继承父类
class Son extends Father {}

let son = new Son();
console.log(son.money()); // 复用父类方法：存款10万

// instanceof 判断实例关系
console.log(son instanceof Father); // true
console.log(son instanceof Son); // true
```

#### 3.4.3 super 关键字（调用父类）

##### 知识点

1. `super(参数)`：子类构造内调用父类构造，必须写在 this 之前
2. `super.方法()`：子类重写同名方法时，调用父类原方法

##### 完整注释代码

```typescript
class Father {
  a: number;
  b: number;
  constructor(a: number, b: number) {
    this.a = a;
    this.b = b;
  }
  sum() {
    return this.a + this.b;
  }
}

class Son extends Father {
  constructor(a: number, b: number) {
    // 调用父类构造，传递修改后的参数
    super(a + 1, b + 1);
  }
  // 重写父类sum方法
  sum() {
    // 调用父类sum方法，再叠加1
    return super.sum() + 1;
  }
}
let son = new Son(1, 2);
console.log(son.sum()); // (2+3)+1 = 6
```

#### 3.4.4 访问控制修饰符

表格

| 修饰符         | 同类访问 | 子类访问 | 类外部访问 |
| :------------- | :------- | :------- | :--------- |
| public（默认） | ✅        | ✅        | ✅          |
| protected      | ✅        | ✅        | ❌          |
| private        | ✅        | ❌        | ❌          |

##### 完整注释代码

```typescript
class Student {
  public name = "小明"; // 全部可访问
  protected phone = "13800138000"; // 同类、子类可用
  private money = 5000; // 仅当前类内可用

  test() {
    // 同类全部可访问
    console.log(this.name, this.phone, this.money);
  }
}

class StuSub extends Student {
  test() {
    console.log(this.name); // public ✅
    console.log(this.phone); // protected ✅
    // console.log(this.money); // private 编译报错
  }
}

let s = new Student();
console.log(s.name); // public ✅
// console.log(s.phone / s.money); // protected/private 外部报错
```

##### 拓展：面向对象三大特性

1. **封装**：private/protected 隐藏内部细节，仅开放 public 接口
2. **继承**：复用父类代码，减少重复
3. **多态**：子类重写父类同名方法，同一方法不同对象执行不同逻辑

#### 3.4.5 类实现接口 implements

##### 知识点

1. `class 类 implements 接口1,接口2`，一个类可实现多个接口
2. 类必须完整实现接口所有必填属性 / 方法；`?`代表可选，可省略

##### 完整注释代码

```typescript
// 定义学生接口规范
interface IStudent {
  name: string;
  age?: number; // 可选属性
  introduce(sub: string): string;
}

// 类实现接口，必须补齐所有必填成员
class Student implements IStudent {
  name: string = "小刚";
  // 实现接口规定的方法
  introduce(sub: string): string {
    return `学习：${sub}`;
  }
}
let s = new Student();
console.log(s.introduce("物理"));
```

#### 3.4.6 接口继承 extends

##### 知识点

接口可以继承其他接口，子接口自动合并父接口所有约束

```typescript
interface Point2D {
  x: number;
  y: number;
}
// 子接口继承2D坐标，新增z轴
interface Point3D extends Point2D {
  z: number;
}
// 必须包含x,y,z三个属性
let p: Point3D = { x: 1, y: 2, z: 3 };
```

### 3.5 泛型

#### 知识点

泛型``类型占位符，定义时不固定类型，使用时指定，一套代码兼容多种类型。

1. 泛型函数 ``
2. 泛型类
3. 泛型接口
4. 泛型约束 `T extends 接口` 限制类型范围

#### 1）泛型函数

```typescript
// T、U为泛型占位符
function demo<T, U>(a: T, b: U): U {
  return b;
}
// 调用显式指定类型 T=number U=string
let res = demo<number, string>(10, "测试");
console.log(res);
```

#### 2）泛型类

```typescript
class Box<T, U> {
  val1: T;
  val2: U;
  constructor(a: T, b: U) {
    this.val1 = a;
    this.val2 = b;
  }
}
// 实例化指定泛型类型
let box = new Box<number, string>(100, "泛型");
```

#### 3）泛型接口

```typescript
interface Data<T> {
  id: number;
  info: T;
}
// 使用泛型接口，T为字符串数组
let user: Data<string[]> = {
  id: 1,
  info: ["张三", "李四"],
};
```

#### 4）泛型约束（限制 T 必须包含指定结构）

```typescript
interface HasNum {
  num: number;
}
// T必须满足HasNum接口，必须有num属性
function getNum<T extends HasNum>(obj: T): number {
  return obj.num;
}
console.log(getNum({ num: 999 }));
```

### 3.6 常用内置对象

#### 3.6.1 Math 数学工具（全部静态属性 / 方法，无需实例）

```typescript
// 圆周率
console.log(Math.PI);
// 绝对值
console.log(Math.abs(-10));
// 最大最小值
console.log(Math.max(1, 99, 20));
console.log(Math.min(1, 99, 20));
// 幂、平方根
console.log(Math.pow(3, 4)); // 3^4=81
console.log(Math.sqrt(81)); // 9
// 向上/向下取整
console.log(Math.ceil(3.1)); //4
console.log(Math.floor(3.9)); //3
// 四舍五入（负数特殊：-2.5 输出-2）
console.log(Math.round(-2.5));
// 随机整数 min~max 包含两端
function randomInt(min: number, max: number) {
  return Math.floor(Math.random() * (max - min + 1) + min);
}
console.log(randomInt(1, 10));
```

#### 3.6.2 Number 数字处理

```typescript
// toFixed 保留小数位
let n = 1.35;
console.log(n.toFixed(1)); // 1.4
// 字符串转数字
let num = Number("666");
console.log(num.toString()); // 转字符串
```

#### 3.6.3 Date 日期时间（重点难点）

##### 关键规则

1. 月份 0~11：0=1 月，11=12 月
2. 星期 0 = 周日，1 = 周一...6 = 周六
3. `getTime()` / `Date.now()` 获取 1970 至今毫秒时间戳

```typescript
// 1. 获取当前时间
let now = new Date();
// 2. 指定时间 年,月(0开始),日,时,分,秒
let target = new Date(2026, 5, 23, 12, 0, 0);

// 获取年月日时分秒星期
let year = now.getFullYear();
let month = now.getMonth() + 1;
let day = now.getDate();
let weekArr = ["周日", "周一", "周二", "周三", "周四", "周五", "周六"];
let week = weekArr[now.getDay()];
console.log(`${year}年${month}月${day} ${week}`);

// 时间戳
console.log(Date.now()); // 当前毫秒
console.log(target.getTime()); // 指定时间毫秒
```

#### 3.6.4 Array 数组全套 API

```typescript
// 创建数组
let arr = [10, 20, 30];
let arr2 = new Array("苹果", "香蕉");

// 长度
console.log(arr.length);

// 遍历
arr.forEach((val, idx) => console.log(idx, val));
let newArr = arr.map((v) => v * 2); // 返回新数组
for (let item of arr) console.log(item);

// 增删（修改原数组）
arr.push(40); // 末尾加
arr.unshift(0); // 开头加
arr.pop(); // 删除末尾
arr.shift(); // 删除开头
arr.splice(1, 1); // 索引1删除1个元素

// 反转、排序
arr.reverse();
let numArr = [3, 1, 5];
numArr.sort((a, b) => a - b); // 升序
numArr.sort((a, b) => b - a); // 降序

// 查找索引
console.log(arr.indexOf(20));
console.log(arr.lastIndexOf(20));

// 转换字符串
console.log(arr.join("-"));

// 截取、拼接、填充
arr.slice(0, 2); // 新数组，不修改原
arr.concat([100, 200]);
arr.fill(0, 1, 3);
```

#### 3.6.5 String 字符串

```typescript
let str = "HelloArkTS";
console.log(str.length);
// 查找索引
console.log(str.indexOf("A"));
// 取字符
console.log(str.charAt(0), str[0]);
// 大小写
console.log(str.toUpperCase(), str.toLowerCase());
// 替换
console.log(str.replace("ArkTS", "鸿蒙"));
// 截取、分割
console.log(str.slice(0, 5));
console.log(str.split("l"));
// 前后填充
console.log(str.padStart(15, "*"));
```

#### 3.6.6 JSON 数据序列化 / 反序列化

```typescript
interface User {
  name: string;
  age: number;
}
let user: User = { name: "张三", age: 20 };
// 对象转JSON字符串
let jsonStr = JSON.stringify(user);
console.log(jsonStr);
// JSON字符串转回对象
let parseUser = JSON.parse(jsonStr) as User;
console.log(parseUser.name);
```

### 3.7 模块化 export /import

#### 1）导出模块 modules.ets

```typescript
// 命名导出（多个）
export class B {}
export class C {}
// 默认导出（一个文件只能一个default）
export default class A {
  name = "A类";
}
```

#### 2）导入使用页面 Index.ets

```typescript
// 默认导入 + 按需命名导入
import A, { B, C } from "../modules";
console.log(new A().name);

// 导入重命名 as
import { B as BClass } from "../modules";
console.log(new BClass());
```

### 3.8 错误处理 try /catch/throw

```typescript
// 捕获内置异常
try {
  // 非法JSON，会抛出语法错误
  JSON.parse("{name:'小明'}");
} catch (err) {
  console.log("捕获异常：", err);
}

// 手动抛出自定义错误
try {
  throw new Error("自定义业务错误：参数不能为空");
} catch (e) {
  console.log(e.message);
}
```

### 3.9 ArkTS API 异步处理 Promise & async/await导入弹窗 API

```typescript
// 两种导入方式任选其一
import promptAction from "@ohos.promptAction";
// import { promptAction } from "@kit.ArkUI";
```

#### 方式 1：then/catch 链式处理 Promise

```typescript
function showDialogByThen() {
  promptAction
    .showDialog({
      message: "确认支付？",
      buttons: [{ text: "取消" }, { text: "确认" }],
    })
    .then((res) => {
      if (res.index === 1) console.log("点击确认");
      else console.log("点击取消");
    })
    .catch(() => {
      console.log("弹窗被关闭");
    });
}
```

#### 方式 2：async/await 同步写法（推荐）

```typescript
async function showDialogByAwait() {
  try {
    let res = await promptAction.showDialog({
      message: "确认支付？",
      buttons: [{ text: "取消" }, { text: "确认" }],
    });
    if (res.index === 1) console.log("确认付款");
    else console.log("取消付款");
  } catch {
    console.log("弹窗关闭");
  }
}
```

### 3.10 综合案例：计算时间差（倒计时）

#### 需求

输入目标时间戳，计算当前到目标的天数、时、分、秒，返回`X天X时X分X秒`格式。

#### 完整带注释代码

```typescript
/**
 * 计算当前时间与目标时间的差值
 * @param targetTime 目标时间毫秒戳
 * @returns 格式化时间差字符串
 */
function getTimeDiff(targetTime: number): string {
  // 获取当前时间戳
  let nowTime = Date.now();
  // 总毫秒差
  let diffMs = targetTime - nowTime;
  if (diffMs <= 0) return "活动已开始/结束";

  // 单位换算
  const oneDay = 24 * 60 * 60 * 1000; // 一天毫秒
  const oneHour = 60 * 60 * 1000; // 一小时
  const oneMin = 60 * 1000; // 一分钟
  const oneSec = 1000; // 一秒

  // 计算天、时、分、秒
  let day = Math.floor(diffMs / oneDay);
  let remain1 = diffMs % oneDay;
  let hour = Math.floor(remain1 / oneHour);
  let remain2 = remain1 % oneHour;
  let min = Math.floor(remain2 / oneMin);
  let sec = Math.floor((remain2 % oneMin) / oneSec);

  return `距离活动还有${day}天${hour}时${min}分${sec}秒`;
}

// 测试：目标时间 2026-06-25 00:00:00
let targetDate = new Date(2026, 5, 25, 0, 0, 0);
let res = getTimeDiff(targetDate.getTime());
console.log(res);
```

## 六、本章整体总结

1. 编程思想分为面向过程（流程函数）、面向对象（对象封装属性行为），鸿蒙项目优先使用面向对象开发。
2. 创建对象两种：字面量 + 接口（简单对象）、class 类（批量实例，复用方法）。
3. 成员分实例（new 后访问 this）、静态（类名 static）；三大访问修饰符实现封装。
4. 类通过`extends`继承、`super`调用父类；`implements`实现接口，接口可继承。
5. 泛型解决多类型复用，``占位符，`T extends`做类型约束。
6. 内置工具对象：Math 数值、Number 小数、Date 时间、Array 数组、String 字符串、JSON 序列化，是业务高频工具。
7. 模块化`export/import`拆分代码，分默认导出、命名导出，实现文件复用。
8. `try/catch`捕获运行异常，`throw`主动抛出错误，避免程序崩溃。
9. ArkTS 系统 API 多为异步 Promise，可使用 then 链式或 async/await 简化异步代码。
10. Date 时间戳换算可以实现倒计时、时间差等常见页面业务。

## 七、课后作业

### 基础作业

1. 手写对比面向过程、面向对象的区别表格，举生活案例说明。
2. 分别用字面量、class 创建「图书对象」，包含书名、价格、阅读方法。
3. 给图书类添加静态属性`分类：书籍`、静态打印分类方法。
4. 创建动物父类，猫、狗子类继承，重写叫声方法，使用 super 调用父类。
5. 定义泛型工具函数，实现数组去重，兼容数字 / 字符串数组。

### 进阶作业

1. 使用 Date 编写倒计时页面逻辑，输入指定年月日，实时打印剩余时分秒。
2. 封装工具模块`utils.ets`，导出数组排序、时间格式化函数，在页面导入调用。
3. 封装 JSON 读取工具，增加 try/catch 捕获非法 JSON 异常。
4. 调用 promptAction 弹窗 API，使用 async/await 实现点击按钮弹出确认框，打印用户选择。
5. 定义用户接口，编写类实现该接口，使用 protected 隐藏手机号，仅开放获取姓名的公开方法。

