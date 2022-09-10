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



## 5. 想办法评估下 Lua和C# 在循环里持续产生内存垃圾对性能的影响
## 6. 加载unity资源：AnimatorController，材质， 贴图，动作文件 需不需要释放，需要的释放使用什么接口？
## 7. MonoBehaviour 数量过多 对性能影响如何，或者一个MonoBehaviour 自带的消耗有哪些？
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
//ltable.c
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

1. 从c语言源码中可以看出，获得的新table长度在`2^(i - 1) + 1 and 2^i`之间
2. 由 C++中vector满时，申请新内存地址造成的重大开销，建议初始化阶段赋值 或者 开辟预计大小空间初始化。
3. `resize`代价很高，当我们把一个新键值赋给表时，若数组和哈希表已经满了，则会lua在申请内存基础上还需要 重置哈希(`rehash`)。
4. 重置哈希的代价是高昂的。首先会在内存中分配一个新的长度的数组，然后将所有记录再全部哈希一遍，将原来的记录转移到新数组中。
5. 

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

1. 当 key 为整数时，table 就可以当成数组来用。而且这个数组是一个 索引从1开始 ，没有固定长度，可以根据需要自动增长的数组，使用 ipairs 对数组进行遍历。
2. 

## 11. 详细描述下项目使用的 class 机制
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