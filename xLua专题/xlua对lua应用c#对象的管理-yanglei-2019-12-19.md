# XLua C#对象管理 #
C#端 map[obj] = index, objPool[index] = obj

C端 注册表的一个元素 table[index] = userdata

XLua中在Lua中操作C#对象时,要先将其push到Lua栈中
push操作时会从map中找index 根据index 从 C中table查找缓存userdata
缓存命中失败再在Lua中新建 并缓存到上述数据结构中(代码不放了 基本就是对上述结构的操作)


关于对象回收, 出XLua特殊处理 C#对象在Lua以userdata表示 尺寸为sizeof(int)
其创建过程中设置了知道其行为的元表 其中有"__gc"  还有一个标记xlua_tag
lua中对象回收会触发"__gc" 元方法 删掉C#的缓存
```
// StaticLuaCallbacks.cs
public static int LuaGC(RealStatePtr L)
{
    ...
    int udata = LuaAPI.xlua_tocsobj_safe(L, 1);// 验证这个userdata是否未XLua管理的C#对象
    if (udata != -1)
    {
        ObjectTranslator translator = ObjectTranslatorPool.Instance.Find(L);
        if ( translator != null )
        {
            translator.collectObject(udata); //udata就是传给C端的index 这里会删除C#的相应缓存
        }
    }
    return 0;
    ...
}

//xlua.c
LUA_API int xlua_tocsobj_safe(lua_State *L,int index) {
	int *udata = (int *)lua_touserdata (L,index);
	if (udata != NULL) {
		if (lua_getmetatable(L,index)) {
		    lua_pushlightuserdata(L, &tag); //元素元表有tag元素说明是XLua的C#对象 
			lua_rawget(L,-2);
			if (!lua_isnil (L,-1)) {
				lua_pop (L, 2);
				return *udata;//取userdata的值 即上述的index
			}
			lua_pop (L, 2);
		}
	}
	return -1;
}
```

# 进程 线程 协程 的概念及意义 #
## 进程 ##
操作系统对一个正在执行的程序的抽象 操作系统有(PCB结构描述进程) 主要包括:寄存器 执行栈 进程状态 进程id 堆数据 io状态数据等
现代主流操作系统中 对进程的定位是:线程的容器   每个进程至少包含一个线程

## 线程 ##
操作系统执行调度的基本单位 (与PCB对应有TCB结构描述线程) 包含在进程中 同一进程的多个线程共享 虚拟内存地址 文件描述符等 但执行上下文是独立的

## 协程 ## 
协程的使用不包含系统调用 其调用有程序自身维护 Lua的协程为协作式而非抢占式 其调度由客户管理