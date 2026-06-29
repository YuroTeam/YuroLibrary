# ArkTS_P3

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
  name: string
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

   ##### 语法

   ```typescript
   // 1. 定义接口，规范学生对象必须包含的结构
   interface 接口名{
   	属性1：数据类型
   	属性2：数据类型
   	可选属性？：数据类型
       
   	方法名：（参数：数据类型）=> 返回的数据类型
       //注意：逗号隔开
   }
   // 2. 字面量创建对象，
   let  对象名:接口名{
         属性1：值1，
         属性2：值2，
         //需要逗号隔开
         //与定义接口中的属性和方法一一对应
         
         方法名：（参数：数据类型）=> 返回的数据类型
   }
   // 调用对象属性方法
   对象名.属性
   对象名.方法
   //修改属性值
   对象名.属性=值
   ```

   

##### 完整注释代码

```typescript
// 1. 定义接口，规范学生对象必须包含的结构
interface Student { 
  name: string // 必填字符串姓名
  age: number // 必填数字年龄
  gender?: string // 可选性别
  study: (subject: string) => string // 学习方法，入参科目，返回字符串
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

   

##### 语法

```typescript
//类
//定义类
class 类名（首字母大写）{
	字段名1:类型='初始值'
	//如果不想写初始值，可以使用可选属性
	字段名2?:类型
    方法名(){}
}

//根据类实例化对象
let 对象[变量]名=new 类名（）

//获取对象的方法和属性
对象名.属性
对象名.方法
//修改属性值
对象名.属性=修改的值
```

##### 完整注释代码

```typescript
// 定义学生类
class Student {
  // 实例属性，默认空字符串
  name: string = ""
  // 实例方法
  study(subject: string) {
    return `${this.name}正在学习${subject}`
  }
}

// 实例化对象
let stu = new Student();
stu.name = "小明";
console.log(stu.study("数学")); // 小明正在学习数学
```

interface和class创建对象的区别

interface接口：只是一份规则说明说

​                          只规定必须有哪些属性，哪些方法，没有实际代码逻辑，不能直接new创建实体对象

class类：完整模板（生产图纸）

​               既要定义属性，又能 编写方法实现逻辑，自带构造函数，能直接new创建实体对象



接口本身是不能真正造出对象，只能约束对象的格式

实操作业一：接口interface专项（创建接口对象）

作业要求

1. 定义接口  Student ，包含：只读学号id、姓名name、可选年龄age、方法showInfo()
2. 方式1：字面量创建两个接口对象，一个带年龄，一个不带年龄
3. 方式2：新建类  StuModel  使用implements实现Student接口，通过new创建对象
4. 三个对象分别调用showInfo()输出信息

思考题（写完代码回答）

1. 接口能不能直接 new Student() 创建对象？
2. 实现接口的类，必须补齐接口里所有属性和方法吗？

实操作业二：class类专项（类创建对象+构造函数）

作业要求

1. 定义  Phone  类：公开brand品牌、私有password解锁密码
2. 编写构造函数，一次性传入品牌、密码完成初始化
3. 编写方法printBrand()，只打印手机品牌，不输出私有密码
4. new 创建2台手机对象，分别调用printBrand()
5. 尝试在类外部直接打印密码，观察报错并说明原因



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
语法：
class 类名{
    字段名1：属性（不需要写初始值）
    字段名2：属性（不需要写初始值）
    字段名3：属性（不需要写初始值）
    字段名4：属性（不需要写初始值）
    constructor(字段名1:属性,字段名2:属性,字段名3:属性,字段名3:属性){
     this.属性1=字段名1（一般属性=字段名）
     this.属性2=字段名2（一般属性=字段名）
     this.属性3=字段名3（一般属性=字段名）
     this.属性4=字段名4（一般属性=字段名）
        this.方法名()
    }
    方法名(){
         方法体
         return
     }
}

```



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

**没有构造函数的缺点：**

1：每次新建对象都要手动赋值，代码重复且冗余

2：容易忘记赋值，出现空数据或者错误数据

**构造函数的作用：**

1：创建对象时自动执行，不用手动赋值

2:实例化对象必须传入关键数据，否则报错

3：统一对象初始化逻辑，代码复用

4：可以创建对象时附加初始逻辑化（构造函数当中的方法可以参与逻辑业务）

**作业：基础构造方法实操**

业一：基础构造方法实操

需求

1. 创建 Book 图书类，包含属性：编号bookId、书名bookName、价格price
2. 编写构造方法，一次性接收三个参数给属性赋值
3. 定义showBook()方法，打印图书全部信息
4. 使用new创建3本不同图书对象，分别调用展示方法

#### 3.4.2 类的继承 extends

##### 知识点

1. `class 子类 extends 父类`，子类自动拥有父类所有非私有成员

2. `instanceof` 判断对象是否属于某个类（包含继承链）

   ```typescript
   //语法
   / 1.父类
   class 父类名 {
     // 父类属性
     属性名: 类型;
     // 父类构造
     constructor(参数:类型) {
       this.属性名 = 参数;
     }
     // 父类实例方法
     方法名() {
       // 父类逻辑
     }
   }
   
   // 2.子类继承父类
   class 子类名 extends 父类名 {
     // 子类独有属性
     子类属性: 类型;
   
     // 子类构造函数
     constructor(父类所需参数:类型, 子类参数:类型) {
       // 固定第一行：调用父类构造
       super(父类所需参数);
       // 给子类独有的属性赋值
       this.子类属性 = 子类参数;
     }
   
     // 子类自有方法
     子类方法() {}
   }
   
   // 3.创建子类对象
   let 对象名 = new 子类名(父类参数, 子类参数);
   // 可直接调用父类继承来的方法 + 子类自己的方法
   对象名.父类方法();
   对象名.子类方法();
   ```

   

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

#### 继承的核心作用（面向对象）

1. 代码复用
子类直接复用父类已写好的属性与方法，不用重复编写相同逻辑，大幅减少冗余代码。
2. 扩展功能
子类可以在父类基础上新增属性、方法，也能重写父类方法，实现原有功能的升级改造。
3. 统一规范，多态基础
所有子类继承同一个父类，拥有统一的顶层结构，方便统一管理；继承是实现多态的前提。
4. 层次化设计
把同类事物抽象成父类，细分类型做成子类，搭建清晰的类层级，便于项目维护与迭代。



#### 一、作业题目

需求

1. 定义父类：Phone

- 成员属性（protected）：brand 品牌，price 价格

- 构造方法：给品牌、价格赋值

- 成员方法 call()：控制台打印“手机可以打电话”

2. 定义子类 SmartPhone，继承 Phone

- 新增私有属性：system 系统

- 子类构造方法接收品牌、价格、系统，第一行使用 super 调用父类构造

- 使用 override 重写 call() 方法，打印“智能手机可视频通话”

- 新增独有方法 surfInternet()：打印“智能手机可以上网”

3. 页面 Index.ets 中完成测试

- 创建智能手机对象：品牌华为，价格4999，系统鸿蒙

- 调用重写后的 call()、独有 surfInternet() 方法，查看日志输出

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

1. private 私有属性：子类和外部绝对无法访问，只能类内自用

2. protected 属性：仅子类可继承使用，外部 new 对象无法调用

3. 多态核心：**同名方法，不同子类，不同逻辑**

4. 继承核心：**复用父类代码，不重复造轮子**

5. 封装核心：**隐藏隐私数据，开放安全接口**

   ```typescript
   // 父类：学生类
   class Student {
     // public 公开：本类、子类、外部全部能访问
     public studentName: string = "小明";
   
     // protected 受保护：仅本类 + 子类可访问，外部无法访问
     protected phoneNum: string = "13800138000";
   
     // private 私有：只能在当前 Student 类内部访问，子类、外部都不行
     private walletMoney: number = 5000;
   
     // 类内部自身方法：三种修饰符属性全都能正常读取
     innerTest() {
       console.log("===== 本类内部访问全部属性 =====");
       console.log("public姓名：", this.studentName);
       console.log("protected手机号：", this.phoneNum);
       console.log("private存款：", this.walletMoney);
     }
   }
   
   // 子类：优等生，继承父类 Student
   class ExcellentStudent extends Student {
     // 子类重写测试方法
     subTest() {
       console.log("\n===== 子类访问父类属性 =====");
       // public 允许访问
       console.log("public姓名：", this.studentName);
       // protected 允许访问（子类专属权限）
       console.log("protected手机号：", this.phoneNum);
   
       // 下面代码打开会直接编译报错！
       // private 私有属性子类无权读取
       // console.log("private存款：", this.walletMoney);
     }
   }
   
   // ========== 外部调用测试 ==========
   // 创建父类实例
   const student = new Student();
   console.log("\n===== 类外部访问对象属性 =====");
   // public 外部正常访问
   console.log("外部读取public姓名：", student.studentName);
   
   // 下面两行打开全部编译报错
   // protected 外部禁止访问
   // console.log(student.phoneNum);
   // private 外部禁止访问
   // console.log(student.walletMoney);
   
   // 调用类内部方法
   student.innerTest();
   
   // 创建子类实例
   const goodStu = new ExcellentStudent();
   // 子类外部同样只能访问public
   console.log("\n子类外部读取姓名：", goodStu.studentName);
   // 执行子类方法，验证子类可读取protected
   goodStu.subTest();
   ```

   ```typescript
   // 父类：人类（基础模板）
   class Person {
     // private 私有属性：封装隐私，外部/子类不能直接改
     private idCard: string = "3601xxxx1234";
     // protected：子类可以用，外人看不到
     protected age: number = 18;
     // public：对外公开信息
     public name: string;
   
     constructor(name: string) {
       this.name = name;
     }
   
     // 封装：对外提供公开方法，间接操作私有数据，保护隐私
     showIdCard() {
       console.log("身份证(仅内部可展示)：", this.idCard);
     }
   
     // 父类通用方法（多态基础）
     sayHello() {
       console.log(`我是普通人${this.name}，今年${this.age}岁`);
     }
   }
   
   // 子类：学生，继承 Person（继承特性，复用父类代码）
   class StudentPerson extends Person {
     // 子类独有属性
     public studentId: string;
   
     constructor(name: string, sid: string) {
       super(name); // 固定写法：调用父类构造函数
       this.studentId = sid;
     }
   
     // 重写父类同名方法 → 实现多态
     sayHello() {
       // 子类可以读取父类protected age
       console.log(`我是学生${this.name}，学号${this.studentId}，年龄${this.age}`);
     }
   }
   
   // 子类：老师，继承 Person
   class Teacher extends Person {
     public teacherNo: string;
   
     constructor(name: string, tid: string) {
       super(name);
       this.teacherNo = tid;
     }
   
     // 重写同一个sayHello，不同逻辑，多态体现
     sayHello() {
       console.log(`我是老师${this.name}，工号${this.teacherNo}，从教${this.age - 22}年`);
     }
   }
   
   // ========== 测试三大特性 ==========
   console.log("========= 封装测试 =========");
   const p = new Person("张三");
   p.showIdCard(); // 通过公开方法访问私有身份证
   // 直接访问 p.idCard 会报错，隐私被封装保护
   
   console.log("\n========= 继承测试 =========");
   const stu = new StudentPerson("小明", "2026001");
   // 直接复用父类public name属性，不用重复定义
   console.log("继承父类姓名：", stu.name);
   
   console.log("\n========= 多态测试 =========");
   // 同一个方法名 sayHello，不同对象执行不同逻辑
   const personList: Person[] = [
     new Person("路人"),
     new StudentPerson("小红", "2026002"),
     new Teacher("李老师", "T005")
   ];
   personList.forEach(item => {
     item.sayHello();
   });
   ```

   ### 一、作业题目（综合编程题）

   #### 题目：搭建「员工管理体系」类结构

   需求：请基于面向对象三大特性，完成 **父类员工、子类全职员工、子类兼职员工** 的代码编写，具体需求如下：

   ##### 1. 父类：Employee 员工类（实现封装）

   - **私有属性 private**：身份证 idCard、基础工资 salary（外部、子类禁止直接访问）
   - **受保护属性 protected**：员工姓名 name、部门 department（子类可访问，外部不可访问）
   - **公开属性 public**：员工工号 jobId
   - **构造函数**：初始化所有属性（jobId、name、department、idCard、salary）
   - **封装公开方法**：       
     - getSalaryInfo()：脱敏输出工资信息，禁止直接返回原始薪资
     - showBaseInfo()：通用员工自我介绍方法（父类默认逻辑）

   ##### 2. 子类1：FullTimeEmployee 全职员工（继承父类）

   - 继承 Employee 父类，无需重复编写父类已有属性和方法
   - 新增独有公开属性：annualBonus 年终奖
   - **重写父类 showBaseInfo 方法（多态）**：输出全职员工专属介绍，包含姓名、部门、年终奖信息

   ##### 3. 子类2：PartTimeEmployee 兼职员工（继承父类）

   - 继承 Employee 父类，复用父类所有公开、受保护资源
   - 新增独有公开属性：workHour 月工作时长
   - **重写父类 showBaseInfo 方法（多态）**：输出兼职员工专属介绍，包含姓名、部门、工作时长信息

   ##### 4. 测试要求（

   1. 创建父类普通员工、全职员工、兼职员工三个实例对象
   2. 分别调用各自的 `showBaseInfo()` 方法，验证多态效果
   3. 调用 `getSalaryInfo()` 方法，验证封装隐私保护效果
   4. **禁止直接打印私有属性**，

   ```typescript
   
   /**
    * 面向对象三大特性作业：员工管理系统
    * 核心考点：封装、继承、多态 + 三种访问修饰符
    */
   
   // 父类：员工类（实现封装特性）
   class Employee {
     // 【封装-private 私有】仅本类可访问，子类、外部均无法访问
     private idCard: string;
     private salary: number;
   
     // 【封装-protected 受保护】本类、子类可访问，外部不可访问
     protected name: string;
     protected department: string;
   
     // 【public 公开】任意位置可访问
     public jobId: string;
   
     // 构造函数：统一初始化所有属性
     constructor(jobId: string, name: string, department: string, idCard: string, salary: number) {
       this.jobId = jobId;
       this.name = name;
       this.department = department;
       this.idCard = idCard;
       this.salary = salary;
     }
   
     // 【封装公开接口】脱敏展示薪资，禁止外部直接获取原始私有数据
     getSalaryInfo() {
       console.log(`【薪资脱敏展示】岗位基础薪资：${this.salary} 元/月`);
     }
   
     // 父类通用方法（多态基础方法）
     showBaseInfo() {
       console.log(`【普通员工】工号：${this.jobId}，姓名：${this.name}，所属部门：${this.department}`);
     }
   }
   
   // 子类1：全职员工（继承父类 + 重写方法实现多态）
   class FullTimeEmployee extends Employee {
     // 子类独有公开属性
     public annualBonus: number;
   
     // 构造函数：super继承父类所有属性，扩展自身独有属性
     constructor(jobId: string, name: string, department: string, idCard: string, salary: number, annualBonus: number) {
       super(jobId, name, department, idCard, salary); // 固定：调用父类构造，实现继承
       this.annualBonus = annualBonus;
     }
   
     // 【多态】重写父类自我介绍方法，自定义专属逻辑
     showBaseInfo(): void {
       console.log(`【全职员工】工号：${this.jobId}，姓名：${this.name}，部门：${this.department}，年度年终奖：${this.annualBonus} 元`);
     }
   }
   
   // 子类2：兼职员工（继承父类 + 重写方法实现多态）
   class PartTimeEmployee extends Employee {
     // 子类独有公开属性
     public workHour: number;
   
     constructor(jobId: string, name: string, department: string, idCard: string, salary: number, workHour: number) {
       super(jobId, name, department, idCard, salary);
       this.workHour = workHour;
     }
   
     // 【多态】重写父类方法，实现兼职员工专属展示
     showBaseInfo(): void {
       console.log(`【兼职员工】工号：${this.jobId}，姓名：${this.name}，部门：${this.department}，每月工作时长：${this.workHour} 小时`);
     }
   }
   
   // ====================== 【必做：测试代码】 ======================
   console.log("===== 1. 普通父类员工测试（封装验证）=====");
   const normalEmp = new Employee("E001", "张三", "行政部", "3601XXXX1234", 4500);
   normalEmp.showBaseInfo();
   normalEmp.getSalaryInfo();
   
   console.log("\n===== 2. 全职员工测试（继承+多态验证）=====");
   const fullEmp = new FullTimeEmployee("E002", "李四", "技术部", "3602XXXX5678", 6000, 12000);
   fullEmp.showBaseInfo();
   fullEmp.getSalaryInfo();
   
   console.log("\n===== 3. 兼职员工测试（继承+多态验证）=====");
   const partEmp = new PartTimeEmployee("E003", "王五", "市场部", "3603XXXX9012", 3000, 80);
   partEmp.showBaseInfo();
   partEmp.getSalaryInfo();
   
   // ====================== 【拓展加分题：多态批量遍历】 ======================
   console.log("\n===== 拓展题：多态批量统一调用 =====");
   // 父类数组接收所有子类对象，完美体现多态
   const empList: Employee[] = [normalEmp, fullEmp, partEmp];
   empList.forEach(emp => emp.showBaseInfo());
   
   ```

   

#### 3.4.5 类实现接口 implements

##### 知识点

1. `class 类 implements 接口1,接口2`，一个类可实现多个接口

2. 类必须完整实现接口所有必填属性 / 方法；`?`代表可选，可省略

   ##### 语法

```typescript

// 1.先定义接口（规则）
interface 接口名称 {
  属性名: 类型;
  方法名(参数:类型): 返回值类型;
}

// 2.类实现接口
class 类名 implements 接口名称 {
  // 必须全部实现接口里的属性、方法
  属性名: 类型;

  方法名(参数:类型): 返回值类型 {
    // 自己写逻辑
  }
}
//3.调用
// 实例化对象
const 对象变量 = new 类名(构造实参);
// 调用属性
console.log(对象变量.属性名);
// 调用方法
对象变量.方法名(传入参数);
```



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

类使用 `implements` 后，**必须完整实现接口全部成员**，少写属性 / 方法直接编译报错；

接口只能定义结构，不能写属性默认值、不能写方法体；

实现接口的属性不能用 `private`，建议 `public/protected`；

多接口用逗号隔开；

同时有继承和实现：extends 在前，implements 在后。

#### 作业

##### 题目需求：宠物行为规范

1. 定义第一个接口 `IPet`

- 属性：name 宠物名字 string
- 方法：eat (food:string):void 吃东西

1. 定义第二个接口 `ISwim`

- 方法：swim ():void 游泳

1. 创建 `Dog` 类，**仅实现 IPet 接口**

- 实现接口全部属性、方法
- 额外自定义方法：play ()，输出 “小狗玩耍”

1. 创建 `Fish` 类，**同时实现 IPet、ISwim 两个接口**

- 补齐两个接口所有属性、方法

1. 测试调用（必做）

    1）创建 Dog 实例，打印名字、调用 eat、play 方法

    2）创建 Fish 实例，打印名字、调用 eat、swim 方法

##### 加分拓展题（选做）

新增父类 `Animal`

 修改 Fish 类：先继承 Animal，再实现 IPet、ISwim，完成全部调用测试。

##### 三、答题规范

1. 代码添加注释，标注接口定义、implements 实现、调用代码
2. 严格补齐接口所有成员，不允许缺属性、缺方法
3. 代码无编译报错，运行输出符合逻辑
4. 变量、类、接口命名规范

参考答案

```typescript
// 定义宠物接口规范
interface IPet {
  name: string
  eat(food: string): void
}

// 游泳行为接口
interface ISwim {
  swim(): void
}

// Dog类实现IPet单个接口
class Dog implements IPet {
  name: string
  constructor(name: string) {
    this.name = name
  }
  // 实现接口eat方法
  eat(food: string): void {
    console.log(`${this.name}吃${food}`)
  }
  // 自定义方法
  play(): void {
    console.log("小狗玩耍")
  }
}

// Fish类实现两个接口
class Fish implements IPet, ISwim {
  name: string
  constructor(name: string) {
    this.name = name
  }
  eat(food: string): void {
    console.log(`${this.name}吃${food}`)
  }
  swim(): void {
    console.log(`${this.name}在水里游`)
  }
}

// ========== 调用测试代码 ==========
console.log("=====小狗测试=====")
const dog = new Dog("旺财")
console.log(dog.name)
dog.eat("骨头")
dog.play()

console.log("\n=====小鱼测试=====")
const fish = new Fish("小金鱼")
console.log(fish.name)
fish.eat("鱼食")
fish.swim()

// =====加分拓展：继承+多接口=====
class Animal {
  readonly type: string
  constructor(type: string) {
    this.type = type
  }
}
class GoldFish extends Animal implements IPet, ISwim {
  name: string
  constructor(type: string, name: string) {
    super(type)
    this.name = name
  }
  eat(food: string): void {
    console.log(`${this.name}吃${food}`)
  }
  swim(): void {
    console.log(`${this.name}自由游动`)
  }
}
const gold = new GoldFish("鱼类", "锦鲤")
console.log("\n加分拓展：", gold.type, gold.name)
gold.eat("饲料")
gold.swim()
```



### 3.5 泛型

#### 一、泛型是什么

泛型 = **类型占位符**，符号常用 `T / U / V` 表示。

##### 核心痛点（不用泛型会出现的问题）

如果写一个函数只传数字，后面想传字符串、数组、对象，要复制多份几乎一模一样的代码，重复冗余。

 泛型解决：**只写一套代码，使用的时候再告诉程序当前是什么类型，一套代码兼容所有类型**

简单比喻：

 水杯模具（泛型代码），模具固定不变；倒水时你想装白开水、可乐、果汁都行（调用时指定类型）。模具不用重新造，只换里面装的东西。

### 基础语法符号

尖括号 `<>` 包裹泛型标识，``，T 只是约定俗成名字，换 A、Data 都可以。

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

#### 二、作业题目（必做，共4题）

##### 第1题：泛型函数

编写一个**泛型函数 getValue**，功能：接收一个任意类型数据，原封返回该数据。 要求：

- 使用泛型 T
- 分别调用：传入数字、字符串、数组
- 打印结果

##### 第2题：泛型类

定义一个**泛型类 Storage**，用来存储单个数据。 属性：value（T类型） 方法：`getVal()` 返回 value 要求：

- 实例化两个对象： ① 存储字符串数据 ② 存储数字数组数据
- 调用方法打印数据

##### 第3题：泛型接口

定义泛型接口 **IResult** 结构：

- code：number（固定）
- message：string（固定）
- data：T（泛型可变数据）

要求：

- 定义一个 **data为字符串数组** 的返回对象
- 定义一个 **data为数字** 的返回对象

##### 第4题：泛型约束（重点）

定义接口 **ILength**，要求包含属性 `length: number` 编写泛型函数 **showLength** 功能：接收带 length 属性的数据，打印 length 值。

要求：

- 必须使用 **T extends 接口** 约束
- 分别传入：字符串、数组 测试
- 验证：传入数字会报错（数字无length）

#### 三、作业规范要求

1. 每题代码独立、清晰，加简单注释
2. 必须手写泛型语法，不使用 any
3. 所有案例必须调用、打印、出结果
4. 严格遵守泛型约束规则



#### 五、标准答案

```typescript
// ========== 1. 泛型函数 ==========
function getValue<T>(val: T): T {
  return val;
}

// 多类型调用
console.log(getValue<number>(100));
console.log(getValue<string>("泛型测试"));
console.log(getValue<number[]>([1,2,3]));

// ========== 2. 泛型类 ==========
class Storage<T> {
  value: T;
  constructor(val: T) {
    this.value = val;
  }
  getVal(): T {
    return this.value;
  }
}

// 实例1：字符串
let s1 = new Storage<string>("Hello TS");
console.log(s1.getVal());

// 实例2：数字数组
let s2 = new Storage<number[]>([10,20,30]);
console.log(s2.getVal());

// ========== 3. 泛型接口 ==========
interface IResult<T> {
  code: number;
  message: string;
  data: T;
}

// 数据为字符串数组
let res1: IResult<string[]> = {
  code: 200,
  message: "获取成功",
  data: ["张三","李四"]
}

// 数据为数字
let res2: IResult<number> = {
  code: 200,
  message: "成功",
  data: 666
}
console.log(res1);
console.log(res2);

// ========== 4. 泛型约束 T extends 接口 ==========
// 约束：必须包含 length 属性
interface ILength {
  length: number
}

// T 必须满足 ILength 结构
function showLength<T extends ILength>(target: T): void {
  console.log("长度为：", target.length);
}

// 合法：字符串、数组都有 length
showLength("abc");
showLength([1,2,3]);

// 报错：number 没有 length
// showLength(123);
    
```

### 六、极简总结

- **泛型函数**：函数名<T>，调用时指定类型
- **泛型类**：类名<T>，new 的时候确定类型
- **泛型接口**：接口名<T>，让接口某个字段灵活适配多类型
- **泛型约束**：**T extends 接口**，给泛型加“准入门槛”，限制必须包含某些属性

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

