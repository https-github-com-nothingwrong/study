### 杂项 ###

#### C#调用Lua ####

xLua 扩展了Lua加载器 支持在 C#中调用Unity Resources文件夹中的lua文件
例如 在Resources文件夹中 有一个lua脚本 test.lua.txt 可以这样调用
```
var luaenv = new LuaEnv()
luaenv.DoString("require('test')","test")
```