#### Lua基礎知識

##### 1. Lua的底层数据类型

- 常用的数据类型包括：

  - nil
  - boolean
  - number
  - string
  - function
  - userdata
  - thread
  - table

- ```C
  //Lua源碼中的數據類型宏定義
  //file: lua.h
  /*
  ** basic types
  */
  #define LUA_TNONE		(-1)
  
  #define LUA_TNIL		0
  #define LUA_TBOOLEAN		1
  #define LUA_TLIGHTUSERDATA	2
  #define LUA_TNUMBER		3
  #define LUA_TSTRING		4
  #define LUA_TTABLE		5
  #define LUA_TFUNCTION		6
  #define LUA_TUSERDATA		7
  #define LUA_TTHREAD		8
  
  #define LUA_NUMTYPES		9
  ```

- 当中没有被包括在常用类型的basic types为：

  - None
  - LightUserData
  - NumTypes

----

##### 2. Table存储形式与Resize

- 其他语言提供的所有结构---数组，记录，列表，队列，集合这些在lua中都用table来表示

- Lua的table是由数组部分（array part）和哈希部分（hash part）组成。

  - 数组部分索引的key是1~n的整数，当 key 为整数时，table 就可以当成数组来用。而且这个数组是一个索引从1开始，没有固定长度，可以根据需要自动增长的数组。
  - 哈希部分是一个哈希表（open address table），哈希表本质是一个数组，它利用哈希算法将键转化为数组下标，使Table可以当成字典来用。 它的 key 值可以是除了 nil 之外的任何类型的值。

- 向table中插入数据时，如果已经满了，Lua会重新设置数组部分或哈希表的大小，容量是成倍增加的，哈希部分还要对哈希表中的数据进行整理。

  - ```C
    //ltable.c
    void luaH_newKey(lua_State *L, Table *t, const TValue *key, TValue *value)
    {
        ...
        if (!isempty(gval(mp)) || isdummy(t)) {  /* main position is taken? */
        Node *othern;
        Node *f = getfreepos(t);  /* get a free place */
        if (f == NULL) {  /* cannot find a free place? */
          rehash(L, t, key);  /* grow table */
          /* whatever called 'newkey' takes care of TM cache */
          luaH_set(L, t, key, value);  /* insert key into grown table */
          return;
        }
        ...
    }
    ```

  - rehash

    - ```C
      //ltable.c
      /*
      ** nums[i] = number of keys 'k' where 2^(i - 1) < k <= 2^i
      */
      static void rehash (lua_State *L, Table *t, const TValue *ek) {
        unsigned int asize;  /* optimal size for array part */
        unsigned int na;  /* number of keys in the array part */
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
      ```

  - resize

    - ```c
      //ltable.c
      void luaH_resize (lua_State *L, Table *t, unsigned int newasize,
                                                unsigned int nhsize) {
        unsigned int i;
        Table newt;  /* to keep the new hash part */
        unsigned int oldasize = setlimittosize(t);
        TValue *newarray;
        /* create new hash part with appropriate size into 'newt' */
        setnodevector(L, &newt, nhsize);
        if (newasize < oldasize) {  /* will array shrink? */
          t->alimit = newasize;  /* pretend array has new size... */
          exchangehashpart(t, &newt);  /* and new hash */
          /* re-insert into the new hash the elements from vanishing slice */
          for (i = newasize; i < oldasize; i++) {
            if (!isempty(&t->array[i]))
              luaH_setint(L, t, i + 1, &t->array[i]);
          }
          t->alimit = oldasize;  /* restore current size... */
          exchangehashpart(t, &newt);  /* and hash (in case of errors) */
        }
        /* allocate new array */
        newarray = luaM_reallocvector(L, t->array, oldasize, newasize, TValue);
        if (l_unlikely(newarray == NULL && newasize > 0)) {  /* allocation failed? */
          freehash(L, &newt);  /* release new hash part */
          luaM_error(L);  /* raise error (with array unchanged) */
        }
        /* allocation ok; initialize new part of the array */
        exchangehashpart(t, &newt);  /* 't' has the new hash ('newt' has the old) */
        t->array = newarray;  /* set new array part */
        t->alimit = newasize;
        for (i = oldasize; i < newasize; i++)  /* clear new slice of the array */
           setempty(&t->array[i]);
        /* re-insert elements from old hash part into new parts */
        reinsert(L, &newt, t);  /* 'newt' now has the old hash */
        freehash(L, &newt);  /* free old hash part */
      }
      ```

- **resize**代价高昂，当我们把一个新键值赋给表时，若数组和哈希表已经满了，则会触发一个再哈希(rehash)。再哈希的代价是高昂的。首先会在内存中分配一个新的长度的数组，然后将所有记录再全部哈希一遍，将原来的记录转移到新数组中。新哈希表的长度是最接近于所有元素数目的2的乘方。

- ```luA
  local a = {}     --容量为0
  a[1] = true      --重设数组部分的size为1
  a[2] = true      --重设数组部分的size为2
  a[3] = true      --重设数组部分的size为4
  
  local b = {}     --容量为0
  b.x = true       --重设哈希部分的size为1
  b.y = true       --重设哈希部分的size为2
  b.z = true       --重设哈希部分的size为4
  ```

----

##### 3. Table 遍历

- 第一种：pairs迭代器

  - 遍历键值对，但过程中元素的键顺序可能是随机的（键可以是非数字的元素，因此会对键进行哈希处理，导致键顺序不同），但每个元素确保只会出现一次

  - ```lua
    --pairs
    t = {10, print, x = 12, k = "hi", 80, b = "cc", 999, collectgarbage("count")}
    for k, v in pairs(t) do
    	print(k, v)
    end
    ```

  - 但随机主要是在元素增删后发生（k <-> x）

    - 1	10
      2	function: 00D46B10
      k	hi
      x	12
    - 1	10
      2	function: 00B56DB0
      3	80
      4	999
      5	21.5947265625
      x	12
      k	hi
      b	cc

- 第二种：ipairs迭代器

  - 遍历列表（键必然是数字），确保元素是顺序出现的

  - ```lua
    --ipairs
    t = {10, print, 12, "hi"}
    for k, v in ipairs(t) do
    	print(k, v)
    end
    ```

  - 1	10
    2	function: 00C66A10
    3	12
    4	hi

  - ```lua
    --ipairs
    t = {10, print, 12, "hi", 8, 9, "DD"}
    for k, v in ipairs(t) do
    	print(k, v)
    end
    ```

  - 1	10
    2	function: 009E6770
    3	12
    4	hi
    5	8
    6	9
    7	DD

- 第三种：数值型for循环

  - 3.1：#<集合名>

    - 遍历n次（n = 集合长度）

  - ```lua
    --for #
    t = {10, print, 12, nil, "hi", a = 10}
    for k = 1, #t do
    	print(t[k])
    end
    ```

  - 10
    function: 005D6C50
    12
    nil
    hi

  - 3.2：table.maxn(集合名)

    - 遍历集合中键为整数的元素

    - ```lua
      t = {2, 5, 9, b = "DD", nil, "hi", a = 10} --table.maxn(t) = 5
      for k = 1, table.maxn(t) do
      	print(t[k])
      end
      ```

    - 2
      5
      9
      nil
      hi

----

##### 4. 项目使用的 class 机制

- 项目中的class机制由class.lua脚本实现。该脚本被项目中的绝大多数脚本所继承。该脚本是lua实现OOP的基础，基类中的基类，使其他继承它的lua脚本具有类、继承的概念。

- 该脚本大致分为以下几个部分：

  1. 在全局变量表中注册类表

     - ```lua
       --_G為全局變量表
       --把"__class"作為鍵注冊進去，值為一個table
       --類表
       if _G["__class"] == nil then
           _G["__class"] = {}
       end
       --如果全局變量表中已有類表，則取類表，否則建一個空表
       --保存所有的類容器
       --鍵：類容器（包含了繼承關係和構造方法）；值：類的所有成員
       local _class = _G["__class"] or {}
       ```

  2. 类函数（function class(super)）

     1. 初始化

        - ```lua
          local class_type={} --建一個空表作為類的容器
          --初始化類型的父類和構造方法
          class_type.ctor=false
          class_type.super=super
          ```

     2. new方法（function(self_class_type, ...)）定义

        1. 创建「容器」，设置元表和__index

           - ```lua
             local obj= {} --創建一個對象容器
             obj.__vt = _class[class_type] --這個容器包含了類的成員表
             --把成員表設成自身的元表
             local mt = {__index = obj.__vt} 
             setmetatable(obj, mt)
             ```

        2. 设置继承关係

           - ```lua
             --保存當前類的父類
             local typeSuper = self_class_type.super
             --只要當前父類不為空，則一直循環
             while typeSuper ~= nil do
                 local typeSuperL = typeSuper
                 --聲明父類對象容器
                 local objSuper = {}
                 --把對象容器中的父類設為該容器
                 obj[typeSuper] = objSuper
                 --為該容器設置元表，使得該容器被訪問時，會調用以下方法，保存被繼承的方法
                 setmetatable(objSuper, {__index=
                         function(t,k)
                             --得到父類的繼承方法
                             local ret=_class[typeSuperL][k]
                             if type(ret) == "function" then
                                 local func = ret
                                 ret = function(self1, ...)
                                     return func(obj, ...)
                                 end
                                 --保存下來
                                 t[k] = ret
                             else
                                 ret = nil
                             end
                             return ret
                         end
                     })
                 --移至上一層繼續遞歸
                 typeSuper = typeSuper.super
             end
             ```

        3. 调用父类及自身的构造函数

           - ```lua
             do
                 --從最上層的父類開始，調用每層的構造函數
                 local create
                 create = function(c,...)
                     if c.super then
                         create(c.super,...)
                     end
                     --最後調用自己的構造函數
                     if c.ctor then
                         c.ctor(obj,...)
                     end
                 end
                 create(class_type,...)
             end
             --把完成構造的對象返回
             return obj
             ```

     3. 成员容器（local vtbl）

        1. 判断类型的兼容性

           - ```lua
             __is_class = true,
             --判斷函數
             is_class_type = function(type)
                 --如果調用的函數的類型與成員表所屬的類型是否相同
                 if class_type == type then
                     return true
                 end
                 --判斷調用該函數的類型是否為成員表所屬類型的父類
                 local typeSuper = class_type.super
                 while typeSuper ~= nil do
                     if typeSuper == type then
                         return true
                     end
                     typeSuper = typeSuper.super
                 end
                 return false
             end,
             ```

        2. 声明成员

           - ```lua
             DeclareMembers = function(obj, members)
                 --得到原始表中的成員表
                 local vt = rawget(obj, "__vt")
                 --得到元表
                 local mt = getmetatable(obj)
                 --把__index和__newindex指向新的成員
                 mt.__index = members
                 mt.__newindex = members
                 --把原始表的成員表設為新成員的元表，並讓使用者在嘗試為不存在的成員賦值時，使用__newindex來創建值
                 setmetatable(members, {__index = vt, __newindex = function(t, key, value) rawset(obj, key, value) end})
             end,
             ```

        3. 声明成员变量

           - ```lua
             DeclareVar = function(obj, name, value)
                 if obj[name] ~= nil then 
                     error(string.format("成员变量:%s 已存在，声明失败", name))
                     return
                 end
                 --跳過元表判斷，直接設置原始表的成員
                 rawset(obj, name, value)
             end
             ```

     4. 后续处理

        - ```lua
          _class[class_type]= vtbl --把成員表設置在類表中對應的位置
          
          --為類型設置元表，使得類型在新增成員時，存到vtbl中
          setmetatable(class_type,{__newindex=
                  function(t,k,v)
                      vtbl[k]=v
                  end
              })
          
          --如果有父類
          if super then
              --把父類賦值給成員表
              vtbl.__super = super
              --為對象表設置元表，使得可以在無法從子類得到成員的情況下，就嘗試從父類入手
              setmetatable(vtbl,{__index=
                      function(t,k)
                          --取父類中的指定成員
                          local ret=_class[super][k]
                          --把該值賦予給當前類的成員表
                          vtbl[k]=ret
                          return ret
                      end
                  })
          end
          
          --返回類容器
          return class_type
          ```

  3. 功能函数

     1. is_class(class_type)

        - ```lua
          function is_class(class_type)
              return _class[class_type] ~= nil --判斷一個類是否存在於類表中
          end
          ```

     2. super(selfObj, superClass)

        - ```lua
          function super(selfObj, superClass)
              --嘗試得到父類
              superClass = superClass or selfObj.__super
              if superClass then --如果父類不為空
                  --返回父類對象
                  return selfObj[superClass]
              end
          end
          ```

     3. get_static(class_type)

        - ```lua
          function get_static(class_type)
              if class_type then --如果實參的類型不為空
                  return _class[class_type] --從類表中返回類對象
              end
          end
          ```

----

##### 5. Lua的GC机制

- Lua的GC使用的基本算法是标记和清扫算法

  - 该算法一共有4个阶段：
    - 标记阶段：把根集标记为活跃，该集合由Lua可直接访问的对象组成。一个活跃对象可以到达的对象也是活跃的。当所有活跃对象被标记后，该阶段结束。
    - 整理阶段：这个阶段处理析构器和弱引用表。首先，Lua遍历所有被标记为需要析构，但又没有活跃标记的对象，这些对象会暂时被标记为活跃（复苏）并放到一个单独的列表裡（析构阶段使用）；然后Lua遍历弱引用表，从中移除键或值未被标记的元素
      - 析构器：元表中含__gc的对象
      - 弱引用表：元素中含__mode = "k"或"v"或两者都有的对象
    - 清扫阶段：遍历所有对象，如果一个对象没有被标记为活跃，则将其回收；否则，Lua清理标记，准备下次清理周期
    - 析构阶段：调用整理阶段单独分离出的对象的析构器

- 目前Lua可以选择使用两种GC模式的其中一种

  - 增量模式（incremental mode）（默认）
  - 分代模式（generational mode）

- 增量模式（分步模式）

  - 增量模式实际上是把Lua5.1版本前一次性完成，且会在GC期间停止主程序运行的回收过程，改成与解释器一起交替运行。因此增量GC模式的不会在回收过程中停止主程序的运行。

    - 每当解释器分配了一定数量的内存时，GC也执行一小步。

      - GC的步进倍率和间歇率可以分别通过setpause和setstepmul设置。前者为一次收集完成后等待多久再开始新一轮的收集；后者为每分配1KB内存，GC应该进行多少工作（值越高，GC越不积极）

    - 这个模式会使GC在工作期间，可能会改变一个对象的可达性。因此，Lua使用了一些手段确保了GC的正确性。

      - 三色标记算法 => 每个对象都可能有以下三个状态的其中一个：

        - 白色：所有没有被标记过（无法到达）都为白色，需要被清除

          - 所有对象创建的时候/GC周期开始之前的所有对象都会被设为白色

  - ```C
    /*
    ** create a new collectable object (with given type and size) and link
    ** it to 'allgc' list.
    */
    GCObject *luaC_newobj (lua_State *L, int tt, size_t sz) {
        global_State *g = G(L);
        GCObject *o = cast(GCObject *, luaM_newobject(L, novariant(tt), sz));
        o->marked = luaC_white(g);
        o->tt = tt;
        o->next = g->allgc;
        g->allgc = o;
        return o;
    }
    ```

  - 灰色：对象本身虽然被标记了，但它引用的对象没有被标记。表示了对象标记的一个中间状态

  - 黑色：对象自身以及它引用的对象都已被标记（可以到达）

  - ```C
    #define WHITE0BIT	3  /* object is white (type 0) */
    #define WHITE1BIT	4  /* object is white (type 1) */
    #define BLACKBIT	5  /* object is black */
    #define FINALIZEDBIT	6  /* object has been marked for finalization */
    #define TESTBIT		7
    
    #define WHITEBITS	bit2mask(WHITE0BIT, WHITE1BIT)
    
    #define iswhite(x)      testbits((x)->marked, WHITEBITS)
    #define isblack(x)      testbit((x)->marked, BLACKBIT)
    #define isgray(x)  /* neither white nor black */  \
        (!testbits((x)->marked, WHITEBITS | bitmask(BLACKBIT)))
    ```

  - 对所有对象进行遍历时，会根据可到达的情况给对象「上色」（黑色和灰色）

    - ```C
      static void reallymarkobject (global_State *g, GCObject *o) {
          switch (o->tt) {
              case LUA_VSHRSTR:
              case LUA_VLNGSTR: {
                  set2black(o);  /* nothing to visit */
                  break;
              }
              case LUA_VUPVAL: {
                  UpVal *uv = gco2upv(o);
                  if (upisopen(uv))
                      set2gray(uv);  /* open upvalues are kept gray */
                  else
                      set2black(uv);  /* closed upvalues are visited here */
                  markvalue(g, uv->v);  /* mark its content */
                  break;
              }
              case LUA_VUSERDATA: {
                  Udata *u = gco2u(o);
                  if (u->nuvalue == 0) {  /* no user values? */
                      markobjectN(g, u->metatable);  /* mark its metatable */
                      set2black(u);  /* nothing else to mark */
                      break;
                  }
                  /* else... */
              }  /* FALLTHROUGH */
              case LUA_VLCL: case LUA_VCCL: case LUA_VTABLE:
              case LUA_VTHREAD: case LUA_VPROTO: {
                  linkobjgclist(o, g->gray);  /* to be visited later */
                  break;
              }
              default: lua_assert(0); break;
          }
      }
      ```

  - 总的来说，白色对象集就是会被回收的部分；黑色就是需要保留的部分；灰色就是GC目前黑白色的边界。

    - 在理想的情况下，黑色的对象在任何时间点都不可能指向白色的对象；所有被根集引用的对象要麽是黑色的、要麽是灰色的。
    - 随著GC回收工作一步一步的进行，当所有灰色对象都转成黑色对象后，GC回收工作结束。

  - 不过，由于增量式GC模式的工作是与解释器交叉推进的，使得有的对象的可达性会受影响，因此，有可能会出现「黑色指向白色」的情况

    - 当出然这种情况时，就会在这种赋值的地方插入一个write barrier，恢复不变条件（黑色不会指向白色）

    - 要麽是把白色的对象变成灰色（forward）

      - ```C
        void luaC_barrier_ (lua_State *L, GCObject *o, GCObject *v) {
            global_State *g = G(L);
            lua_assert(isblack(o) && iswhite(v) && !isdead(g, v) && !isdead(g, o));
            if (keepinvariant(g)) {  /* must keep invariant? */
                reallymarkobject(g, v);  /* restore invariant */
                if (isold(o)) {
                    lua_assert(!isold(v));  /* white object could not be old */
                    setage(v, G_OLD0);  /* restore generational invariant */
                }
            }
            else {  /* sweep phase */
                lua_assert(issweepphase(g));
                if (g->gckind == KGC_INC)  /* incremental mode? */
                    makewhite(g, o);  /* mark 'o' as white to avoid other barriers */
            }
        }
        ```

    - 要麽是把黑色的对象变成灰色（backward）

      - ```C
        void luaC_barrierback_ (lua_State *L, GCObject *o) {
            global_State *g = G(L);
            lua_assert(isblack(o) && !isdead(g, o));
            lua_assert((g->gckind == KGC_GEN) == (isold(o) && getage(o) != G_TOUCHED1));
            if (getage(o) == G_TOUCHED2)  /* already in gray list? */
                set2gray(o);  /* make it gray to become touched1 */
            else  /* link it in 'grayagain' and paint it gray */
                linkobjgclist(o, g->grayagain);
            if (isold(o))  /* generational mode? */
                setage(o, G_TOUCHED1);  /* touched in current cycle */
        }
        ```

    - 重新变回灰色的对象会被放到单独的一个列表中，等待原子阶段（atomic phase）时遍历

  - 标记阶段会被原子步骤（atomic step）所终结，在该步骤中：

    - 所有「被重置为灰色」的对象都会被遍历
    - 弱引用表会被清理
    - 分离出需要调用的析构器

  - 增量式收集器

  - GC 的步进只和虚拟机分配新的内存有关，只要虚拟机不分配更多内存，GC 是不会自动运行的。

    - 步进式 GC 需要用setpause和setstepmul来控制 GC 的工作时间：

  - 总而言之，步进式 GC 能够减少每次 GC 工作时的停顿时间，但是无法减少 GC 带来的额外开销，相反，GC 的时间成本（额外的 Barrier ）和空间成本（未能及时回收不再使用的内存）反而较之全量 GC 增加了。

- 分代模式

  - 分代模式假设大部分的对象都会很快被销毁，所以，收集器可以集中工作量在这些「年轻」的对象上

  - 所有的对象被分类为「年轻」和「年老」两代。当它们被创建出来时是「年轻」的；经过两次次级收集（minor collection）后仍然没被回收的就成长为「年老」的

    - 次级收集：只会遍历且清理「年轻」对象

  - 分代 GC 的不变项：

    - 「老对象」不会指向「新对象」

    - 这使得分步模式的不变项难以被保持

      - 无论是 forward 还是 backward 都有问题：
        - 对于 forward ，也就是把新对象变成老的，无疑会制造大量老对象，还需要递归变量，否则就会打破规则。
        - 如果是采用 backward 策略，更很难保持条件成立（对象很难知道谁引用了自己，就无法准确的把老对象变回新的）。

    - 所以，需要引入第三态：「被接触过的」

      - 如果back barrier检测到一个「老对象」，该「老对象」会被标记为「被接触过的」，并将它放到一个特殊的列表裡
      - 这些「被接触过」的对象在次级收集中也会被遍历，但是不会被收集。
      - 经过两次次级收集周期后，「被接触」的对象返回到「老对象」状态，直到它再次「被接触」
        - 在Lua 5.2 的分代 GC 算法中，只针对单个次级收集周期处理。任何对象活过当前的收集周期，就会变老。但这样处理，如果一个对象刚刚被创建出来，次级收集过程就开启了，它很容易就活过这个周期，对象就迅速变老了。

    - ```C
      /* object age in generational mode */
      #define G_NEW		0	/* created in current cycle */
      #define G_SURVIVAL	1	/* created in previous cycle */
      #define G_OLD0		2	/* marked old by frw. barrier in this cycle */
      #define G_OLD1		3	/* first full cycle as old */
      #define G_OLD		4	/* really old object (not to be visited) */
      #define G_TOUCHED1	5	/* old object touched this cycle */
      #define G_TOUCHED2	6	/* old object touched in previous cycle */
      
      #define AGEBITS		7  /* all age bits (111) */
      ```


----

##### 6. Lua的全局变量跟local变量

- Lua的全局变量跟local变量的区别

  - 实现：局部变量是在栈上的register上通过整数索引得到的；全局变量则是在table或者userdata中,通过string常量或者string变量作为键来索引全局变量。

  - 性能：通过实现不同,局部变量稍微比全局变量速度快。

  - Lua是如何查询一个全局变量

    - Lua将所有的全局变量保存在一个常规的table中,这个table称之为环境(_G),使 用下面的代码可以打印当前环境中所有全局变量的名称

    - ```C
      //lauxlib.h
      /* global table */
      #define LUA_GNAME	"_G"
      
      //lapi.c
      /*
      ** Get the global table in the registry. Since all predefined
      ** indices in the registry were inserted right when the registry
      ** was created and never removed, they must always be in the array
      ** part of the registry.
      */
      #define getGtable(L)  \
      	(&hvalue(&G(L)->l_registry)->array[LUA_RIDX_GLOBALS - 1])
      
      LUA_API int lua_getglobal (lua_State *L, const char *name) {
        const TValue *G;
        lua_lock(L);
        G = getGtable(L);
        return auxgetstr(L, G, name);
      }
      
      LUA_API void lua_setglobal (lua_State *L, const char *name) {
        const TValue *G;
        lua_lock(L);  /* unlock done in 'auxsetstr' */
        G = getGtable(L);
        auxsetstr(L, G, name);
      }
      
      /*
      ** get functions (Lua -> stack)
      */
      static int auxgetstr (lua_State *L, const TValue *t, const char *k) {
        const TValue *slot;
        TString *str = luaS_new(L, k);
        if (luaV_fastget(L, t, str, slot, luaH_getstr)) {
          setobj2s(L, L->top, slot);
          api_incr_top(L);
        }
        else {
          setsvalue2s(L, L->top, str);
          api_incr_top(L);
          luaV_finishget(L, t, s2v(L->top - 1), L->top - 1, slot);
        }
        lua_unlock(L);
        return ttype(s2v(L->top - 1));
      }
      ```

    - ```lua
      a = 3
      b = "JJ"
      d = print
      c = a
      djkfd = "(JIFE"
      fefe = 4343
      
      
      for n in pairs(_G) do
      	print(n)
      end
      --[[ output
      a
      string
      xpcall
      b
      package
      tostring
      print
      os
      ...
      --]]
      ```

  - 每一个函数中,局部变量的数量最大为200个。

    - ```C
      //lparser.c
      /* maximum number of local variables per function (must be smaller
         than 250, due to the bytecode format) */
      #define MAXVARS		200
      ```

    - Lua在运行任何代码之前,Lua预编译源代码成为内部字节码格式。字节码格式类似于真正CPU的机器指令。内部字节码格式通过C语言解释。實際上就是一個龐大的while-loop，包含了無數的switch語句，一個指令一個case。自从5.0版本,Lua使用一种register-based virtual machine 虚拟机。 “registers” 不依赖于CPU真正的指令。相反,lua使用stack 来解决registers。每一个函数最大有250个registers,。

    - 依赖于大量的registers，Lua預編譯器可以保存所有的局部變量在register中。所以，在Lua中访问局部变量非常快速。

  - 作用域：局部变量的作用域在语法块中。在运行时,全局变量可以在任何环境table赋值的函数中获取。

    - local的变量的作用域规则
      - 如果定义在一个函数体中, 那么作用域就在函数中.
      - 如果定义在一个控制结构中, 那么就在这个控制结构中.
      - 如果定义在一个文件中, 那么作用域就在这个文件中.

  - 声明：局部变量必须通过明显的声明去定义和使用，但是,全局变量可以在代码执行时,动态的访问,甚至可以在运行时设置全局变量。局部变量的列表的定義是静态的；而全局变量列表可以在运行时才决定，甚至是改變。

  - 标准库的访问：所有Lua的标准库都是通过全局变量暴露给使用者。

  - 优先权：局部变量覆盖全局变量

----

##### 7. 元表

- 元表主要是通过一系列的元方法，来为其所服务的table提供一些「面对非预定义行为的指定解决方法」，比如提供类似构造方法的调用的元方法__call、让两个表之间进行运算、提供类似继承的关係。总的来说，是让表具有一些类的行为。

  - __call：调用表的时候同时调用元表（如有）中的该元方法，可传参。

    - ```lua
      local t = { }
      local mt = { }
      --定義元方法，默認第一個參數為self
      function mt:__call(arg)
      	print("with arg:", arg)
      end
      setmetatable(t, mt)
      t("114514")
      ```

  - __add：类似于重载加法运算符

    - ```lua
      local X = { c = "x" }
      local Y = { c = "y" }
      local m = {
      	--運算符重載
      	__add = function(t1, t2)
      		return { c = t1.c .. t2.c }	end
      }
      setmetatable(X, m)
      setmetatable(Y, m)
      print((X + Y).c) --xy
      print((Y + X + Y).c) --yxy
      ```

  - __index：当尝试访问表中的某个键时，发现表裡没有，就会向元表寻找index元方法中的定义（可以是表/键值对/方向），为表带来了类似于继承的功能

    - ```lua
      local A = {}
      local metaA = { }
      function metaA:__index(arg)
          return arg * arg
      end
      function metaA:__call(arg1, arg2)
          print(A[arg1 + arg2])
      end
      setmetatable(A, metaA)
      --[[
      使用A.a來查找元素時，如果表中找不到，就去該表的元表中__index方法中
      如果__index裡的是一個表，則這些表裡面重複上一步；如果是函數，就返回函數的返回值
      --]]
      A(123, 111) --54756
      ```

----

##### 8. Lua会产生的垃圾回收  情况

- ```C
  /*
  ** Union of all Lua values
  */
  typedef union Value {
      struct GCObject *gc;    /* collectable objects */
      void *p;         /* light userdata */
      lua_CFunction f; /* light C functions */
      lua_Integer i;   /* integer numbers */
      lua_Number n;    /* float numbers */
  } Value;
  ```

- lua每次分配会被GC回收的数据类型（GCObject）时，会主动检查是否满足GC条件

  - 当GCdept大于0时，就会触发自动GC

  - ```C
    /*
    ** Does one step of collection when debt becomes positive. 'pre'/'pos'
    ** allows some adjustments to be done only when needed. macro
    ** 'condchangemem' is used only for heavy tests (forcing a full
    ** GC cycle on every opportunity)
    */
    #define luaC_condGC(L,pre,pos) \
    	{ if (G(L)->GCdebt > 0) { pre; luaC_step(L); pos;}; \
    	  condchangemem(L,pre,pos); }
    
    /* more often than not, 'pre'/'pos' are empty */
    #define luaC_checkGC(L)		luaC_condGC(L,(void)0,(void)0)
    ```

----

##### 9. Lua的尾调用

- 当一个函数（Func_A）的最后一个动作是调用另一个函数（Func_N），就会发生尾调用。被调用的函数完成工作后，不会返回调用Fun_N的位置，而是直接返回到调用Func_A的位置

  - ```lua
    function func_A(x)
        n = x * x
        print(n)
        func_N(n)
    end
     function func_N(n)
        if(n > 0) then return func_N(n - 1) end
     end
    func_A(3)
    ```

- 在调用func_N(n)后，func_A(x)就没有任何后续行为，因此也没有必要保留后者的栈信息，减少栈溢出的机会。