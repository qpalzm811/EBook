## 1.Gameobject判断物体和父节点 是否激活 活跃 active

- ``gameObject.active`` 和``gameObject.activeHierarchy``都可以显示父节点 **不激活状态** 返回 **不激活**
- ``gameObject.activeSelf``仅显示自身是否激活，不管父节点是否激活

   [gameObject.activeHierarchy](https://docs.unity.cn/cn/2020.3/ScriptReference/GameObject-activeInHierarchy.html) 与 [GameObject.activeSelf](https://docs.unity.cn/cn/2020.3/ScriptReference/GameObject-activeSelf.html) 不同，它还检查是否有父 GameObject 影响此 GameObject 的当前活动状态。

   若因父节点隐藏，gameObject.activeHierarchy一定是 不活跃，activeSelf 则仅看他自己是不是活跃

