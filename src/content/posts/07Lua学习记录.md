---
title: Lua学习记录
published: 2025-04-12
description: "关于Lua的语法基础讲解 。"
image: "https://typorytp.oss-cn-hangzhou.aliyuncs.com/05.png"
tags: ["热更新", "Lua"]
category: Unity
draft: false
---

# Lua学习记录

## 基础语法

### Lua中的变量

**Lua 是动态类型语言，变量不要类型定义,只需要为变量赋值。 值可以存储在变量中，作为参数传递或结果返回。**

| 数据类型 | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| nil      | 这个最简单，只有值nil属于该类，表示一个无效值（在条件表达式中相当于false）。 |
| boolean  | 包含两个值：false和true。                                    |
| number   | 表示双精度类型的实浮点数                                     |
| string   | 字符串由一对双引号或单引号来表示                             |
| function | 由 C 或 Lua 编写的函数                                       |
| userdata | 表示任意存储在变量中的C数据结构                              |
| thread   | 表示执行的独立线路，用于执行协同程序                         |
| table    | Lua 中的表（table）其实是一个"关联数组"（associative arrays），数组的索引可以是数字、字符串或表类型。在 Lua 里，table 的创建是通过"构造表达式"来完成，最简单构造表达式是{}，用来创建一个空表。 |

变量在使用前，需要在代码中进行声明，即创建该变量。

编译程序执行代码之前编译器需要知道如何给语句变量开辟存储区，用于存储变量的值。

Lua 变量有三种类型：全局变量、局部变量、表中的域。

Lua 中的变量全是全局变量，哪怕是语句块或是函数里，除非用 local 显式声明为局部变量。

局部变量的作用域为从声明位置开始到所在语句块结束。

变量的默认值均为 nil。

### 字符串

~~~Lua
--字符串允许单双引号定义，使用\n或者[[]]方式定义多行字符串
a ="abc"
b= '哈哈哈'
c=[[
a
b
c
]]

print(a)
print(b)
print(c)

--字符串使用..进行拼接，使用#获取长度,字母占1位，汉字占三位
print(a..b)
print(#(a..b))
--使用%d,%f，%a，%s分别关联整形数字,浮点型数字，字符，字符串
a ="abc"
b= '哈哈哈'
c=233.333
print(string.format("a=%s,b=%s,c=%d,c2=%f",a,b,c,c))

--字符串类型转换
a=true;
print(tostring(a))
--小写转大写
a="Abc"
print(string.upper(a))
--大写转小写
print(string.lower(a))
--字符串截取
a="abcdefg"
print(string.sub(a,1,4))
--字符串查找
a="agbcdefg"
print(string.find(a,"cde"))--返回起始和结束位置
--字符串替换
a="acbcdefg"
print(string.gsub(a,"c","d"))--返回替换后的字符串和替换次数
--字符串长度
a="abcdefg"
print(string.len(a))
--字符串修改
a="abcabc"
print(string.gsub(a,"ab","**"))

~~~

### 运算符

Lua中没有自增减和复合运算，例如++，--，+=,*=等等；

Lua的条件运算符除了不等于（**!=**）是**~=**，其他都一致;

Lua逻辑运算符是 and、 or、 not；

Lua不支持逻辑运算符、三目运算符；

### 流程控制与循环

#### if条件分支语句

注意写法就好，用法都一样

~~~Lua
a= 1
if a==1 then
    print("a=1")
    a=2
elseif a==2 then
    print("a=2")
    a=3
else
    print("a=3")
end
--打印 a=1
~~~

#### 循环语句

~~~Lua
--while
a=0
while(a<5) do
    print(a)
    a=a+1
end
--do while
b=0
repeat
    print(b)
    b=b+1
until b>5
--for
--for循环里前两个参数定义左右边界，第三个参数定义步长，第三个参数可以省略
for i=5,1,-1 do
    print(i)
end
~~~

### 函数

~~~Lua
--函数定义之后才能调用
--test(1)在这里调用会报错
function  test1()
    print("test1")
end
test1()
--第二种声明方式
test2 = function()
    print("test2")
end
test2()
--有参数的函数
function test3(a,b)
    print("两数之和"..a+b)
end

~~~

关于参数的特殊点需要注意：

```lua
function test3(a,b,c)
    print(a,b,c)
end

test3(1,true,"abc")
--如果参数没有填完整，也不会报错，只会赋值为nil
test3(1)

--Lua的函数允许设置任意长度参数
function test5(...)
    args={...}
    for i=1,#args do
        print(args[i])
    end
end

test5("one", "two", "three")
```

带返回值的函数

~~~Lua
function test3(a)
    return a,true,"CS2";
end

print(test3(1))
--打印：1,true,CS2
temp1 = test3(1)
print(temp1)
--打印：1
temp1,temp2 = test3(1)
print(temp1,temp2)
--打印：1,true
temp1,temp2,temp3 = test3(1)
print(temp1,temp2,temp3)
--打印：1,true,CS2
temp1,temp2,temp3,temp4 = test3(1)
print(temp1,temp2,temp3,temp4)
--打印：1,true,CS2,nil
~~~

Lua不支持函数的重载，只会调用最近声明的函数

~~~Lua
function test4()
    print("test4")
end

function test4(a)
    print(tostring(a).."最近的一个函数")
end

print(test4())
--打印nil最近的一个函数

~~~

闭包

```lua
--闭包 = 函数 + 创建时的词法环境。当 createCounter() 返回内部函数时：
--内部函数 永久持有 外部函数的 count 变量
--即使外部函数 createCounter() 已执行完毕，count 依然存活
function createCounter()
    local count = 0  -- 这个变量会被闭包"记住"

    return function()
        count = count + 1  -- 操作闭包捕获的count
        return count
    end
end

-- 创建两个独立的计数器
counterA = createCounter()
counterB = createCounter()

print(counterA())  --> 1
print(counterA())  --> 2
print(counterB())  --> 1
print(counterA())  --> 3
```

## 表

表（table）是Lua的一种复杂数据类型，是一切复杂数据类型的基础，相当于数组。

### 基础用法

Lua允许一个表中存入各种类型的变量，当nil在最后的位置时，将不计入长度，**表从1开始遍历**

~~~lua
a = {1,2,3,"a","bc",nil,true}
b = {1,2,3,"a","bc",true,nil}
c = {1,2,3,"a","bc",true,nil,nil,nil}
print(#a)--打印7
print(#b)--打印6
print(#c)--打印6

--多维表
a={{1,"a",false},{2,"b",true},{3,"c",false,"ssss"}}


~~~

#### 遍历

~~~lua
--遍历
for i=1,#a do
    print(a[i])
end 

--多维表遍历
a={{1,"a",false},{2,"b",true},{3,"c",false,"ssss"}}

for i=1,#a do
    b=a[i]
    for j=1,#b do
        print(b[j])
    end
end
~~~

#### 自定义索引

```lua
--自定义索引
a = {[-1]=1,2,[0]=3,"a","bc",nil,true}
--打印1，3，2，a，bc，nil，true

print(#a)--打印5 

--跳跃性地设置自定义索引会出问题
a = {[-1]=1, [1]=2, [8]=3, [3]=4, [4]=5}
for i = -1,#a do
    print(i,a[i])--只会打印-1和1的索引，以及0 nil
end
print(#a)--打印1
```

#### 迭代器

~~~LUA
a = { [-1]=1,2,3,[8]=4,5,6}
--inpairs从1开始遍历，而且只遍历连续的索引
for i,v in ipairs(a) do
	print(i,v)
end
--inpairs遍历所有键值
for i,v in pairs(a) do
    print(i,v)
end
--遍历所有的键
for i in pairs(a) do
    print(i)
end
~~~

#### 字典

Lua实现字典的方式还是使用自定义索引，定义每一个键值对，如果在引用时使用没有出现过的键，则为nil，如果想要删除直接定义那个键的值为nil即可。

~~~lua
--字典
dict = {['a']='1'}
--新增
dict["b"]='2'
--删除
dict["a"]=nil
--修改
dict["b"]='3'
--遍历
for k,v in pairs(dict) do
    print(k,v)
end
~~~

### 表的公共操作

~~~lua
t1 = {{age =1 , name = "tom"},{age =2 , name = "SB"}}
t2 = {age =3 , name = "CS"}

print(#t1)
--插入   把t2插入到t1中，t2消失
table.insert(t1,t2)
print(#t1)
print(#t2)
for i,v in pairs(t1) do
    print(v.age,v.name)
end

--移除  移除最后一位元素
table.remove(t1)
--移除指定位置的元素
table.remove(t1,1)
for i,v in pairs(t1) do
    print(v.age,v.name)
end

t2 = {4,87,32,2,3,66}
--排序  升序排序
table.sort(t2)
--降序排序 第一个参数是排序表，第二个是排序规则
table.sort(t2,function(a,b)
	return a>b
end)

for k,v in pairs(t2) do
	print(k,v)
end


--拼接  常用于拼接字符串,第一个是需要拼接的表，第二个是拼接符号
t3 = {"1","2","3"}
str=table.concat(t3,",")
print(str)
~~~

## 多脚本执行

### 全局变量和本地变量

Lua声明一个变量默认是一个全局变量，使用local修饰才能变成局部变量，for循环里的i也属于局部变量

~~~lua
for i=1,3 do
    print(i)
    j="存在"
end
print(i)--nil
print(j)--存在
function test()
    local a="不存在"
    b="存在"
end
test()
print(a)--nil
print(b)--存在
~~~

### 多脚本执行

全局变量可以跨脚本执行，但局部变量不可以

在test2.lua添加如下代码

~~~lua
print("Test2")
testB="CS2"
local testLocalB="Local CS2"
~~~

test1.lua执行如下代码

~~~lua
--使用require执行test2.lua
require("test2")
print(testB)--输出CS2
print(testLocalB)--输出nil
~~~

### 脚本的加载与卸载

require加载过一次后不会再被执行

~~~lua
--使用require加载过一次后不会再被执行
require("test2") --此时test2的print不再打印
~~~

脚本卸载

```lua
--使用require执行test2.lua
require("test2")
print(testB)--输出CS2
print(testLocalB)--输出nil

--使用require加载过一次后不会再被执行
require("test2") --此时test2的print不再打印
--判断脚本是否被执行
print(package.loaded["test2"])
--卸载脚本
package.loaded["test2"]=nil
print(package.loaded["test2"])
```

### 大G表

~~~lua
--_G表是一个总表，将我们的所有全局变量都存储在其中,本地变量不会存储
for k,v in pairs(_G) do
    print(k,v)
end
~~~

虽然不会存储局部变量，但我们可以在一个脚本里return一个局部变量，然后去获取它

~~~lua
--tets2.lua
print("Test2")
testB="CS2"
local testLocalB="Local CS2"
return testLocalB


--test1.lua
local testB = require("test2")
print(testB)--打印Local CS2
~~~

## 协程

### 协程的创建

~~~lua
fun = function(a)
  print(a)
end

--协程的本质是创建一个线程对象
co = coroutine.create(fun)

print(co)
print(type(co))
--这种方式创建就是一个函数
co2 = coroutine.wrap(fun)
print(co2)
print(type(co2))


~~~

### 协程的挂起

~~~lua
fun = function()
    local a=1
    while true do
        print(a)
        a=a+1
        --挂起协程，下一次执行协程继续在这个while循环里执行,返回a的值
        coroutine.yield(a)
    end
end
--创建但不执行
co= coroutine.create(fun)
--执行
coroutine.resume(co)
coroutine.resume(co)
coroutine.resume(co)
coroutine.resume(co)
--此时a=5
isOK,tempA=coroutine.resume(co)
--再次执行后a=6
print(isOK,tempA)--打印true,6

temp = coroutine.wrap(fun)
print(temp())

~~~



### 协程的状态

~~~lua
fun = function()
    local a=1
    while a<2 do
        print(a)
        a=a+1
        --打印协程运行（runing）状态
        print(coroutine.status(co))
        coroutine.yield(a)
    end
end
--创建但不执行
co= coroutine.create(fun)

coroutine.resume(co)
--打印协程挂起（suspended）状态
print(coroutine.status(co))

--打印协程结束（dead）状态
coroutine.resume(co)
print(coroutine.status(co))

~~~

## 元表

### 概念

表的元数据：每个表可关联一个元表（通过setmetatable），用于定义该表的特殊行为
元方法机制：通过预定义键（如__index, __add）挂载函数来实现功能扩展

~~~lua
--任何表变量都可以作为另一个表的元表
--任何表变量都可以有自己的元表
--当字表进行一些操作时会执行元表中的操作
meta = {}
myTable={}
--第一个是子表，第二个是元表
setmetatable(myTable,meta)


~~~

### 元表的特定操作

#### __tostring

~~~lua
meta ={
    --当字表要当做字符串时，调用__tostring方法
    __tostring =function(t)
        return t.name
    end
}

myTable={
    name="donk666",
}
print(myTable)--打印table: 00CE9710
setmetatable(myTable,meta)
print(myTable)--打印donk666
~~~

#### __call

~~~lua

meta2 ={
    --当子表要当做函数时，调用__call方法
    --__call默认第一个参数是表本身
    __call =function(a,b)
        print(a.name)
        print(b)
    end
}

myTable2={
    name="donk666",
}

setmetatable(myTable2,meta2)

myTable2("niko")--打印donk666  niko
~~~

#### 运算符重载

~~~Lua
meta ={
    --运算符重载，当子表使用+-*/^%等运算符时会调用方法
    --运算符+
    __add = function(t1,t2)
        return t1.age + t2.age
    end,
    --运算符-
    __sub = function(t1,t2)
        return t1.age - t2.age
    end,
    --运算符*
    __mul  = function(t1,t2)
        return t1.age * t2.age
    end,
    --运算符/
    __div = function(t1,t2)
        return t1.age / t2.age
    end,
    --运算符%
    __mod = function(t1,t2)
        return t1.age % t2.age
    end,
    --运算符^
    __pow = function(t1,t2)
        return t1.age ^ t2.age
    end,
    --运算符<
    __lt = function(t1,t2)
        return t1.age < t2.age
    end,
    --运算符<=
    __le = function(t1,t2)
        return t1.age <= t2.age
    end,
    --运算符==
    __eq = function(t1,t2)
        return t1.age == t2.age
    end,
    --运算符..
    __concat  = function(t1,t2)
        return t1.age .. t2.age
    end,


}

myTable1 = {
    age = 10
}
myTable2 ={
    age = 20
}
setmetatable(myTable1, meta)

print(myTable1 + myTable2)
print(myTable1 - myTable2)
print(myTable1 * myTable2)
print(myTable1 / myTable2)
print(myTable1 % myTable2)
print(myTable1 ^ myTable2)
--如果使用条件运算符重载，必须设置两个字表的元表一致
setmetatable(myTable2, meta)
print(myTable1 < myTable2)
print(myTable1 <= myTable2)
print(myTable1 == myTable2)
print(myTable1 .. myTable2)

~~~

#### __index

~~~lua
meta ={
    age = 1
}
myTable ={

}
setmetatable(myTable,meta)
--子表中找不到指定属性时，会去元表的__index指定的表查找
print(myTable.age)--打印nil
--__index最好写在表外部,不然可能出问题
meta.__index = {age = 2}
print(myTable.age)--打印2

~~~

####  __newindex

~~~lua
meta ={

}

myTable ={

}
myTable.__newindex = {}
setmetatable(myTable,meta)
--当赋值时赋值给一个不存在的索引，则会把值赋到newindex所指的表中，不会修改自己
myTable.age = 1
print(myTable.age)

~~~



## 面向对象

### 类与结构体

Lua并没有面向对象概念，只是通过结构体模仿面向对象来操作，但结构体本质上还是表。

~~~lua
Student  = {
    name = "Tom",
    age = 18,
    isCS = true,
    show = function(a)
        print(a.name, a.age,a.isCS)
    end
}

Teacher = {
    name = "Jerry",
    age = 30,
    show = function(a)
        print(a.name, a.age)
    end
}

print(Student.name)--打印Tom
-- .调用方法就是正常调用
Student.show(Student)--打印Tom 18 true
Student.show(Teacher)--打印Jerry 30 nil
--  :调用方法会把调用者作为第一个参数传入方法中
Student:show()--打印Tom 18 true     Student:show()等价于Student.show(Student)
--表在声明之后可以在表外修改变量和方法
Student.name="NM"
--还可以新增变量
Student.AA="asdda"
--也可以修改函数方法
Student.show = function(a)
    print(a.name, a.age,a.isCS,a.AA)
end
Student:show()
~~~

关于符号  **:**  和  **.**  还是补充说明一下，使用：调用方法只是把调用者作为**第一个**参数传入，如果函数后续还有参数则还需要额外传入；

~~~lua
Student  = {
    name = "Tom",
    age = 18,
    isCS = true,
    show = function(a,b)
        print(a.name, a.age,a.isCS,b)
    end
}
Student:show() --打印Tom	18	true	nil
Student:show(23333)--打印Tom	18	true	23333
~~~

使用   ：声明一个方法相当于把自身作为一个参数，常常配合self用于在声明之后的外部使用， self是一个关键字，表示默认传入调用者为第一个参数

~~~lua
Student  = {
    name = "Tom",
    age = 18,
    isCS = true,
    --这里的self只是作为一个参数来使用，不是关键字
    show = function(self, b)
        print(self.name, self.age, self.isCS, b)
    end
}
--和上面的方法完全等价，self相当于Student，
function Student:show(b)
    print(self.name, self.age, self.isCS, b)
end
Student:show() --打印Tom	18	true	nil
Student:show(23333)--打印Tom	18	true	23333
~~~

### 封装

~~~lua
Object ={}
Object.id =1

function Object:Test()
    print(self.id)
end

function Object:new()
    local obj ={}
    self.__index =self
    setmetatable(obj,self)
    return obj
end

local myObj=Object:new()
print(myObj.id)
myObj:Test()
~~~



