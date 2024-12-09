---
title: Ts
date: 2023-06-28 21:00:10
tags:



---



# TS安装命令

**起步安装**  ***npm install typescript -g***

1.需转换JS
**TS文件转换JS**  ***tsc 文件名.ts***
**JS文件运行** ***node 文件名.js***

2.可以直接运行TS，不用转换JS
**TS库** ***npm install ts-node -g***
**生成package.json** ***npm init -y***
**生成package-lock.json** ***npm i @types/node -D***

# 基础类型

### 1.字符串类型

字符串是使用string定义的

```TypeScript
let a: string = '123'
//普通声明

//也可以使用es6的字符串模板
let str: string = `dddd${a}`
```

### 2.数字类型

支持十六进制、十进制、八进制和二进制；

```tsx
let notANumber: number = NaN;//Nan
let num: number = 123;//普通数字
let infinityNumber: number = Infinity;//无穷大
let decimal: number = 6;//十进制
let hex: number = 0xf00d;//十六进制
let binary: number = 0b1010;//二进制
let octal: number = 0o744;//八进制s
```

### 3.布尔类型

注意，使用构造函数 `Boolean` 创造的对象**不是**布尔值，而是布尔对象

```typescript
let booleand: boolean = true //可以直接使用布尔值
 
let booleand2: boolean = Boolean(1) //也可以通过函数返回布尔值
```

### 4.空值类型

在 [TypeScript](https://so.csdn.net/so/search?q=TypeScript&spm=1001.2101.3001.7020) 中，可以用 `void` 表示没有任何返回值的函数

```typescript
let u: void = undefined
let n: void = null;
```

### 5.Null和undefined类型

```typescript
let u: undefined = undefined;//定义undefined
let n: null = null;//定义null
```

void 和 undefined 和 null 最大的区别
与 void 的区别是，undefined 和 null 是所有类型的子类型。也就是说 undefined 类型的变量，可以赋值给 string 类型的变量：

```typescript
//这样写会报错 void类型不可以分给其他类型
let test: void = undefined
let num2: string = "1"

num2 = test
//这样是没问题的
let test: null = null
let num2: string = "1"

num2 = test

//或者这样的
let test: undefined = undefined
let num2: string = "1"

num2 = test
```

# 任意类型

### Any 类型

1.没有强制限定哪种类型，随时切换类型都可以 我们可以对 any 进行任何操作，不需要检查类型 
2.声明变量的时候没有指定任意类型默认为any3.弊端如果使用any 就失去了TS类型检测的作用
3.弊端如果使用any 就失去了TS类型检测的作用
4.[TypeScript](https://so.csdn.net/so/search?q=TypeScript&spm=1001.2101.3001.7020) 3.0中引入的 unknown 类型也被认为是 top type ，但它更安全。与 any 一样，所有类型都可以分配给unknown

```typescript
let anys:any = 123
anys = '123'
anys = true

let a;
a = 123
a = true
```



###  unknown 顶级类型

unknow类型比any更加严格当你要使用any 的时候可以尝试使用unknow

```typescript
//unknown可赋值对象只有unknown 和 any
let bbb:unknown = '123'
let aaa:any= '456'
 
aaa = bbb
```

unkoown 类型 是不能调用属性和方法的，any 可以

```typescript
如果是any类型在对象没有这个属性的时候还在获取是不会报错的
let obj:any = {b:1}
obj.a
 
 
如果是unknow 是不能调用属性和方法
let obj:unknown = {b:1,ccc:():number=>213}
obj.b
obj.ccc()
```

### Object 和 object类型

Object 包括所有类型
object 包括非值类型的类型
{}包括所有类型，与Object类似

tips：{}定义的字面量模式不能修改值的。

```typescript
let o:object = {}  //正确
let o:object = []  //正确
let o:object = ()=>123  //正确
let o:object = 123 //错误
let o:object = 'abc' //错误


let a:{}= {age:1}
a.age = 3  // 无法修改
```

# interface 接口

### 1.interface 重合

```typescript
interface Fn {
    name : string,
    age : number,
}

interface Fn {
    id : number
}
interface可以重名，并且继承该interface的对象需声明全部属性和函数
let o:Fn = {
    name : 'nihao',
    age : 14,
    id : 12
}
```

### 2.interface 任意key

如果在interface中不知道定义何种属性，可以通过索引签名：
[名字:string]:any  使用后该属性不需要在对象中声明。

设置可选属性， 在interface中定义： age ?:number 添加一个问号， 使用后该属性不需要在对象中声明。

**readonly 为 只读，用来修饰属性和函数，不可修改。**

### 3.interface定义函数

```typescript
interface Fn {
    (name:string):string // 有参函数
}

const fn:Fn = function(name:string){
    return name
}


interface Fn {
    ():string // 无参函数
}

const fn:Fn = function(){
    return "肖镇楠"
}
```

# 数组类型

```typescript
let arr1:string[] = ["1","2"]
let arr2:Array<number> = [1,2,2,3]

interface Fn {
    name : string
    age ?: number
}
对象数组
let f:Fn[] = [{name:"xzn",age:1},{name:"wo"}]

二维
let arr:any[][] = [[1],[2,"2"],[3,true]]
```

# 函数类型

### 普通函数

```typescript
有参函数
function add(a : number , b : number): number {
    return a + b
}

无参无反函数
function cao(): void {
    console.log("不想玩啦")
}

函数可选参数
function add(a : number  = 2, b : number): number {
    return a + b
}
```

### 函数接口

```typescript
第一种写法
interface Obj{
    user:number[]	// 数组
    add:(num:number)=>void	// 添加
}
let obj:Obj = {
    user :[1,2,3],
    add(num:number){	// 视频中第一个参数是this
        this.user.push(num)
    }
}
第二种 带this
interface Obj{
    user:number[]
    add:(this:Obj,num:number)=>void
}
let obj:Obj = {
    user :[1,2,3],
    add(this:Obj，num:number){
        this.user.push(num)
    }
}
```



### 函数重载

```typescript
let user:number[] = [1,2,3]

function findNum(str:string):any  有参函数
function findNum(id:number):any	
function findNum():any			 无参函数
function findNum(ids?:number | string):any{  //联合类型
    if(typeof ids =='number'){		//匹配类型
        return user.filter(v => v== ids)
    }else if(typeof ids == 'string'){
        return "好累"
    }else{
        return user
    }
}
console.log(findNum("1"))
```

# 联合类型

**属性联合**

```typescript
let str: string | number
str = 123
str = "23"
```

**函数联合**

```typescript
interface People{
    name:string,
    age:number
}

interface Man{
    sex:number
}
// 两个接口都继承
const nan = (man: People & Man) :void=>{
    console.log(man)
}

nan({
    name:"xzn",
    age:15,
    sex:1
})
```

### 类型断言

```typescript
let fn = function(num: number |string):void{
    // 第一种写法
    console.log((num as string).length);  // num如果是number类型转换为string类型
    // 第二种写法
    console.log((<string>num).length);  // num如果是number类型转换为string类型
} 
fn('1213')
```

