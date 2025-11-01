---
title: XLua学习记录
published: 2025-05-01
description: XLua的基本用法和C#之间的通信用法概述。"
image: "https://typorytp.oss-cn-hangzhou.aliyuncs.com/lua02.png"
tags: ["热更新","Lua"]
category: Unity
draft: false
---


# XLua学习记录

## C#调用Lua

### Lua解析器

~~~cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using XLua;

public class L1_LuaEnv : MonoBehaviour
{
   
    void Start()
    {
        //Lua解析器，直接在unity中执行Lua
        //一般保持它的唯一性，用一个Lua解析器执行所有Lua语言
        LuaEnv env = new LuaEnv();
        //执行Lua语言
        env.DoString("print('hello world')");
        //默认从Resource文件夹下去加载Lua脚本
        env.DoString("require('Main')");
        //垃圾回收，帧更新中定时执行
        env.Tick();
        //销毁Lua解析器 
        env.Dispose();
        //
    }

    
    void Update()
    {
        
    }
}

~~~

### Lua文件加载重定向

在每一次require一个Lua脚本时会把文件名作为参数传入自定义函数中，AddLoader执行这个委托，可以根据文件名来重定向加载文件信息。先加载自定义规则，找不到了再去默认路径中找。

~~~cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Windows;
using XLua;
public class L2_Loader : MonoBehaviour
{
    
    void Start()
    {
        LuaEnv env = new LuaEnv();
        //参数是一个委托，AddLoader自动寻找Lua脚本，只有找不到时才使用默认文件路径
        env.AddLoader(MyCustomLoader);
        env.DoString("require('Main')");
    }

    //自动执行 
    private byte[] MyCustomLoader(ref string filepath)
    {
        //通过函数中的逻辑去加载Lua文件
        //传入参数是require执行的lua脚本文件名
        //拼接一个Lua文件所在路径
        string path = Application.dataPath+"/Lua/"+filepath+".lua";
        Debug.Log(path);
        if (File.Exists(path))
        {
            return File.ReadAllBytes(path);
        }
        else
        {
            Debug.Log("重定向失败，文件名称为"+filepath);
        }
        return null;
    }

    
    void Update()
    {
        
    }
}

~~~

### Lua解析器管理器

基于单例模式自己去封装一些Lua解析器常用功能，并且保证解析器的唯一性。

~~~cs
using System.Collections;
using System.Collections.Generic;
using System.IO;
using UnityEngine;
using XLua;

public class LuaManager :BaseManager<LuaManager>
{
    private LuaEnv luaEnv;
    
    //初始化解析器
    public void Init()
    {
        if (luaEnv != null)
        {
            return;
        }
        luaEnv = new LuaEnv();
        luaEnv.AddLoader(MyCustomLoader);
    }

    public void DoString(string str)
    {
        luaEnv.DoString(str);
    }

    private byte[] MyCustomLoader(ref string filepath)
    {
        
        string path = Application.dataPath+"/Lua/"+filepath+".lua";
        Debug.Log(path);
        if (File.Exists(path))
        {
            return File.ReadAllBytes(path);
        }
        else
        {
            Debug.Log("重定向失败，文件名称为"+filepath);
        }
        return null;
    }
    //垃圾回收
    public void Tick()
    {
        luaEnv.Tick();
    }
    //销毁
    public void Dispose()
    {
        luaEnv.Dispose();
        luaEnv = null;
    }
    
}
~~~

Lua脚本最终会放在AB包中，最终会通过加载AB包再加载其中的lua甲苯资源来执行，AB包中如果加载文本，.lua不能被识别，打包时依然需要把lua文件后缀改为txt。

~~~cs
private byte[] MyCustomABLoader(ref string filepath)
    {
        Debug.Log("AB包加载");
        //从AB包中加载Lua文件
        string path = Application.streamingAssetsPath + "/lua";
        
        AssetBundle ab = AssetBundle.LoadFromFile(path);
        
        TextAsset tx = ab.LoadAsset<TextAsset>(filepath + ".lua");
        return tx.bytes;

        // TextAsset lua = ABMgr.GetInstance().LoadRes<TextAsset>("lua", filepath + "lua");
        // if (lua != null)
        //     return lua.bytes;
        // else
        //     Debug.Log("加载Lua文件失败");
        // return null;

    }
~~~

完整manager展示

~~~cs
using System.Collections;
using System.Collections.Generic;
using System.IO;
using UnityEngine;
using XLua;

/// <summary>
/// Lua管理器
/// 提供 lua解析器
/// 保证解析器的唯一性
/// </summary>
public class LuaManager : BaseManager<LuaManager>
{
    //执行Lua语言的函数
    //释放垃圾
    //销毁
    //重定向
    private LuaEnv luaEnv;


    /// <summary>
    /// 得到Lua中的_G
    /// </summary>
    public LuaTable Global
    {
        get
        {
            return luaEnv.Global;
        }
    }


    /// <summary>
    /// 初始化解析器
    /// </summary>
    public void Init()
    {
        //已经初始化了 别初始化 直接返回
        if (luaEnv != null)
            return;
        //初始化
        luaEnv = new LuaEnv();
        //加载lua脚本 重定向
        luaEnv.AddLoader(MyCustomLoader);
        luaEnv.AddLoader(MyCustomABLoader);
    }

    //自动执行
    private byte[] MyCustomLoader(ref string filePath)
    {
        //通过函数中的逻辑 去加载 Lua文件 
        //传入的参数 是 require执行的lua脚本文件名
        //拼接一个Lua文件所在路径
        string path = Application.dataPath + "/Lua/" + filePath + ".lua";

        //有路径 就去加载文件 
        //File知识点 C#提供的文件读写的类
        //判断文件是否存在
        if (File.Exists(path))
        {
            return File.ReadAllBytes(path);
        }
        else
        {
            Debug.Log("MyCustomLoader重定向失败，文件名为" + filePath);
        }


        return null;
    }


    //Lua脚本会放在AB包 
    //最终我们会通过加载AB包再加载其中的Lua脚本资源 来执行它
    //重定向加载AB包中的LUa脚本
    private byte[] MyCustomABLoader(ref string filePath)
    {
        //Debug.Log("进入AB包加载 重定向函数");
        ////从AB包中加载lua文件
        ////加载AB包
        //string path = Application.streamingAssetsPath + "/lua";
        //AssetBundle ab = AssetBundle.LoadFromFile(path);

        ////加载Lua文件 返回
        //TextAsset tx = ab.LoadAsset<TextAsset>(filePath + ".lua");
        ////加载Lua文件 byte数组
        //return tx.bytes;

        //通过我们的AB包管理器 加载的lua脚本资源
        TextAsset lua = ABMgr.GetInstance().LoadRes<TextAsset>("lua", filePath + ".lua");
        if (lua != null)
            return lua.bytes;
        else
            Debug.Log("MyCustomABLoader重定向失败，文件名为：" + filePath);

        return null;
    }


    /// <summary>
    /// 传入lua文件名 执行lua脚本
    /// </summary>
    /// <param name="fileName"></param>
    public void DoLuaFile(string fileName)
    {
        string str = string.Format("require('{0}')", fileName);
        DoString(str);
    }

    /// <summary>
    /// 执行Lua语言
    /// </summary>
    /// <param name="str"></param>
    public void DoString(string str)
    {
        if(luaEnv == null)
        {
            Debug.Log("解析器为初始化");
            return;
        }
        luaEnv.DoString(str);
    }

    /// <summary>
    /// 释放lua 垃圾
    /// </summary>
    public void Tick()
    {
        if (luaEnv == null)
        {
            Debug.Log("解析器为初始化");
            return;
        }
        luaEnv.Tick();
    }

    /// <summary>
    /// 销毁解析器
    /// </summary>
    public void Dispose()
    {
        if (luaEnv == null)
        {
            Debug.Log("解析器为初始化");
            return;
        }
        luaEnv.Dispose();
        luaEnv = null;
    }
}

~~~



### 全局变量获取

~~~Lua
--Main.lua
print("Main Lua启动") 
require("Test1")

--Test1.lua
print("Test1 Running")
testNumber = 1
testFloat = 1.1
testString = "testString"
testBool = true
testDouble = 1.2
~~~

在C#中通过以下方法进行全局变量的值拷贝和值修改，但是无法对局部变量进行操作。

~~~cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class L4_CallValue : MonoBehaviour
{
    // Start is called before the first frame update
    void Start()
    {
        LuaManager.GetInstance().Init();
        LuaManager.GetInstance().DoString("require('Main')");
        //值拷贝
        int i = LuaManager.GetInstance().Global.Get<int>("testNumber");
        Debug.Log(i);
        float f = LuaManager.GetInstance().Global.Get<float>("testFloat");
        Debug.Log(f);
        string s = LuaManager.GetInstance().Global.Get<string>("testString");
        Debug.Log(s);
        bool b = LuaManager.GetInstance().Global.Get<bool>("testBool");
        Debug.Log(b);
        double d = LuaManager.GetInstance().Global.Get<double>("testDouble");
        Debug.Log(d);
        //值修改
        LuaManager.GetInstance().Global.Set("testNumber", 100);
        Debug.Log(LuaManager.GetInstance().Global.Get<int>("testNumber"));
    }

    
    void Update()
    {
        
    }
}

~~~

### 函数获取

lua文件定义如下函数

~~~Lua
print("Test2 Running")

testFun1 = function()
	print("无参无返回")
end


testFun2 = function(a)
    print("有参有返回")
	return a+1
end

testFun3 = function(a) 
	print("有参多返回")
	return 1,2,false,"CS2",a
end

testFun4 = function(a,...)
	print("变长参数")
	print(a)
	for i,v in ipairs({...}) do
		print(v)
	end
end
~~~

对于这些类型的函数，C#通过以下方式去调用

~~~cs
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;
using XLua;

//无参无返回委托
public delegate void CustomCall();
//有参有返回委托
//该特性是在XLua命名空间中的,需要添加一个特性才能让xlua识别，使用之后在窗口中的XLua中
[CSharpCallLua]
public delegate int CustomCall2(int a);
[CSharpCallLua]
public delegate int CustomCall3(int a, out int b, out int c, out bool d, out string e, out int f);
[CSharpCallLua]
public delegate int CustomCall4(int a, ref int b, ref int c, ref bool d, ref string e, ref int f);
[CSharpCallLua]
public delegate void CustomCall5(string a, params object[] args);//变长参数类型根据情况定义


public class L5_CallFunction : MonoBehaviour
{
    // Start is called before the first frame update
    void Start()
    {
        LuaManager.GetInstance().Init();
        LuaManager.GetInstance().DoString("require('Main')");
        
        /*无参无返回*/
        //自定义委托
        CustomCall call = LuaManager.GetInstance().Global.Get<CustomCall>("testFun1");
        call();
        //Unity自带委托
        UnityAction ua = LuaManager.GetInstance().Global.Get<UnityAction>("testFun1");
        ua();
        //C#提供委托
        Action ac = LuaManager.GetInstance().Global.Get<Action>("testFun1");
        ac();
        //xlua提供委托,少用
        LuaFunction lf = LuaManager.GetInstance().Global.Get<LuaFunction>("testFun1");
        lf.Call();

        /*有参有返回*/
         //自定义委托
         CustomCall2 call2 = LuaManager.GetInstance().Global.Get<CustomCall2>("testFun2");
         Debug.Log(call2(10));
         //C#自带的泛型委托
         Func<int, int> sFun = LuaManager.GetInstance().Global.Get<Func<int, int>>("testFun2");
         Debug.Log(sFun(20));
         //xlua提供委托,少用
         LuaFunction lf2 = LuaManager.GetInstance().Global.Get<LuaFunction>("testFun2");
         Debug.Log(lf2.Call(30)[0]);
         
         /*多返回值*/
         //自定义委托out
         CustomCall3 call3 = LuaManager.GetInstance().Global.Get<CustomCall3>("testFun3");
         int b;
         int c;
         bool d;
         string e;
         int f;
         Debug.Log(call3(10, out b, out c, out d, out e, out f));
         Debug.Log(b.ToString()+c.ToString()+d.ToString()+e+f.ToString());
         //自定义委托ref
         CustomCall4 call4 = LuaManager.GetInstance().Global.Get<CustomCall4>("testFun3");
         int b1 = 0;
         int c1 = 0;
         bool d1 = false;
         string e1 = "";
         int f1 = 0;
         Debug.Log(call4(10, ref b1, ref c1, ref d1, ref e1, ref f1));
         Debug.Log(b1.ToString()+c1.ToString()+d1.ToString()+e1+f1.ToString());
         //xlua提供委托,少用
         LuaFunction lf3 = LuaManager.GetInstance().Global.Get<LuaFunction>("testFun3");
         object[] objs = lf3.Call(1000);
         for (int i = 0; i < objs.Length; i++)
         {
             Debug.Log("第"+i+"个参数："+objs[i]);
         }
         
         /*变长参数*/
         //自定义委托out
         CustomCall5 call5 = LuaManager.GetInstance().Global.Get<CustomCall5>("testFun4");
         call5("testFun4", 1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
         //xlua提供委托,少用
         LuaFunction lf4 = LuaManager.GetInstance().Global.Get<LuaFunction>("testFun4");
         lf4.Call("testFun4", 1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

    }

    // Update is called once per frame
    void Update()
    {
        
    }
}

~~~

### List和Dictionary映射Table

~~~Lua
testList = {1,2,3,4,5,6}
testList2 = {"123","123",true,1,2.4}


testDic = {
   ["1"] = 1,
   ["2"] = 2,
   ["3"] = 3,
}

TetsDic2 = {
    ["1"] = 1,
    [true] = 2;
    ["123"] = false;
}
~~~

~~~cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class L6_CallListDic : MonoBehaviour
{
    // Start is called before the first frame update
    void Start()
    {
        LuaManager.GetInstance().Init();
        LuaManager.GetInstance().DoLuaFile("Main");
        
        /*同一类型List*/
        List<int> list = LuaManager.GetInstance().Global.Get<List<int>>("testList");
        foreach (int item in list)
        {
            Debug.Log(item);
        }
        //值拷贝，不会修改原数据
        list[0] = 100;
        List<int> list2 = LuaManager.GetInstance().Global.Get<List<int>>("testList");
        Debug.Log(list2[0]);
        
        /*不指定类型*/
        List<object> list3 = LuaManager.GetInstance().Global.Get<List<object>>("testList2");
        foreach (var item in list3)
        {
            Debug.Log(item);
        }
        
        /*字典指定类型*/
        Dictionary<string, int> dic = LuaManager.GetInstance().Global.Get<Dictionary<string, int>>("testDic");
        foreach (string item in dic.Keys)
        {
            Debug.Log(item + ":" + dic[item]);
        }
        /*字典不指定类型*/
        Dictionary<object, object> dic2 = LuaManager.GetInstance().Global.Get<Dictionary<object, object>>("TetsDic2");
        foreach (object item in dic2.Keys)
        {
            Debug.Log(item + ":" + dic2[item]);
        }

    }

    // Update is called once per frame
    void Update()
    {
        
    }
}

~~~

### 类映射

~~~Lua
testClass = {
    testInt = 2;
    testFloat = 2.4;
    testString = "123";
    testBool = true;
    testFun = function()
        print("testFun")
    end;
}
~~~

~~~cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;


public class CallClass
{
    //必须为public，成员变量可以或多或少，但名称必须和Lua保持一致
    public int testInt;
    public bool testBool;
    public string testString;
    public float testFloat;
    public UnityAction testFun;

    public CallInClass testInClass;
    
    
    public void Test()
    {
        
    }
}

public class CallInClass
{
    public int testInInt;
}

public class L7_CallClass : MonoBehaviour
{
    
    
    
    void Start()
    {
        LuaManager.GetInstance().Init();
        LuaManager.GetInstance().DoLuaFile("Main");
        
        //值拷贝
        CallClass obj = LuaManager.GetInstance().Global.Get<CallClass>("testClass");
        Debug.Log(obj.testInt);
        obj.testFun();
        Debug.Log(obj.testInClass.testInInt);
        
        

    }

    
    void Update()
    {
        
    }
}

~~~

### 接口映射

~~~Lua
testClass = {
    testInt = 2;
    testFloat = 2.4;
    testString = "123";
    testBool = true;
    testFun = function()
        print("testFun")
    end

}
~~~

~~~cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;
using XLua;

[CSharpCallLua]
public interface ICSharpCallInterface
{
    int testInt
    {
        get;
        set;
    }
    float testFloat
    {
        get;
        set;
    }
    string testString
    {
        get;
        set;
    }
    
    bool testBool
    {
        get;
        set;
    }

    UnityAction testFun
    {
        get;
        set;
    }
    
}



public class L8_CallInterface : MonoBehaviour
{
    // Start is called before the first frame update
    void Start()
    {
        LuaManager.GetInstance().Init();
        LuaManager.GetInstance().DoLuaFile("Main");
        
        ICSharpCallInterface obj = LuaManager.GetInstance().Global.Get<ICSharpCallInterface>("testClass");
        Debug.Log(obj.testInt);
        obj.testFun();
        
    }

    // Update is called once per frame
    void Update()
    {
        
    }
}

~~~

### LuaTable映射table

~~~Lua
testTable = {
    testInt = 2;
    testFloat = 2.4;
    testString = "123";
    testBool = true;
    testFun = function()
        print("testFun")
    end

}
~~~

~~~cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using XLua;

public class L9_CallLuaTable : MonoBehaviour
{
    // Start is called before the first frame update
    void Start()
    {
        LuaManager.GetInstance().Init();
        LuaManager.GetInstance().DoLuaFile("Main");
        //不建议使用LuaTable和LuaFunction 效率低
        //引用拷贝，修改值也会影响原来的Lua文件
        LuaTable table = LuaManager.GetInstance().Global.Get<LuaTable>("testTable");
        Debug.Log(table.Get<string>("testString"));
        Debug.Log(table.Get<int>("testInt"));
        table.Get<LuaFunction>("testFun").Call();
        
        
        table.Set("testInt", 100);
        LuaTable table2 = LuaManager.GetInstance().Global.Get<LuaTable>("testTable");
        Debug.Log(table2.Get<int>("testInt"));

    }

    // Update is called once per frame
    void Update()
    {
        
    }
}

~~~

## Lua调用C#

### Lua使用类

~~~cs
#region  类
//自定义类
public class Test
{
    public void Speak(string str)
    {
        Debug.Log("Test1:" + str);
    }
}

namespace MrTang
{
    public class Test2
    {
        public void Speak(string str)
        {
            Debug.Log("Test2:" + str);
        }
    }
}
#endregion
~~~

~~~Lua
print("*********Lua调用C#类相关知识点***********")

--lua中使用C#的类非常简单
--固定套路
--CS.命名空间.类名
--Unity的类 比如 GameObject Transform等等 —— CS.UnityEngine.类名
--CS.UnityEngine.GameObject

--通过C#中的类 实例化一个对象 lua中没有new 所以我们直接 类名括号就是实例化对象
--默认调用的 相当于就是无参构造
local obj1 = CS.UnityEngine.GameObject()
local obj2 = CS.UnityEngine.GameObject("唐老狮")

--为了方便使用 并且节约性能 定义全局变量存储 C#中的类
--相当于取了一个别名
GameObject = CS.UnityEngine.GameObject
local obj3 = GameObject("唐老狮好爱同学们")

--类中的静态对象 可以直接使用.来调用
local obj4 = GameObject.Find("唐老狮")

--得到对象中的成员变量  直接对象 . 即可
print(obj4.transform.position)
Debug = CS.UnityEngine.Debug
Debug.Log(obj4.transform.position)

Vector3 = CS.UnityEngine.Vector3
Vector3 = CS.UnityEngine.Vector3

--如果使用对象中的 成员方法！！！！一定要加:
obj4.transform:Translate(Vector3.right)
Debug.Log(obj4.transform.position)

--自定义类 使用方法 相同  只是命名空间不同而已
local t = CS.Test()
t:Speak("test说话")

local t2 = CS.MrTang.Test2()
t2:Speak("test2说话")

--继承了Mono的类
--继承了Mono的类 是不能直接new 
local obj5 = GameObject("加脚本测试")
--通过GameObject的 AddComponent添加脚本
--xlua提供了一个重要方法 typeof 可以得到类的Type
--xlua中不支持 无参泛型函数  所以 我们要使用另一个重载
obj5:AddComponent(typeof(CS.LuaCallCSharp))
~~~

### Lua使用枚举

~~~cs
#region 枚举
/// <summary>
/// 自定义测试枚举
/// </summary>
public enum E_MyEnum
{
    Idle,
    Move,
    Atk,
}
#endregion
~~~

~~~Lua
print("*********Lua调用C#枚举相关知识点***********")

--枚举调用 
--调用Unity当中的枚举
--枚举的调用规则 和 类的调用规则是一样的
--CS.命名空间.枚举名.枚举成员
--也支持取别名 
--同样 如果报错 需要在CustomSetting中去加上
PrimitiveType = CS.UnityEngine.PrimitiveType
GameObject = CS.UnityEngine.GameObject

local obj = GameObject.CreatePrimitive(PrimitiveType.Cube)

--自定义枚举 使用方法一样 只是注意命名空间即可
E_MyEnum =  CS.E_MyEnum

local c = E_MyEnum.Idle
print(c)
--枚举转换相关
--数值转枚举
local a = E_MyEnum.__CastFrom(1)
print(a)
--字符串转枚举
local b = E_MyEnum.__CastFrom("Atk")
print(b)

~~~
