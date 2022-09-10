# Program Basic QA

## ~~Q1(Finished)~~

1. C#函数中 new 一个struct 对象 会不会产生垃圾回收？
- 不会，struct对象为值类型。即使使用new关键字把struct new出来，编译器仍然会把它识别为值类型，因此会生成对应的IL代码，在线程栈上分配一个实例。
  
     - ```C#
       //C#
       class Program
       {
           static void Main(string[] args)
           {
               Person p = new Person();
               Grade g = new Grade();    
           }
       }
       struct Person { }
       class Grade { }
       ```
     
     - ```IL
       //IL
       .method private hidebysig static void  Main(string[] args) cil managed
       {
         .entrypoint
         // 程式碼大小       22 (0x16)
         .maxstack  1
         .locals init (valuetype ProgramBasicQA_Test.Person V_0,
                  class ProgramBasicQA_Test.Grade V_1)
         IL_0000:  nop
         IL_0001:  ldloca.s   V_0
         IL_0003:  initobj    ProgramBasicQA_Test.Person
         IL_0009:  newobj     instance void ProgramBasicQA_Test.Grade::.ctor()
         IL_000e:  stloc.1
         IL_000f:  call       string [System.Console]System.Console::ReadLine()
         IL_0014:  pop
         IL_0015:  ret
       } // end of method Program::Main
       ```
     
     - 从上述IL可以看出，new Person实际编译的IL指令为「initobj」，该指令只将其下的每个字段初始化为0。分配在线程栈上的对象就会在它的定义域结束后被弹栈销毁。
     
     - 而实例化一个class的new，使用的IL指令是一个「newobj」，该指令会创建一个对象并向它分现托管堆上的内存，并把堆的地址引用推到线程栈上。托管堆会在发现没有足够地址空间分配对象时进行垃圾回收。
     
     - 因此，被分配到栈上的值类型Person则不会产生垃圾回收

## ~~Q2(Finished)~~

1. C#在函数中 new 一个数组 会不会产生垃圾回收？

   - ```IL
     //C#
     static void Main(string[] args)
     {
         Person[] p = new Person[] { };
         Grade[] g = new Grade[] { };    
         Console.ReadLine();
     }
     
     //IL
     .method private hidebysig static void  Main(string[] args) cil managed
     {
         .entrypoint
         // 程式碼大小       22 (0x16)
         .maxstack  1
         .locals init (valuetype ProgramBasicQA_Test.Person[] V_0,
         class ProgramBasicQA_Test.Grade[] V_1)
         IL_0000:  nop
         IL_0001:  ldc.i4.0
         IL_0002:  newarr     ProgramBasicQA_Test.Person
         IL_0007:  stloc.0
         IL_0008:  ldc.i4.0
         IL_0009:  newarr     ProgramBasicQA_Test.Grade
         IL_000e:  stloc.1
         IL_000f:  call       string [System.Console]System.Console::ReadLine()
         IL_0014:  pop
         IL_0015:  ret
     } // end of method Program::Main
     ```

   - new一个数组时的IL指令为newarr，该指令将数组的对象引用地址推到线程栈，因此数组也是一个引用类型。其内存分配在托管堆上，会产生垃圾回收。

## ~~Q3(Finished)~~

1. C#在函数中 new 一个struct对象数组 会不会产生垃圾回收？

   - 参考上题IL代码，可见即使new一个Person（值类型）的数组，其所编译出来的IL指令也是newarr，代表struct对象数组也是一个引用类型，其内存分配在托管堆上，也会产生垃圾回收。

   ----

## ~~Q4(Finished)~~

1. unity的部分对象如：Mesh，AnimationClip 等对象 new 出来之后需不需要主动释放？

   - 需要通过Destroy删除这些new出来的对象，从而释放它们的内存

   - 以下列代码为例：

     - ```C#
       AnimationClip clip;
       Mesh m;
       
       // Start is called before the first frame update
       void Start()
       {
           clip = new AnimationClip();
           m = new Mesh();
       
           //Can't Release        
           Invoke("TryToEmpty", 10);
           Invoke("TryToGC", 15);
           Invoke("TryToDestroy", 20);
           Invoke("TryToGC", 25);
       }
       
       void TryToEmpty()
       {
           Debug.Log("Try To Empty");
           clip = null;
           m = null;
       }
       
       void TryToDestroy()
       {
           Debug.Log("Try To Destroy");
           Destroy(clip);
           Destroy(m);        
       }
       
       void TryToGC()
       {
           Debug.Log("Try to GC");
           GC.Collect();
       }
       ```

     - 在本例中，我首先在Start裡分别new了一个Mesh对象和AnimationClip对象，并分别延时5秒和15秒后调用TryToGC和TryToDestroy方法

   - 以下是Profiler中Memory显示的状态变化

     - 首先是new了一个Mesh对象和AnimationClip

       - ![GC_0](\Program Basic QA\ScreenShot\GC_0.png)

     - 然后在10秒时尝试置空引用、15秒后强迫GC回收一次

       - ![GC_1](\Program Basic QA\ScreenShot\GC_1.png)
       - 引用空了，但内存仍然被佔用

     - 然后在20秒时删除对象且在25秒再强迫GC回收一次

       - ![GC_2](\Program Basic QA\ScreenShot\GC_2.png)
       - 并没有甚麽用

     - 另一种是直接5秒后Destroy

       - ```C#
         Invoke("TryToDestroy", 5);
         ```

       - 一开始仍然是把对象new出来

         - ![GC_0](\Program Basic QA\ScreenShot\GC_0.png)

       - 期间不进行任何动作，5秒后调用Destroy直接摧毁对象，则能有效释放内存占用

         - ![GC_3](\Program Basic QA\ScreenShot\GC_3.png)

## ~~Q5(Finished)~~

1. 想办法评估下 Lua和C# 在循环里持续产生内存垃圾对性能的影响

   - 使用profiler查看循环产生内存垃圾对性能的影响（CPU Usage）

   - Lua方面，使用toLua进行测试

   - 测试规则：

     - 启动测试后，脚本会在每帧生成<currSpawnCntPerFrame>数量（初始为100个）的SpawnObj，该生成量每秒增加<spawnCntIncreaseVal>（默认为100个）个；测试共120次增量——时间大约为2分钟，最后一帧的生成量为12000
   
     - C#
   
       - ```C#
         public class UnitySpawnTest : MonoBehaviour
         {
             bool isStop = true;
             [SerializeField] int currSpawnCntPerFrame = 0;
             [SerializeField] int spawnCntIncreaseVal = 1; //每幀new的數量   
             [SerializeField] int stepUpConfig = 60; //多少次循環增加一次壓力
             [SerializeField] int stepCounter = 0; //循環計數器    
             [SerializeField] int limitStep = 120; //2分鐘後觀察
             [SerializeField] int limitCounter = 0;
             private void Start()
             {
                 currSpawnCntPerFrame = spawnCntIncreaseVal;
                 stepCounter = 0;
                 StartCoroutine("CreatingGO");
             }
         
             private void Update()
             {
                 if (Input.GetKeyDown(KeyCode.S))
                 {
                     isStop = !isStop;
                     if (isStop)
                     {
                         stepCounter = 0;
                         currSpawnCntPerFrame = spawnCntIncreaseVal;
                     }
                 }
             }
         
             IEnumerator CreatingGO()
             {
                 while(true)
                 {
                     Debug.Log("IEnumerator Activated !");
                     if (!isStop)
                     {
                         if(stepCounter >= stepUpConfig)
                         {
                             Debug.Log("Step Up !");
         
                             if(limitCounter >= limitStep)
                             {
                                 Debug.LogError("Test Finished.............");
                                 isStop = true;
                                 yield break;
                             }
                            
                             currSpawnCntPerFrame += spawnCntIncreaseVal;
                             stepCounter = 0;
                         }
         
                         Debug.Log("Spawning...");
                         for (int i = 0; i < currSpawnCntPerFrame; i++) { new SpawnObj("Spawned: " + i); }
                         stepCounter++;              
                         yield return new WaitForEndOfFrame();
                     }
                     yield return new WaitForEndOfFrame();
                 }
             }
         }
         
         public class SpawnObj
         {
             readonly string m_Name;
             public SpawnObj(string name) { m_Name = name; }
         }
         ```
   
       - ![CSharpGC_test](C:\Users\Kelvin\Desktop\ProgramBasicQA\Program Basic QA\ScreenShot\CSharpGC_test.png)
   
     - Lua
   
       - ```C#
         public class LuaSpawnTest : MonoBehaviour
         {
             LuaState lua = null;    
             [SerializeField] string luaFile = "Lua_Spawner";
             LuaFunction func;	
             //...
             
             // Use this for initialization
             void Start()
             {        
                 lua = new LuaState();
                 lua.Start();
                 
                 string sceneFile = Application.dataPath + "/Lua";
                 lua.AddSearchPath(sceneFile);        
                 lua.Require(luaFile);
                 func = lua.GetFunction("Spawner");
                 //...
             }
         
             private void Update() { /*...*/ }
             IEnumerator CreatingGO()
             {
                 while (true)
                 {
                     //...
                         for (int i = 0; i < currSpawnCntPerFrame; i++) { func.Call(); }
                     //...
                 }
             }
         }
         ```
   
       - ```lua
      --Lua_SpawnModule
         Lua_SpawnObj = { name = "" }
         local mt =
         {
         __index = Lua_SpawnObj,
             __call = function(self, initName) self.name = initName end
         }
         function Lua_SpawnObj:new(initName)
             self = self or {}
             setmetatable(self, mt)
             self(initName)
             return self
         end
         ```
     
       - ```lua
         --Lua_Spawner
         require "Lua_SpawnModule"
         function Spawner() Lua_SpawnObj:new("Tester") end
         ```
     
       - ![LuaGC_test](C:\Users\Kelvin\Desktop\ProgramBasicQA\Program Basic QA\ScreenShot\LuaGC_test.png)`
     
     - 就垃圾回收的频率而言，Lua的性能较弱，从中段开始，垃圾回收频率约为30帧一次；而C#进行的循环生成造成的垃圾回收频率较优，两次垃圾回收之间隔了较长的时间。
     
     - 就FPS而言，C#性能能维持较高的FPS
     
     - 较GC Allocation per Frame而言，Lua的每帧GC分配约为C#的两倍

## ~~Q5.5(Finished)~~

1. 加载unity资源：AnimatorController，材质， 贴图，动作文件 需不需要释放，需要的释放使用什么接口？

   - Unity加载资源的方法一般有两种：
     
- Resources.Load
  
  - 一般对Resource.Load的使用方法都是加载指定的资源，然后将其实例化到场景中。这一连串的动作使得该资源一方面会被加载到Asset中，另一方面还会产生一个Clone到Scene Memory中。
  
- ![GC_4](\Program Basic QA\ScreenShot\GC_4.png)
  
- 如果只是把生成出来的物体给删除掉，那只是会释放掉在Scene Memory中的内存，而Asset裡面的内存则会一直被占用。
  
       - ```C#
         void Start() { anim = Instantiate(Resources.Load<RuntimeAnimatorController>("testAnimCon")); }
         void Update(){ if (Input.GetKeyDown(KeyCode.U)) { Destroy(anim); } }
    ```
     
  - 虽然也可以使用DestroyImmediate(object, true)来告诉Unity可以摧毁Asset，但是这只会在「只有在Asset，没有在SceneMemory占用」的情况下生效，如果同时存在两者的占用，那DestroyImmediate只能释放在SceneMemory中的内存，且即使释放了Scene Memory的内存后，再次使用DestroyImmediate也不能释放在Asset裡面的内存占用
     
         - ```C#
           if (Input.GetKeyDown(KeyCode.D))
           {
               Debug.Log("Destroy Asset if no Clone existed");
               DestroyImmediate(mat, true);
               DestroyImmediate(anim, true);
           }
   ```
  
- 除了Destroy相关的方法外，Resource本身还提供了一些静态方法来释放内存
  
  - Resources.UnloadAsset(obj)
    
    - 该方法无法释放SceneMemory中的内存（场景中的Clone），只能释放Asset裡面的内存，且只能释放一些individual assets（mesh/texture/material/shader），不能释放一些组件/AssetBundle的资源（比如Animator）
    
       - ```C#
         Resources.UnloadAsset(mat);            
         Resources.UnloadAsset(anim);       
    ```
    
  - ![GC_5](\Program Basic QA\ScreenShot\GC_5.png)
    
  - Resources.UnloadUnusedAssets();
    
    - 该方法则相对UnloadAsset而言更为可靠且杀伤力大，可以释放所有Resource加载的所有Asset内存；如果在调用该方法前把相关的引用置空，则可以把Asset和SceneMemory裡的内存一并释放。适合作为最后把关的清道夫。
    
         - ```C#
           mat = null;
           anim = null;
           Resources.UnloadUnusedAssets();
      ```
  
- 总的来说，通过Resources加载的资源是需要释放的，即使把场景中（Scene Memory）的资源释放了，还有主动调用DestroyImmediate(obj, true)/UnloadAsset(obj)/UnloadUnusedAssets()来释放在Asset裡的内存占用
  
- 较好的释放策略是，对于一些不会重複使用的Individual asset可以加载完之后马上调用UnloadAsset；在场景关闭前，调用DestroyImmediate(obj, true)清除所有的SceneMemory裡的占用，最后再使用UnloadUnusedAssets确认释放所有内存。
  
- AssetBundle.LoadFrom(File/Stream/Memory)/UnityWebRequest
  
  - 如以下代码，使用LoadFromFile加载了指定本地指定路径上的一个AB包
    
         - ```C#
           AssetBundle ab;
           void Start()
           {
               string path = "AssetBundles/scene/model.ab";
               ab = AssetBundle.LoadFromFile(path);
               //AssetBundle.LoadFromStream            
               //AssetBundle.LoadFromMemory
               GameObject[] obj = ab.LoadAllAssets<GameObject>();
               foreach (var item in obj)
               {
                   GameObject go = Instantiate(item);
               }
           }
     
  - 卸载AB包加载的资源内存占用的方式有两种：
    
    - (instance)ab.Unload(bool unloadAllLoadedObject)
    
    - (static)AssetBundle.UnloadAllAssetBundles(bool unloadAllLoadedObject)
    
    - 两种方法其实都是没有太大区别，对象方法只释放某个AB自身的内存占有，静态方法则是释放所有AB的占有内存
    
    - AssetBundle的Unload方法含有一个bool参数，该参数告诉AB要不要把Load的Asset
    
           - false => 只释放掉在AB包本身的资源
             
           - ![GC_6](\Program Basic QA\ScreenShot\GC_6.png)
    
       - true => 同时释放AB包加载出来的Asset资源，尽管该资源正在被引用（引用资源的地方将会失去对它的引用）
        - ![GC_7](\Program Basic QA\ScreenShot\GC_7.png)
    
    - 如果要把场上的物体也释放掉，则需要和Destroy系（和Resources系）的方法协同
    
           - 比如：
             
             - ```C#
               AssetBundle ab;
               public GameObject[] asset_Go;
               public GameObject[] clone_Go = new GameObject[2];
               
               void Start()
               {
                   string path = "AssetBundles/scene/model.ab";
                   //加載AB包
                   ab = AssetBundle.LoadFromFile(path);
                   //加載AB包中的Asset
                   asset_Go = ab.LoadAllAssets<GameObject>();
                   //把Asset的Clone生成到場景
                   for (int i = 0; i < asset_Go.Length; i++) { clone_Go[i] = Instantiate(asset_Go[i]); }
               }
               
               private void Update
               {
                   //釋放AB包內存，但不釋放其加載的Asset
                   if (Input.GetKeyDown(KeyCode.R)) { AssetBundle.UnloadAllAssetBundles(false); }
                   //釋放Clone在SceneMemory中的內存
                   if (Input.GetKeyDown(KeyCode.O)) { foreach (var item in clone_Go) { Destroy(item); } }
                   //釋放Asset內存
                   if (Input.GetKeyDown(KeyCode.U))
                   {
                       asset_Go = null;
                       Resources.UnloadUnusedAssets();
                   }
               }
               ```

## ~~Q6(Finished)~~

1. MonoBehaviour 数量过多 对性能影响如何，或者一个MonoBehaviour 自带的消耗有哪些？

   - 每一个Monobehaviour都是通过反射来调用生命周期函数的（Awake, Update, LateUpdate...）。Monobehaviour会在游戏开始时首先得到所有写有生命周期函数的脚本，保存下来后再调用这些生命周期方法。

   - 因此，只要在Monobehaviour中声明了Update这样的生命周期函数，无论函数体裡是否有东西，该方法都会被执行，从对CPU造成一定的负荷
   
     - 场景上带有Monobehaviour且带有各种生命周期（尤其是Update类）的脚本越多，CPU负荷越大，即使所有生命周期的方法体裡都没有内容
   
     - 以下是测试脚本
     
     - ```c#
       using System.Collections;
       using System.Collections.Generic;
       using UnityEngine;
       
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
     
     - 场景上只有一个挂有该脚本的空物体时的CPU Usage
     
       - ![CPU_1](\Program Basic QA\ScreenShot\CPU_1.png)
     
     - 场景上有6000+个挂有该脚本的空物体时的CPU Usage
     
       - ![CPU_0](\Program Basic QA\ScreenShot\CPU_0.png)
   
   ----

## ~~Q7(Finished)~~

1. Lua的底层数据类型有哪些？

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

## ~~Q8(Finished)~~

1. 为什么说Lua一切皆Table,Table有哪两种存储形式，Table是如何Resize的，了解Resize的代价

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


## ~~Q9(Finished)~~

1. table 遍历有几种形式 有什么不同

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

## ~~Q10(Finished)~~

1. 详细描述下项目使用的 class 机制

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

## ~~Q11(Finished)~~

1. 大体了解下Lua的GC机制

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

   - https://www.lua.org/wshop18/Ierusalimschy.pdf

## ~~Q12(Finished)~~

1. Lua的全局变量跟local变量的区别，Lua是如何查询一个全局变量的，local的变量的作用域规则是怎么样的？

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

## ~~Q13(Finished)~~

1. 详细说明元表的机制和它的一些特性

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

## ~~Q14(Finished)~~

1. lua 那些情况会产生垃圾回收 

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

## ~~Q15(Finished)~~

1. C#不想让外部使用new的方式 生成对象，有什么办法？

   - 把构造函数设为private，然后为类提供一个静态得到其对象的方法

     - 单例模式

   - ```C#
     class Program
     {
         static void Main(string[] args)
         {
             Singleton s1 = new Singleton(); //錯誤	CS0122	'Singleton.Singleton()' 由於其保護層級之故，所以無法存取。
             Singleton s = Singleton.instance; //編譯通過
         }
     }
     public class Singleton
     {
         private Singleton() { }
         public static Singleton instance 
         {
             get
             {
                 if(instance == null)
                 {
                     instance = new Singleton();
                 }
                 return instance;
             }
             private set
             {
                 instance = value;
             }
         }
     }
     ```

## ~~Q16(Finished)~~

1. 非运行时的代码比如OnDrawGizmos 里面 new出来的资源，删除时机的隐患是什么（注意不加编辑器运行的属性，OnDisable，OnDestroy是不会执行的）

   - OnDrawGizmos实际上也是每帧调用的，在下列代码运行且点撃"G"后，场景会一直生成新的GameObject。

     - 如果没有开关在内部控制OnDrawGizmos裡new的元素，这些元素会在编辑模式下开始生成

     - ```C#
       isDrawGizmos = false;
       private void Update()
       {
           if (Input.GetKeyDown(KeyCode.G))
           {
               isDrawGizmos = !isDrawGizmos;
               Debug.Log($"Gizmos On State: {isDrawGizmos}");
           }
       }
       private void OnDrawGizmos()
       {
           if (isDrawGizmos)
           {
               Gizmos.DrawWireSphere(Vector3.zero, 10);
               GameObject go = new GameObject();
           }
       }
       ```

     - 如果在首次初始化时把开关设为true，启动了OnDrawGizmos类的代码，修改代码后还需要先进入一次Playmode后才能停止OnDrawGizmos的运行

     - 由于OnDrawGizmos裡new出来的资源不会被OnDisable或OnDestroy所释放，因此手动Destroy释放

   ----

## ~~Q17(Finished)~~

1. 什么是lua的尾调用，它有什么作用？

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

## ~~Q18(Finished)~~

1. 评估Lua字符串拼接的消耗，有没效率更高的拼接方式

   - Lua字符串的拼接方法及运行时间测试

     - ".."

       - ```lua
         function operatorConcat()
             local result = ""
             for i = 1, 100000 do
                 result = result .. "a"
             end
         end
         ```

     - string.format

       - ```lua
         function formatConcat()
         	local result = ""
         	for i = 1, 100000 do
         		result = string.format("%s%s", result, "a")
         	end
         end
         ```

     - 改用table.concat來拼接

       - ```lua
         function tableConcat()
             local t = {}
             for i = 1, 100000 do
                 table.insert(t,"a")
             end
             table.concat(t)
         end
         ```

     - 各运行5次并计时(/sec)

       - |               | 1st   | 2nd   | 3rd   | 4th   | 5th   | Avg.   |
         | ------------- | ----- | ----- | ----- | ----- | ----- | ------ |
         | ".."          | 0.48  | 0.433 | 0.397 | 0.399 | 0.406 | 0.423  |
         | string.format | 0.433 | 0.471 | 0.476 | 0.466 | 0.463 | 0.4618 |
         | table.concat  | 0.011 | 0.013 | 0.013 | 0.013 | 0.013 | 0.0126 |

       - table.conat > .. > string.format

## ~~Q19(Finished)~~

1. lua 表的2种初始化方式：哪种性能高 差多少 ？ 为什么？

   1. ```Lua
      for i = 1, 1000000 do
          local a = {}　
          a[1] = 1;
          a[2] = 2;
          a[3] = 3
      end
      ```

   2. ```lua
      for i = 1, 1000000 do
          local a = {true, true, true}
          a[1] = 1; 
          a[2] = 2; 
          a[3] = 3
      end
      ```

   - 第二种性能高，每次循环固定申请长度为3的数组，并进行赋值；第一种每次循环都先生成一个容量为0的数组，然后对该数组尝试赋值，会导致该数组发生Resize。而Resize会使数组先在内存中分配一个新的长度的数组（除第一个元素外，之后都每次遇到数组爆满时，数组容量成倍增加），然后将所有记录再哈希一遍，将原来记录转移到新数组中。

     - a[1] = 1时，数组容量扩展为1
     - a[2] = 2时，数组容量扩展为1 * 2 = 2
     - a[3] = 3时，数组容量扩展为2 * 2 = 4

   - 具体时间差（以下面的代码运行5次的结果平均时间差）：

     - ```lua
       function NonDefinedLengthArr()
           for i = 1, 1000000 do
               if i == 1 then
                   begin_nd = os.clock()
               end
       
               local a = {}
               a[1] = 1
               a[2] = 2
               a[3] = 3
       
               if i == 1000000 then
                   print(string.format("nonDefine total time:%.2fms\n", ((os.clock() - begin_nd) * 1000)))
               end
           end
       end
       
       function DefinedLengthArr()
           for i = 1, 1000000 do
               if i == 1 then
                   begin_d = os.clock()
               end
       
               local a = {true, true, true}
               a[1] = 1
               a[2] = 2
               a[3] = 3
       
               if i == 1000000 then
                   print(string.format("define total time:%.2fms\n", ((os.clock() - begin_d) * 1000)))
               end
           end
       end
       
       NonDefinedLengthArr() 
       DefinedLengthArr()
       ```
       - |            | 1st   | 2nd   | 3rd   | 4th   | 5th   | Avg.      |
         | ---------- | ----- | ----- | ----- | ----- | ----- | --------- |
         | NonDefined | 938ms | 783ms | 848ms | 808ms | 766ms | **828.6** |
         | Defined    | 432ms | 450ms | 454ms | 437ms | 409ms | **436.4** |

       - 第二种的运行时间比第一种的运行时平均快接近2倍

     - 在内存占用上，分别在两个方法的最后一次遍历后使用（collectgarbage("count") * 1024）进行测试返回lua使用的总内存字节数。

       - ```lua
         print(string.format("nonDefineMem:%.2f", collectgarbage("count") * 1024))
         ```

       - 第一种返回33603.00字节

       - 第二种返回27056.00字节

## ~~Q20(Finished)~~

1. 测试一下以下2种代码的性能差异

   1. ```lua
      for i = 1, 1000000 do　
          local x = math.sin(i)
      end
      ```

   2. ```lua
      local sin = math.sin
      for i = 1, 1000000 do　　
          local x = sin(i)
      end
      ```

   - 测试代码

     - ```Lua
       function Sin_In()
           for i = 1, 1000000 do
               if(i == 1) then
                   begin_In = os.clock()
               end
       
               local x = math.sin(i)
       
               if(i == 1000000) then
                   print(string.format("Sin_In Time: %.2f ms\n", (os.clock() - begin_In) * 1000))
                   print(string.format("Sin_In Mem: %.2f", collectgarbage("count") * 1024))
               end
           end
       end
       
       local sin = math.sin
       function Sin_Out()
           for i = 1, 1000000 do
               if(i == 1) then
                   begin_Out = os.clock()
               end
       
               local x = sin(i)
       
               if(i == 1000000) then
                   print(string.format("Sin_Out Time: %.2f ms\n", (os.clock() - begin_Out) * 1000))
                   print(string.format("Sin_Out Mem: %.2f", collectgarbage("count") * 1024))
               end
           end
       end
       
       for i = 1, 5 do
           print("Round: ", i)
           Sin_In()
           collectgarbage("restart")
           Sin_Out()
           collectgarbage("restart")
           print("+++++++++++++++++")
       end
       ```

   - 运行时间差异：

     - |         | 1st  | 2nd  | 3rd  | 4th   | 5th  | Avg.     |
       | ------- | ---- | ---- | ---- | ----- | ---- | -------- |
       | Sin_In  | 88ms | 88ms | 91ms | 112ms | 93ms | **94.4** |
       | Sin_Out | 77ms | 76ms | 73ms | 86ms  | 77ms | **77.8** |

     - 在math.sin声明在函数外，且通过一个变量来调用的运行效率比在函数体裡每次循环调用math.sin函数的运行效率平均高1.2倍，主要是因为避免了方法的动态编译

   - 内存佔用差异：

     - |         | 1st   | 2nd   | 3rd   | 4th   | 5th   | Avg.    |
       | ------- | ----- | ----- | ----- | ----- | ----- | ------- |
       | Sin_In  | 26634 | 24760 | 24932 | 25105 | 25277 | 25341.6 |
       | Sin_Out | 26483 | 24837 | 25009 | 25182 | 24993 | 25300.8 |

     - 两种方法在使用的总内存数差异上并不明显

## ~~Q21(Finished)~~

1. local 变量的在访问，在函数中定义直接访问和在文件中定义 性能上有何区别？

   - 运行代码之前，Lua 会把源代码翻译（预编译）成一种内部格式，这种格式由一连串虚拟机的指令构成，与真实 CPU 的机器码很相似。接下来，这一内部格式交由 C 代码来解释，基本上就是一个 `while` 循环，里面有一个很大的 `switch`，一种指令对应一个 `case`。

   - 自 5.0 版起，Lua 使用了一个基于寄存器的虚拟机。这些「寄存器」跟 CPU 中真实的寄存器并无关联，因为这种关联既无可移植性，也受限于可用的寄存器数量。Lua 使用一个栈（由一个数组加上一些索引实现）来存放它的寄存器。

     - 每个活动的（active）函数都有一份活动记录（activation record），活动记录占用栈的一小块，存放着这个函数对应的寄存器。

     - 因此，每个函数都有其自己的寄存器。由于每条指令只有 8 个 bit 用来指定寄存器，每个函数便可以使用多至 250 个寄存器。

     - Lua 的寄存器如此之多，预编译时便能将所有的局部变量存到寄存器中。所以，在 Lua 中访问局部变量是很快的。举个例子， 如果`a` 和 `b` 是局部变量，语句`a = a + b` 只生成一条指令：`ADD 0 0 1`（假设 `a` 和 `b` 分别在寄存器 `0` 和 `1` 中）。

     - 对比之下，如果 a 和 b 是全局变量，生成上述加法运算的指令便会如下：

       - ```C
         GETGLOBAL    0 0     ; a
         GETGLOBAL    1 1     ; b
         ADD          0 0 1
         SETGLOBAL    0 0     ; a
         ```

- 測試

  - 全局变量

    - ```lua
      a = 0
      function Global()
          for i = 1, 1000000 do
              if(i == 1) then
                  timer = os.clock()
              end
      
              a = a + 1
      
              if(i == 1000000) then
                  print(string.format("Global Time: %.2fms\n", (os.clock() - timer) * 1000))
              end
          end
      end
      Global()
      ```

    - 运行时间：35.00ms

  - 外层局部变量

    - ```lua
      local b = 0
      function OutSideLocal()
          for i = 1, 1000000 do
              if(i == 1) then
                  timer = os.clock()
              end
      
              b = b + 1
      
              if(i == 1000000) then
                  print(string.format("OutSide Local Time: %.2fms\n", (os.clock() - timer) * 1000))
              end
          end
      end
      OutSideLocal()
      ```

    - 运行时间：27.00ms

  - 内层局部变量

    - ```lua
      function InsideLocal()
          local c = 0
          for i = 1, 1000000 do
              if(i == 1) then
                  timer = os.clock()
              end
      
              c = c + 1
      
              if(i == 1000000) then
                  print(string.format("Inside Local Time: %.2fms\n", (os.clock() - timer) * 1000))
              end
          end
      end
      ```

    - 运行时间：19.00ms

  - 可见：函数内部的local变量访问的速度 > 函数外部的local变量访问速度 > 全局变量的访问速度