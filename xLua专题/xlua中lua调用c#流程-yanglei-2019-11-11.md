### 杂项 ###

#### C#调用Lua ####

xLua 扩展了Lua加载器 支持在 C#中调用Unity Resources文件夹中的lua文件
例如 在Resources文件夹中 有一个lua脚本 test.lua.txt 可以这样调用
```
var luaenv = new LuaEnv()
luaenv.DoString("require('test')","test")
```

#### Lua虚拟机 栈 注册表 ####

##### Lua虚拟机 #####
模拟计算机执行模型 执行脚本 其核心功能有两个:
+ 分析脚本源码生成字节码 然后执行字节码
+ 维护整个执行环境

Lua创建虚拟机API
```
lua.h
LUA_API lua_State *(lua_newstate) (lua_Alloc f, void *ud);
```
及辅助库提供的默认实现
```
lauxlib.h
LUALIB_API lua_State *(luaL_newstate) (void);
```

两个API会初始化lua虚拟机环境

虚拟机初始化会申请内存创建 main thread, global state 及内部的初始化
包括 虚拟机 注册表 内部模块的初始化 然后返回一个lua_State对象作为主现成对象

##### 栈 ######
lua通过栈来与C交互沟通 实现跨语言的交流 每个栈元素都是一个lua对象
C调用Lua 方式:将lua函数压栈 然后将参数依次压栈 使用lua_call调用 此时参数及函数入栈 结果被压栈
Lua调用C 方式:将被调C函数包装成typedef int (*lua_CFunction) (lua_State *L) 形式 Lua保证调用时
栈内已压入参数 通过lua_gettop(L)获取 如需将结果传给lua 将结果入栈 最后返回结果个数 lua保证结果之下的
栈数据被丢调

##### 注册表 #####

初始化lua虚拟机时初始化的一个lua table 为C提供一个保存lua值的途径
初始化时lua已经占用了两个key 分别是"main state"(key = 1)和"全局环境"(key = 2)
解释下全局环境:是每个lua代码块缺省的使用全局环境做外部环境 比如在代码块中访问table.insert()
其实就是访问全局环境的数据 注册表的整数键禁止乱用 使用luaL_ref() 和lua_unref管理

#### Lua调用C# ####
实现Lua调用C#的思路如下

- 将C的函数包装成Lua环境认可的函数(在C#中就是将C#代码元素以已Lua认可的方式实现)
- 将包装好的函数注册到Lua环境中
- 像使用普通Lua函数那样使用注册函数

>CS.UnityEngine.Debug.Log('hello world')

使用者视角概述：
    如上述代码 调用时XLua会检测这个类是否已经注册过了。如是 直接调用完成操作
否则 检查是否有注册的离线生成的胶水代码 有就加载wrap代码 无胶水代码就使用反射
的方式解析类型的字段 方法 事件 等注册到Lua中

下面是对上述原理的实现
XLua中初始化LuaEnv时会创建一个Lua虚拟机 并xlua库注册到Lua中 这样我们可以像使用
内置库一样使用xlua的方法 

然后会在在C#中执行如下一段代码 设置全局表CS的元表自定义其查找机制
```
init_xlua.lua
local metatable = {}
local rawget = rawget
local setmetatable = setmetatable
local import_type = xlua.import_type
function metatable:__index(key)
    -- 这里传入的时元表对应表的key 真实查找 不触发元表
    local fqn = rawget(self,'.fqn')
    -- 查找结果非nil说明是缓存的程序集 例如:UnityEngine
    fqn = ((fqn and fqn .. '.') or '') .. key
    -- 传入XLua初始化时注册的方法 这里会跳转到C#中 也就是将C#注册到Lua的实现啦
    local obj = import_type(fqn)
    if obj == nil then
        -- 可能是程序集 缓存
        obj = { ['.fqn'] = fqn }
        setmetatable(obj, metatable)
    elseif obj == true then
        -- 注册过了 可以直接取
        return rawget(self, key)
    end
    rawset(self, key, obj)
    return obj
end
CS = CS or {}
setmetatable(CS, metatable)
...
```
其中比较重要的就是xlua.import_type() 当第一次调用某个C#类型时 会调用这个函数进行注册 其实现在C# 如下 

可以看到该函数签名符合Lua的要求
```
StaticLuaCallbacks.cs
public static int ImportType(RealStatePtr L)
{
    ObjectTranslator translator = ObjectTranslatorPool.Instance.Find(L);
    string className = LuaAPI.lua_tostring(L, 1);
    Type type = translator.FindType(className);//根据类名取类型 这里是纯C#操作
    if (type != null)
    {
        /*
        * 根据类型取typeId 这个id对应在Lua注册表保存的一张表
        * XLua使用lazy load方式 首次调用此类型才会注册表数据
        */
        if (translator.GetTypeId(L, type) >= 0)
        {
            LuaAPI.lua_pushboolean(L, true);
        }
        else
        {
            return LuaAPI.luaL_error(L, "can not load type " + type);
        }
    }
    else  //没找到类型 说明传的时程序集名 直接返nil
    {
        LuaAPI.lua_pushnil(L);
    }
    return 1;
}
```
这里说明一下 注册过了会将类型对应table压栈并返回 但是未注册就需要进行加载
```
ObjectTranslator.cs
public bool TryDelayWrapLoader(RealStatePtr L, Type type)
{
    if (loaded_types.ContainsKey(type)) return true;
    loaded_types.Add(type, true);
    //在注册表中添加类型对应表 如已有就取已有的 然后入栈
    LuaAPI.luaL_newmetatable(L, type.FullName); //先建一个metatable，因为加载过程可能会需要用到
    LuaAPI.lua_pop(L, 1);
    Action<RealStatePtr> loader;
    int top = LuaAPI.lua_gettop(L);         
    if (delayWrap.TryGetValue(type, out loader))
    {
        delayWrap.Remove(type);
        //适配代码加载
        loader(L);
    }
    else
    {
        ...
        //反射加载
        Utils.ReflectionWrap(L, type, privateAccessibleFlags.Contains(type));
        ...
    }
    ...        
    return true;
}
```
加载之后已经生成了一张包含类型全部C#元素实现的table 然后将其设置到CS.UnityEngine.Debug
中就可以调用了

下面利用Vector3来具体看看我们调用C#对象时的具体执行步骤
选择Vector3是因为其在游戏开发中很常用 也是几大Lua插件对(si)比(bi)的重点之一
需要说明的是 XLua支持对C#值类型的GC优化 可以避免对Vector3的操作产生GC

```
---Lua端的一次调用实现如下
CS.UnityEngine.Vector3(10,10,10)

// UnityEngineVector3Wrap.cs
// 构造器符合Lua调用标准 其注册机制如上述 
static int __CreateInstance(RealStatePtr L){
    // 取lua端的x,y,z
    float _x = (float)LuaAPI.lua_tonumber(L, 2);
    float _y = (float)LuaAPI.lua_tonumber(L, 3);
    float _z = (float)LuaAPI.lua_tonumber(L, 4);
    // 根据lua的x,y,z 构造一个 vector3                    
    UnityEngine.Vector3 gen_ret = new UnityEngine.Vector3(_x, _y, _z);
    /*
    　* 将构造的vector3 作为userdata传给lua
      * 传递到lua 这个对象旋即被丢弃
    */
    translator.PushUnityEngineVector3(L, gen_ret); /*ObjectTranslator在XLua中是一个非常重要的概念 作用是提供Lua与CSharp交互的抽象*/
    ...
}

//WrapPusher.cs
void PushUnityEngineVector3(RealStatePtr L, UnityEngine.Vector3 val){
    /*
     * 首先就是找到注册表上UnityEngine.Vector对应的表,设为push的userdata的元表
     * 这张表里有UnityEngine.Vector对应的方法
     * getTypeId就是上述的注册操作
    */
    if (UnityEngineVector3_TypeID == -1) {
        bool is_first;
        UnityEngineVector3_TypeID = getTypeId(L, typeof(UnityEngine.Vector3), out is_first);     
    }
    //userdata 入栈 设元表 set数据
    IntPtr buff = LuaAPI.xlua_pushstruct(L, 12, UnityEngineVector3_TypeID);
    if (!CopyByValue.Pack(buff, 0, val)) {
        throw new Exception("pack fail fail for UnityEngine.Vector3 ,value="+val);
    }
}

//PackUnPack.cs
bool Pack(IntPtr buff, int offset, UnityEngine.Vector3 field){
    /*
     * 将构造的vector3的 x,y,z传给 Lua对应的结构
     * 这里会对push到lua交互栈struct做一次校验
    */
    if(!LuaAPI.xlua_pack_float3(buff, offset, field.x, field.y, field.z)){
        return false;
    }     
    return true;
}
```
>上述代码中LuaAPI.xxx是通过C#提供的PInvoke机制调用C代码 这些代码在xlua.c中 XLua已经为我们编译好了

```
xlua.c
// 这个结构用于存储值类型
typedef struct {
	int fake_id;
    unsigned int len;
	char data[1];
} CSharpStruct;

//这里会将vector3的数据存到一个userdata用Lua调用
LUA_API void *xlua_pushstruct(lua_State *L, unsigned int size, int meta_ref) {
	CSharpStruct *css = (CSharpStruct *)lua_newuserdata(L, size + sizeof(int) + sizeof(unsigned int));
	css->fake_id = -1;
	css->len = size;
    lua_rawgeti(L, LUA_REGISTRYINDEX, meta_ref);
	lua_setmetatable(L, -2);
	return css;
}
LUALIB_API int xlua_pack_float3(void *p, int offset, float f1, float f2, float f3) {
	CSharpStruct *css = (CSharpStruct *)p;
	if (css->fake_id != -1 || css->len < offset + sizeof(float) * 3) {
		return 0;
	} else {
		float *pos = (float *)(&(css->data[0]) + offset);
		pos[0] = f1;
		pos[1] = f2;
		pos[2] = f3;
		return 1;
	}
}
```
至此 便在Lua中完成了一个Vector3的构造工作

优化策略:

- 可以看到整个调用过程是在第一次使用才在Lua内注册 注册使用胶水代码和反射两种机制
反射的消耗更大 所以尽量避免反射
- C#的值类型如之上的Vector3使用属性标签进行优化 避免装箱拆箱操作
