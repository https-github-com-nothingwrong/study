# HotFix是什么 #
XLua提供的特性 可以用Lua函数替换C#的实例构造函数，方法，属性，事件,析构函数的实现  

# HotFix怎么用 #
HotFix具体实现步骤如下
+ 代码生成
+ IL注入
+ 注册Lua补丁代码

> 项目开发中前两部需添加到打包流程中 XLua也提供Editor选项 一般都是打补丁用 平时开发不用开
> XLua也提供了Example

1.代码生成 
就是生成我们平时生成胶水代码 需要HotFix的C#类型要加 HotFix特性 根据方法签名生成wrap方法

2.IL注入
实现热更代码的关键步骤 在生成的IL中间码中添加代码 每次编译都要做 不然就不会生效

3.注册Lua补丁
将补丁代码在Lua实现 使用xlua.hotfix()或者 util.hotfix_ex()注册 再次调用就会走Lua重写的版本了,即实现了代码热更

# HotFix的实现原理 #
## IL概念及修改 ##
IL(Common Intermediate Language) .net框架支持的低阶编成语言 算是高级汇编语言

Mono.Cecil库 读取 修改 .net程序集元数据 

Unity代码生成 Untiy项目Build 对脚本可选Mono JIT AOT Full Aot 和IL2CPP
但是无论哪种选项首先就是调用编译器将脚本编译为IL中间码

> XLua 通过Mono.Cecil可以修改Untiy 脚本程序集的IL代码 实现代码热更

## XLua的具体代码实现 ##

1.代码生成
XLua根据HotFix特性生成对应签名的方法 完成Lua调用 具体位置在Gen/DelegatesGensBridge.cs
命名为:__Gen_Delegate_Imp%d

2.IL注入 
修改编译生成的IL代码 添加条件判断 确定是否调用Lua

具体代码在Hotfix.cs 中

bool injectMethod(MethodDefinition method, HotfixFlagInTool hotfixType){
    // 这个方法会修改传入方法的IL指令  具体实现略复杂 这里就不放了
}

这里举个简单例子 来说明添加了哪些代码 

```
public class A {
    int Add(int a,int b) {
        return a + b；
    }
}

public class A {
    private static XLua.DelegateBridge __Hotfix0_Add = null;
    int Add(int a,int b) {
        if (__Hotfix0_Add) {
            return __Hotfix0_Add.__Gen_Delegate_Imp%d(a,b); //这里调用签名匹配的方法 注意:类实例方法会传对象自身为首个参数
        } else {
            return a + b;
        }
    }
}
```

需要说明以上操作均在编译期完成 也是支持代码热更的前提条件

3.在Lua注册热更代码
    XLua提供了两个注册接口:

---@param class table or string CS类型会对应字符串

---@param methed_name table or string  方法名或类字段补丁实现

---@param fix function methed为string时 对应的lua实现

xlua.hotfix(class, [method_name], fix)  //支持对单个方法或类元素的打补丁

---@param class table or string CS类型会对应字符串

---@param methed_name string  方法名

---@param fix function 对应的lua实现

util.hotfix_ex(class, method_name, fix)  //仅支持对单个方法打补丁 区别是此补丁会在原生实现执行后调用

xlua是在LuaEnv初始化时注册的 
```
xlua.hotfix = function(cs, field, func)
    if func == nil then func = false end
    local tbl = (type(field) == 'table') and field or {[field] = func}
    for k, v in pairs(tbl) do
        local cflag = ''
        if k == '.ctor' then
            cflag = '_c'
            k = 'ctor'
        end
        local f = type(v) == 'function' and v or nil
        xlua.access(cs, cflag .. '__Hotfix0_'..k, f)  //将引用Lua function的 DelegateBridge赋给C#类型的注入字段
        pcall(function()
            for i = 1, 99 do
                xlua.access(cs, cflag .. '__Hotfix'..i..'_'..k, f)
            end
        end)
    end
    xlua.private_accessible(cs)  //将类型注册到Lua
end
```

这里的xlua.access()　xlua.private_accessible() 实现在StaticLuaCallback.cs 中
> public static int XLuaAccess(RealStatePtr L);       
> public static int XLuaPrivateAccessible(RealStatePtr L);

注意 这两个方法均涉及类型反射 消耗相对较高

util.hotfix_ex() 是xlua提供的工具 选用 用的话要require一下 具体实现在util.lua.txt中
原理就是把func包一下
先清除热补丁 这样就会直接调用原生方法
后调用传入func 然后把这个包装函数设为补丁函数
```
local function hotfix_ex(cs, field, func)
    assert(type(field) == 'string' and type(func) == 'function', 'invalid argument: #2 string needed, #3 function needed!')
    local function func_after(...)
        xlua.hotfix(cs, field, nil)
        local ret = {func(...)}
        xlua.hotfix(cs, field, func_after)
        return unpack(ret)
    end
    xlua.hotfix(cs, field, func_after)
end
```

至此 hotfix的实现完成

# 总结 #

## 开销 ##
加了热更标签但未注册的 由上述可知添加了条件判断分支 会加几条指令 具体开销视占比看实现的复杂度 越复杂的实现影响越小

注册了补丁后 因为涉及到C#调用C 很容易触发marshaling 会大幅增加开销

因此在补丁代码的实现中尽量保持两个原则:

1.尽量减少跨语言调用  2.跨语言调用的传参尽可能传值类型

利用上述事例代码做耗时测试 运行 100000000次  代码很简单这里就不贴了

实验结果:
Lua虚拟机运行与XLua纯Lua实现的调用耗时接近 均为纯C#调用的2倍左右

C#注入比未注入耗时增减10%~20%

Hotfix注册调用方式耗时是C#的30倍左右 

就本例来看 究其原因 是对象实例方法push了对象本身(一个引用类型)

```
public int __Gen_Delegate_Imp15(object p0, int p1, int p2)
{
#if THREAD_SAFE || HOTFIX_ENABLE
    lock (luaEnv.luaEnvLock)
    {
#endif
        RealStatePtr L = luaEnv.rawL;
        int errFunc = LuaAPI.pcall_prepare(L, errorFuncRef, luaReference);
        ObjectTranslator translator = luaEnv.translator;
        translator.PushAny(L, p0);      //这个调用占据整个调用开销的近80% 里面涉及XLua的对象管理 值类型优化 类型注册等 
        LuaAPI.xlua_pushinteger(L, p1);
        LuaAPI.xlua_pushinteger(L, p2);

        PCall(L, 3, 1, errFunc);


        int __gen_ret = LuaAPI.xlua_tointeger(L, errFunc + 1);
        LuaAPI.lua_settop(L, errFunc - 1);
        return __gen_ret;
#if THREAD_SAFE || HOTFIX_ENABLE
    }
#endif
}
```

## 局限性 ##
+ 需要预埋Hotfix特性 否则就无法使用hotfix 即使对此做了规划部署 也很难尽善尽美 另一种比较暴力的方式是全部预埋 但是会有代码段增加和效率下降的问题 灵活度不够 
+ 复杂需求hotfix实现难度大 使用Lua实现业务逻辑时 即时使用反射 也很难保证运行时上下文的数据都可以正常访问 修复一些复杂逻辑的bug或单纯通过hotfix在Lua中做二次开发的局限性很高

> 我们项目使用过xlua.private_accessible(type) 反射注册C#的类型元素进行代码修复 当然这也不算一个成熟的解决方案 因为此方式需要一个脚本的嵌入点

## 使用建议 ##
符合两个条件:
1.经常变动 2.对效率要求不高 

项目初期就对HotFix的使用范围做规划

XLua配置属性集中管理 

Lua端对hotfix的注册操作统一管理 