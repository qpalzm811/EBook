## 1.Gameobject判断物体和父节点 是否激活 活跃 active

- ``gameObject.active`` 和``gameObject.activeHierarchy``都可以显示父节点 **不激活状态** 返回 **不激活**
- ``gameObject.activeSelf``仅显示自身是否激活，不管父节点是否激活

   [gameObject.activeHierarchy](https://docs.unity.cn/cn/2020.3/ScriptReference/GameObject-activeInHierarchy.html) 与 [GameObject.activeSelf](https://docs.unity.cn/cn/2020.3/ScriptReference/GameObject-activeSelf.html) 不同，它还检查是否有父 GameObject 影响此 GameObject 的当前活动状态。

   若因父节点隐藏，gameObject.activeHierarchy一定是 不活跃，activeSelf 则仅看他自己是不是活跃

## 2.递归多层次结构，并且每个gameobject分别进行处理，性能极糟糕

[Jobs](https://catlikecoding.com/unity/tutorials/basics/jobs/)

![image-20230731151207050](Unity%E5%B0%8F%E9%97%AE%E9%A2%98.assets/image-20230731151207050.png)

Unity的sphere比cube的顶点多，但是性能基本没啥差距，说明不是顶点太多的问题，瓶颈bottleneck在cpu。

## 3.什么情况下调用awake但是不调用 start

1. 当一个游戏对象实例化后立即被销毁

   （例如，在同一帧中使用`Instantiate`创建并立即销毁对象），`Awake`函数会被调用，但是由于对象没有机会被激活，`Start`函数将不会被调用。

2. 在脚本附加到游戏对象之后，再将该游戏对象设置为非激活状态，然后重新激活它。

   在这种情况下，`Awake`函数会在脚本附加时被调用，但由于对象一开始处于非激活状态，`Start`函数不会被调用。只有当你将对象重新激活时，`Start`函数才会被调用。

## 4.不要用低效的SendMessage

```c#
// 1 SendMessage
SendMessage(nameof(SendMessageHelper.CallThisFunction), 69);
// 2 获取组件
var caller = GetComponent<SendMessageHelper>();
caller.CallThisFunction(69);
```

第二种方法还能验证这个函数在不在。

性能差距在两倍左右 SendMessage447ms  获取组件202ms













