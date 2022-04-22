# 第三章、Windows编程基础
## 1.WinMain函数——程序的入口
### 1.Winmain函数
**函数头：**
```c++
int APIENTRY wWinMain(_In_ HINSTANCE hInstance,
                     _In_opt_ HINSTANCE hPrevInstance,
                     _In_ LPWSTR    lpCmdLine,
                     _In_ int       nCmdShow)
```
#### 前缀 
其中APIENTRY（教材中是WINAPI）
```c++
#define WINAPI      __stdcall
#define CALLBACK    __stdcall
#define APIENTRY    WINAPI
```
APIENTRY 等同于 **__stdcall**,是一种函数调用约定，被称为pascal调用约定，微软常用，__stdcall意味着：
1.参数从右向左压入堆栈
2.函数自身修改堆栈
3.函数名前加下划线，后跟@，再跟参数尺寸
编译时，**function(int a,int b)** 翻译成 **function@8**

1.代码中有 `_In_` 前缀是一个宏定义，表示这个参数需要我们自行输入
2.同理`_Out_`前缀，表示这个参数是 函数本身向外输出的参数

**`_In_Opt_` means you may pass NULL 表示可以传NULL参数**
#### 参数
1.`_In_ HINSTANCE hInstance`
- 表示实例句柄 h理解为handle，Instance表示实例
- hInstance是一个数值，当程序运行时，对应一个运行中的实例，只有运行中的程序实例，才有资格分配到实例句柄
- 一个应用程序可运行多个实例，每运行一个实例，系统就分配一个句柄值，通过hInstance传给WinMain函数

2.`_In_opt_ HINSTANCE hPrevInstance`

- 表示当前实例的**前一个实例**的句柄 
- h表示handle，Prev表示previous，Instance是实例
- 之前的MSDN中表示Win32里hPrevInstance一直是NULL

3.`_In_ LPWSTR lpCmdLine`
- 一个以 空 结尾的字符串，指定传给应用程序的命令行参数
- lp表示是指针，Cmd表示command命令，CmdLine表示命令行
- 例如：当我们打开E盘的1.txt文件时候，会将`E:/1.txt`作为命令行参数传递给 记事本 程序的WinMain函数

4.`_In_ int nCmdShow`

- 决定窗口如何显示,传参就是枚举，下面是vs2019枚举值

- ![image-20220412153425246](WindowsProgram.assets\image-20220412153425246.png)![image-20220412170200752](WindowsProgram.assets\image-20220412170200752.png)


### 2.MessageBox函数

```C++
MessageBox(NULL, L"hello,Visual studio", L"消息窗口", 0);
// 函数声明：
WINUSERAPI
int
WINAPI
MessageBoxA(
    _In_opt_ HWND hWnd,
    _In_opt_ LPCSTR lpText,
    _In_opt_ LPCSTR lpCaption,
    _In_ UINT uType);
WINUSERAPI
int
WINAPI
MessageBoxW(
    _In_opt_ HWND hWnd,
    _In_opt_ LPCWSTR lpText,
    _In_opt_ LPCWSTR lpCaption,
    _In_ UINT uType);
#ifdef UNICODE
#define MessageBox  MessageBoxW
#else
#define MessageBox  MessageBoxA
#endif // !UNICODE
```

#### 参数

1. HWND类型hWnd，表示消息框 所属的窗口的 句柄，设置为NULL表示从属于桌面
2. LPCTSTR，以NULL结尾的字符串，表示要显示的消息内容
3. LPCTSTR，以NULL结尾的字符串，表示消息框的标题内容
4. UINT的uType，消息窗口的样式，填写枚举值

![image-20220412192410833](WindowsProgram.assets\image-20220412192410833.png)

![image-20220412193033070](WindowsProgram.assets\image-20220412193033070.png)

最后一个参数uType可以使用 逻辑或 | 连接想要的窗口类型，`MB_YESNO|MB_ICONQUESTION`表示两个都要

#### 返回值

- 若是我们创建了对应按钮的 信息框，就需要给出按键返回值

- ![image-20220412212955388](WindowsProgram.assets\image-20220412212955388.png)

- 常用于错误的提示，比如提示缺少某个.dll文件

- ```c++
  if(errorCode){
  	MessageBox(NULL, L"这里写错误信息", L"这里写错误标题", 0);		
  } 
  ```


### 3.PlaySound 函数

使用PlaySound之前需要连接 winmm.lib 库文件

```c++
BOOL PlaySound(
   LPCTSTR pszSound,
   HMODULE hmod,
   DWORD   fdwSound
);
```

1. LPCTSTR字符串， 指定播放声音文件，NULL时，停止当前播放所有声音
2. HMODULE的 hmod，第一个参数指定文件 作为资源的可执行文件的句柄，暂时设为NULL
3. DWORD的fdwSound，控制声音播放标志，可使用多个标志，用"|"连接

![image-20220414165507204](WindowsProgram.assets\image-20220414165507204.png)

下用`SND_FILENAME|SND_ASYNC` 两个标识，标志第一个参数是文件名，并且声音是异步播放（播放完之后返回调用点）

Returns **TRUE** if successful or **FALSE** otherwise.

### 4.实例程序Firstbolood















