# 基础
## 1. C#函数中 new 一个struct 对象 会不会产生垃圾回收？

```c#
    public struct CoordsStruct    {
        public CoordsStruct(double x, double y)
        { X = x;
          Y = y;        }
        public double X { get; set; }
        public double Y { get; set; }
        public override string ToString() => $"({X}, {Y})";
    }
    public class CoordsClass    {
        public CoordsClass(double x, double y)
        {   X = x;
            Y = y;        }
        public double X { get; set; }
        public double Y { get; set; }
        public override string ToString() => $"({X}, {Y})";
    }        
CoordsStruct coordStruct = new CoordsStruct(1, 1);
CoordsClass coordsClass = new CoordsClass(1, 1);
```

![image-20220906111404988](LuaC#基础题.assets/image-20220906111404988.png)

![image-20220906111413514](LuaC#基础题.assets/image-20220906111413514.png)

1. `new struct` 调用`ldlocal.s` 向栈压入本地变量地址，之后调用构造函数
2. `new class` 调用 `newobj`  创建一个值类型的新对象，并将对象引用推送到计算堆栈上，对象值放在堆上面
2. 所以new 一个struct对象不产生垃圾回收

```c#
CoordsStruct coordStruct = new CoordsStruct(1, 1);
CoordsClass coordsClass = new CoordsClass(1, 1);

var StructA = coordStruct;
var ClassB = coordsClass;

StructA.X = 1000;
StructA.Y = 1000;
ClassB.X = 2000;
ClassB.Y = 2000;
        // 输出
//coordStruct     (1, 1)
//StructA         (1000, 1000)
//coordsClass     (2000, 2000)
//ClassB          (2000, 2000)
```

由此可看出 new 出来的 struct 对象是值类型，赋值操作会有拷贝（传参也会有拷贝），new class对象是引用。

参考博客：[CLR、内存分配和垃圾回收](https://www.cnblogs.com/dotnet261010/p/9248555.html)

## 2. C#在函数中 new 一个数组 会不会产生垃圾回收？

```c#
        int[] intArray = new int[]{1,2,3};
        char[] charArray = new char[]{'1','2','3'};
        string[] stringArray = new string[]{"12","34","56"};
```

![image-20220906215014438](LuaC#基础题.assets/image-20220906215014438.png)

![image-20220906215059392](LuaC#基础题.assets/image-20220906215059392.png)

无论是int char 还是string数组，都调用了`newarr` 命令：为值申请内存分配在托管堆上，并且添加引用在栈上，**会产生垃圾回收**

## 3. C#在函数中 new 一个struct对象数组 会不会产生垃圾回收？

```c#
        CoordsStruct[] coordStruct = new CoordsStruct[]
        {
            new CoordsStruct(1, 1),
            new CoordsStruct(2, 2),
            new CoordsStruct(3, 3),
        };
        CoordsClass[] coordsClass = new CoordsClass[]
        {
            new CoordsClass(1, 1),
            new CoordsClass(2, 2),
            new CoordsClass(3, 3),
        };
```

![image-20220906215215597](LuaC#基础题.assets/image-20220906215215597.png)

无论是new struct 还是class数组，都调用了`newarr` 和`newobj` 命令，**会产生垃圾回收**。

## 4. unity的部分对象如：Mesh，AnimationClip 等对象 new 出来之后需不需要主动释放？

```c#
public Mesh mesh;
public AnimationClip animationClip;

private int num;
// Start is called before the first frame update
void Start()
{
    mesh = new Mesh();
    animationClip = new AnimationClip();
    num = 0;
}

// Update is called once per frame
void FixedUpdate()
{
    num = num + 1;
    if (num == 50 * 5)
    {
        mesh = null;
        animationClip = null;
        Debug.Log("Set Null num :  " + num);
    }        
    if (num == 50 * 10)
    {
        Destroy(mesh);
        Destroy(animationClip);
        Debug.Log("Destroy both num :  " + num);
    }        
    if (num == 50 * 15)
    {
        GC.Collect();
        Debug.Log("GC.Collect num :  " + num);
    }
```

![image-20220914092056085](LuaC#基础题.assets/image-20220914092056085.png)

将mesh 和 Animation Clip置空引用次数没有归零

![image-20220914095323993](LuaC#基础题.assets/image-20220914095323993.png)

调用`Destory`之后引用次数归零

![image-20220914094408616](LuaC#基础题.assets/image-20220914094408616.png)

## 5. 想办法评估下 Lua和C# 在循环里持续产生内存垃圾对性能的影响
## 6. 加载unity资源：AnimatorController，材质， 贴图，动作文件 需不需要释放，需要的释放使用什么接口？

[Resources和AssetBundle最详细的解析](https://blog.csdn.net/xinzhilinger/article/details/115408934)

1. **Resources**

   1. 一般用 `Resources.Load` 加载 `Asset\Resources`目录下的特定资源，一般还会实例化到场景中。

      1. 资源被加载到Asset中，还会产生一个Clone到Scene Memory中

   2. 关于Resources的方法：

      1. ```
         1. FindObjectsOfTypeAll:返回某一种类型的所有资源
         2. Load：通过路径加载资源
         3. LoadAll：加载该Resources下的所有资源
         4. LoadAsync：异步加载资源，通过协程实现
         5. UnloadAsset：卸载加载的资源
         6. UnloadUnusedAssets：卸载在内存中未使用的资源（整个游戏对象层级视图后未访问到某资源（包括脚本组件），则将其视为未使用的资源）
         ```

   3. ```lua
      public class LoadResourcesTwoWays : MonoBehaviour
      {
          private GameObject modelInstantiate;
          // Start is called before the first frame update
          void Start()
          {
              var obj = Resources.Load("shieldUp");
              
              modelInstantiate = Instantiate(obj) as GameObject;
              modelInstantiate.transform.position = Vector3.zero;
              
              obj = null;
              Resources.UnloadUnusedAssets();
              // LoadResourcesToScene();
              // LoadABResource();
          }
          // Update is called once per frame
          void Update()
          {
              if (Input.GetKey(KeyCode.A))
              {
                  Destroy(modelInstantiate);
              }   
          }
      }
      ```

      没有执行`Destory`的内存，Scene Memory内存占用![image-20220915170138169](LuaC#基础题.assets/image-20220915170138169.png)

      调用`Destory`之后Assets内存里还有，Scene Memory内存从218变成42![image-20220915172601075](LuaC#基础题.assets/image-20220915172601075.png)

   4. 可以使用`DestroyImmediate(object, true)`立即摧毁Asset中的内存

      1. 但是这只会在「只有在Asset，没有在SceneMemory占用」的情况下生效
      2. 如果同时存在两者的占用，那`DestroyImmediate`只能释放在 SceneMemory 中的内存，
      3. 且即使释放了Scene Memory的内存后，**再次使用 **DestroyImmediate 也不能释放在Asset裡面的内存占用

   5. `Resources.UnloadAsset`：卸载 Asset 加载的资源

      1. ```c#
         Resources.UnloadAsset(modelInstantiate);
         //该方法无法释放SceneMemory中的内存（场景中的Clone），只能释放Asset裡面的内存
         ```

   6. `Resources.UnloadUnusedAssets`：卸载在内存中未使用的资源（整个游戏对象层级视图后未访问到某资源（包括脚本组件），则将其视为未使用的资源）

      1. ```c#
         model = null;
         AnimationControlTest = null;
         Resources.UnloadUnusedAssets();
         // 释放所有Resource加载的所有Asset内存
         // 以把Asset和SceneMemory裡的内存一并释放
         ```

   7. 总结：**Resources加载的资源是需要释放**，即使调用了 `Destory(obj)`，也要记得`DestroyImmediate(obj, true)`或者`UnloadAsset(obj)` 或者`UnloadUnusedAssets()` 释放 Assets 里面的内存

      1. 对于不会重複使用的 asset可以加载完之后马上调用`UnloadAsset`
      2. 在场景scene关闭前，调用`DestroyImmediate(obj, true)`清除所有的**SceneMemory**裡的占用
      3. 最后再使用`UnloadUnusedAssets`确认释放所有内存

2. **AssetBundle**

   1. 先打包AB包

      1. ![image-20220915092525579](LuaC#基础题.assets/image-20220915092525579.png)
      2. ![image-20220915101518772](LuaC#基础题.assets/image-20220915101518772.png)
         1. No Compression 不压缩，明显会增大AB包的体积，但是在用的是否加载速度会快很多
         2. LZMA，即所有资源一次性压缩，AB包的体积最小，但是对于资源调用时的速度会慢很多，因为调用任何的一个资源都需要全局解压
         3. LZ4，局部压缩，就是对于每一个资源单独压缩，用的时候就是用到哪一个，就解压哪一个

      3. 打包完成![image-20220915101901530](LuaC#基础题.assets/image-20220915101901530.png)

   2. 卸载AB包加载的资源内存占用的方式有两种：

      1. `(instance)ab.Unload(bool unloadAllLoadedObject)`只释放某个AB自身的内存占有

         1. When `unloadAllLoadedObjects` is **false**,compressed file data inside the bundle itself will be freed, 

            but any instances of objects loaded from this bundle will remain intact

         2. (bundle中的压缩文件被释放，实例化成功的保持原样)

         3. When `unloadAllLoadedObjects` is **true**, all objects that were loaded from this bundle will be destroyed as well. If there are GameObjects in your Scene referencing those assets, the references to them **will become missing**.

         4. (所有对象都要被销毁，scene中的asset引用也会被释放)

      2. `(static)AssetBundle.UnloadAllAssetBundles(bool unloadAllLoadedObject)`释放所有AB的占有内存

      3. 二者 都需要配合 `Destroy`销毁场景中的实例Instance，使用`Unload`销毁Assets中的内存




## 7. MonoBehaviour 数量过多 对性能影响如何，或者一个MonoBehaviour 自带的消耗有哪些？

1. 每一个`Monobehaviour` 都是通过**反射**来调用 生命周期函数 的（`Awake, Update, LateUpdate`等）
   1. Monobehaviour会在游戏开始时首先得到所有写有生命周期函数的脚本，保存下来后再调用这些生命周期方法。
2. 只要在`Monobehaviour`中声明了`Update`这样的生命周期函数，无论函数体裡是否有东西，该方法都会被执行，从对CPU造成一定的负荷

   - 场景上带有`Monobehaviour`且带有各种生命周期（尤其是`Update`函数）的脚本越多，CPU负荷越大，即使所有生命周期的方法体都没有内容

代码中只写一句`public class MonoBehaviourTest : MonoBehaviour{ }`的情况对比![image-20220916144941724](LuaC#基础题.assets/image-20220916144941724.png)

```lua
public class MonoBehaviourTest : MonoBehaviour
{
    private void Awake(){ }
    private void OnEnable(){ }
    void Start(){ }
    private void FixedUpdate(){ }
    private void OnTriggerEnter(Collider other){ }
    private void OnTriggerStay(Collider other){ }
    private void OnTriggerExit(Collider other){ }
    private void OnCollisionEnter(Collision collision){ }
    private void OnCollisionStay(Collision collision){ }
    private void OnCollisionExit(Collision collision){ }
    private void OnMouseDown(){ }
    private void OnMouseUp(){ }
    void Update(){ }
    private void LateUpdate(){ }
    private void OnRenderImage(RenderTexture source, RenderTexture destination){ }
    private void OnDisable(){ }
    private void OnDestroy(){ }
    private void OnApplicationQuit(){ }
}
```

测试代码声明内置生命周期函数后，性能消耗有明显上升

![image-20220916150233138](LuaC#基础题.assets/image-20220916150233138.png)

## 8. Lua的底层数据类型有哪些？

```
/*
** basic types
*/
#define LUA_TNONE       (-1)
#define LUA_TNIL        0
#define LUA_TBOOLEAN        1
#define LUA_TLIGHTUSERDATA  2
#define LUA_TNUMBER     3
#define LUA_TSTRING     4
#define LUA_TTABLE      5
#define LUA_TFUNCTION       6
#define LUA_TUSERDATA       7
#define LUA_TTHREAD     8
#define LUA_NUMTAGS     9
```

| 常用数据类型 | 描述                                                        |
| :------- | ------------------------------------------------------------ |
| nil      | 空值                                                         |
| boolean  | 布尔值：false和true。                                        |
| number   | 双精度浮点数                                                 |
| string   | 字符串（可用双引号或单引号）                                |
| function | 函数类型：由 C 或 Lua 编写的函数                             |
| userdata | 主要用来表示在C/C++中定义的类型，即用来实现扩展lua，这些扩展代码通常是用C/C++来实现的。对lua 虚拟机来说userdata提供了一块原始的内存区域 |
| thread   | 用于执行协同程序   [协同程序（线程thread）](https://www.cnblogs.com/Richard-Core/p/4373582.html) |
| table    | Lua 中的表（table）是一个"关联数组"（associative arrays），数组的索引可以是数字、字符串或表类型。在 Lua 里，table 的创建是通过"构造表达式"来完成，最简单构造表达式是{}，用来创建一个空表。 |

| 不常用数据类型 | 描述                                                         |
| :------: | ------------------------------------------------------------ |
|      none      | 仅用于 C API    [Is 'none' one of basic types in Lua?](https://stackoverflow.com/questions/38562720/is-none-one-of-basic-types-in-lua) |
| lightUserData | 轻量级用户数据是表示 C 指针的值（即`void *`值） [Light Userdata](https://www.lua.org/pil/28.5.html) |
| NUMTYPES | 双精度浮点数                                                 |

 `string` `table ` `function` `thread` 四种在 vm 中以**引用**方式共享，是需要被 GC 管理回收的对象。其它类型都以值形式存在。——[Lua GC 的源码剖析 (1)](https://blog.codingnow.com/2011/03/lua_gc_1.html)

## 9. 为什么说Lua一切皆Table,Table有哪两种存储形式，Table是如何Resize的，了解Resize的代价

1. Lua的table由 **数组部分（array part）**和 **哈希部分（hash part）**组成。
   1. 数组部分索引的key是1~n的整数，
   2. 哈希部分是一个哈希表，哈希表本质是一个数组，它利用哈希算法将键转化为数组下标，若下标有冲突，则会将冲突的下标上创建一个链表，用链地址法解决哈希冲突。
   3. table的 key 值可以是除了 nil 之外的任何类型的值
2. 向table中插入数据时，如果table满了，table会重新设置数据部分 和 哈希表的大小，**容量是成倍增加的（C++ vector）**，哈希部分还要对哈希表中的数据进行整理。
   1. 没有赋初始值的table，数组和部分哈希部分默认容量为0。

```c
//luaC语言源码: ltable.c
void luaH_newKey(lua_State *L, Table *t, const TValue *key, TValue *value){
        mp = mainpositionTV(t, key);
        if (!isempty(gval(mp)) || isdummy(t)) {  /* main position is taken? */
        Node *othern;
        Node *f = getfreepos(t);  /* get a free place */
        if (f == NULL) {  /* cannot find a free place? */
          rehash(L, t, key);  /* grow table */	// 重置哈希
          /* whatever called 'newkey' takes care of TM cache */
          luaH_set(L, t, key, value);  /* insert key into grown table */
          return;
        }
}
static void rehash (lua_State *L, Table *t, const TValue *ek) {
          unsigned int asize;  /* optimal size for array part 数组最佳大小*/
          unsigned int na;  /* number of keys in the array part 数组部分的 key（index） 的数量*/
          unsigned int nums[MAXABITS + 1];
          int i;
          int totaluse;
          for (i = 0; i <= MAXABITS; i++) nums[i] = 0;  /* reset counts */
          setlimittosize(t);
          na = numusearray(t, nums);  /* count keys in array part */
          totaluse = na;  /* all those keys are integer keys */
          totaluse += numusehash(t, nums, &na);  /* count keys in hash part */
          /* count extra key */
          if (ttisinteger(ek))
            	na += countint(ivalue(ek), nums);
          totaluse++;
          /* compute new size for array part */
          asize = computesizes(nums, &na);
          /* resize the table to new computed sizes */
          luaH_resize(L, t, asize, totaluse - na);
}
** Compute the optimal size for the array part of table 't'. 'nums' is a
** "count array" where 'nums[i]' is the number of integers in the table
** between 2^(i - 1) + 1 and 2^i. 'pna' enters with the total number of
** integer keys in the table and leaves with the number of keys that
** will go to the array part; return the optimal size.  (The condition
** 'twotoi > 0' in the for loop stops the loop if 'twotoi' overflows.)
*/
static unsigned int computesizes (unsigned int nums[], unsigned int *pna) {
      int i;
      unsigned int twotoi;  /* 2^i (candidate for optimal size) */
      unsigned int a = 0;  /* number of elements smaller than 2^i */
      unsigned int na = 0;  /* number of elements to go to array part */
      unsigned int optimal = 0;  /* optimal size for array part */
      /* loop while keys can fill more than half of total size */
      for (i = 0, twotoi = 1;
           twotoi > 0 && *pna > twotoi / 2;
           i++, twotoi *= 2) {
        a += nums[i];
        if (a > twotoi/2) {  /* more than half elements present? */
          optimal = twotoi;  /* optimal size (till now) */
          na = a;  /* all elements up to 'optimal' will go to array part */
    }
      }
      lua_assert((optimal == 0 || optimal / 2 < na) && na <= optimal);
      *pna = na;
      return optimal;
}
```

1. 从c语言源码中可以看出，获得的新table长度在`2^(i - 1) + 1 and 2^i`之间,
2. **Lua Table在非构造阶段，不论是Array还是Hash部分都是以2的幂次增加的（事实上在构造阶段Hash部分也只能按2的幂次增加）。每当扩容以后，原数据会重新再插入新的内存块中。**
3. 由 C++中vector满时，申请新内存地址造成的重大开销，建议初始化阶段赋值 或者 开辟预计大小空间初始化。
4. `resize`代价很高，当我们把一个新键值赋给表时，若数组和哈希表已经满了，则会lua在申请内存基础上还需要 重置哈希(`rehash`)。
5. 重置哈希的代价是高昂的。首先会在内存中分配一个新的长度的数组，然后将所有记录再全部哈希一遍，将原来的记录转移到新数组中。

```
-- 测试内存占用
collectgarbage("stop")
local mem = collectgarbage("count")
tab = {}
LogInfoFormat("emery table is: %d \t memory usage: %s ", 0, (collectgarbage("count") - mem) * 1024)
for i = 1,10 do
    tab[i] = i
    LogInfoFormat("table length : %d \t memory usage: %s ", #tab, (collectgarbage("count") - mem) * 1024)
end
```

![image-20220909144753412](LuaC#基础题.assets/image-20220909144753412.png)

## 10. table 遍历有几种形式 有什么不同

1. 第一种：pairs迭代器

   - 对所有 键值key遍历 通过`next()`函数判断下一个元素

   - 但是会出现随机遍历，可能不会按顺序遍历

   - 也可以用 `tabName.next()` 测试是不是空表

   - ```lua
     tab = {a = "qwe", b = "asd", 3, 2, 1, c = "zxc"}
     for k, v in pairs(tab) do
     	print(k, v)
     end
     tab.d = "qweasd"
     print("添加新索引")
     for k, v in pairs(tab) do
     	print(k, v)
     end
     
     table.remove(tab,"b")
     print("remove索引")
     for k, v in pairs(tab) do
     	print(k, v)
     end
     输出
     a	qwe
     b	asd
     c	zxc
     3	1
     2	2
     1	3
     添加新索引
     a	qwe
     b	asd
     c	zxc
     3	1
     2	2
     1	3
     d	qweasd
     remove索引
     a	qwe
     c	zxc
     3	1
     2	2
     1	3
     d	qweasd
     ```

2. 第二种：ipairs迭代器

   - 遍历数组，顺序遍历，如果中间key有空，则不会遍历后面的

   - ```lua
     tab = {}
     for i= 4,1,-1 do
         tab[i] = i
     end
     tab[6] = "asdqwe"
     for k, v in ipairs(tab) do
     	print(k, v)
     end
     输出
     1	1
     2	2
     3	3
     4	4
     ```
     
     

## 11. 详细描述下项目使用的 class 机制

1. 声明全局变量 `_G["__class"] ` 让C#调用

```lua
if _G["__class"] == nil then
	_G["__class"] = {}	--_class作为index，赋值为一个空表
end
local _class = _G["__class"] or {}
```

![image-20220913212055477](LuaC#基础题.assets/image-20220913212055477.png)

```lua
local rawset = rawset
local setmetatable = setmetatable


-- 输出如果index值是固定结构报错
local function __disable_newindex(t, key, value)
	LogErrorFormat('properties is fix struct, forbid new index (new index：%s value：%s)', key, value)
end
-- 处理新的index
local function __properties_newindex(t, key, value)
	local properties = t.Properties
	if properties[key] == nil then
		rawset(t, key, value)
	else
		properties[key] = value
	end
end
-- 构造函数，如果有父类，则可以再次调用 本构造函数
local function __ctor(obj, class_type, ...)
	if class_type.super then
		__ctor(obj, class_type.super, ...)
	end
	if class_type.ctor then
		class_type.ctor(obj, ...)
	end
end
-- 通过父类来初始化，实现继承
local function __init_super(obj, self_class_type, ...)
	local typeSuper = self_class_type.super
	while typeSuper ~= nil do
		local typeSuperL = typeSuper
		local objSuper = {}
		obj[typeSuper] = objSuper	--将父类设置到这个该对象的 self_class_type.super 索引里
		setmetatable(objSuper, {__index=	-- 设置元表，该被访问时，会调用下列的函数
			function(t, k)
				local ret=_class[typeSuperL][k]	--得到父类的 继承函数
				if type(ret) == "function" then
					local func = ret
					ret = function(self1, ...)
						return func(obj, ...)
					end
					t[k] = ret	-- 如果是函数类型，保存下父类继承函数，没有就nil
				else
					ret = nil
				end
				return ret
			end
		})
		typeSuper = typeSuper.super
	end
end 

function class(super, enable_properties)
       ---@class BaseClass
       local class_type={}	-- class_type 类模板 
       class_type.ctor=false
       class_type.super=super	-- 父类赋值
       class_type.new=function(self_class_type, ...)	-- 定义new成员方法
              local obj= {}
              setmetatable(obj, _class[class_type].__Metatable)
              -- 多重继承调用指定父类方法设定
              __init_super(obj, self_class_type, ...)	-- 调用上面的函数 初始化父类
              __ctor(obj, class_type, ...)			  -- 调用上面的 构造函数
              return obj
       end

       if enable_properties then	
              class_type.new_with_properties = function(self_class_type, properties, ...)
                     local obj= {Properties = properties}
                     local mt = {__index = properties, __newindex = __properties_newindex }	
            	--元表设置索引的时候调用properties，赋值操作table[key] = value调用 __properties_newindex
                    --设置元表
            	setmetatable(obj, mt)
                     setmetatable(properties,  _class[class_type].__PropertiesMetatable)

                     -- 多重继承调用指定父类方法设定
                     __init_super(obj, self_class_type, ...)
                     __ctor(obj, class_type, ...)
                     return obj
              end
       end
    
	-- vtbl可以理解为类容器
       local vtbl=
       {
          __is_class = true,
          is_class_type = function(type)
                 -- 调用函数类型 和 成员表类型 是否相同
                 if class_type == type then
                    	return true
                 end
                 local typeSuper = class_type.super
                 while typeSuper ~= nil do
                        if typeSuper == type then
                           	return true
                        end
                        typeSuper = typeSuper.super
                 end
                 return false
          end,
	--  声明成员变量
          DeclareVar = function(obj, name, value)
             if obj[name] ~= nil then 
                error(string.format("成员变量:%s 已存在，声明失败", name))
                return
             end
             rawset(obj, name, value)
          end
       }

       vtbl.__Metatable =  {__index = vtbl}
       if enable_properties then
          vtbl.__PropertiesMetatable = { __index = vtbl, __newindex = __disable_newindex }
       end

       _class[class_type]= vtbl
	-- 设置 class_type 赋值的时候调用该函数
       setmetatable(class_type,{__newindex=
          function(t,k,v)
             vtbl[k]=v
          end
       })
    
       if super then
        --如果有父类 父类给vtbl的super赋值
          vtbl.__super = super
        -- 如果没有这个成员，就去父类里面找
          setmetatable(vtbl,{__index=
             function(t,k)
                local ret=_class[super][k]
                vtbl[k]=ret
                return ret
             end
          })
       end

       return class_type
end
```

## 12. 大体 了解下Lua的GC机制
## 13. Lua的全局变量跟local变量的区别，Lua是如何查询一个全局变量的，local的变量的作用域规则是怎么样的？
## 14. 详细说明元表的机制和它的一些特性
## 15. lua 那些情况会产生 垃圾回收
## 16. C# 不想让外部使用new的方式 生成对象，有什么办法？
## 17. 非运行时的代码比如OnDrawGizmos 里面 new出来的资源，删除时机的隐患是什么（注意不加编辑器运行的属性，OnDisable，OnDestroy是不会执行的）
## 18. 什么是lua的尾调用，它有什么作用？
## 19. 评估Lua字符串拼接的消耗，有没效率更高的拼接方式
## 20. lua 表的2种初始化方式：哪种性能高 差多少 ？ 为什么？
1. 第一种
1. for i = 1, 1000000do
    2. locala ={}
    3. 　　a[1] =1; a[2] =2; a[3] =3
    4. end
2. 第二种：
1. for i = 1, 1000000do
    2. locala = {true,true,true}
    3. 　　a[1] =1; a[2] =2; a[3] =3
    4. end
## 21. 测试一下以下2种代码的性能差异
1. 第一种：
1. for i = 1, 1000000do
    2. localx =math.sin(i)
    3. end
2. 第二种
1. localsin =math.sin fori =1,1000000do localx =sin(i) end
## 22. local 变量的访问，在函数中定义直接访问和在文件中定义 性能上有何区别？