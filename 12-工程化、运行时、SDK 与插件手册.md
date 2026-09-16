本手册包含 1 份源文件：E:\GithubProject\docs\general\计算机基础知识整合指南\09-工程化运行时SDK与插件.md

# 工程化、运行时、SDK 与插件

### 9.3 CLI开发

#### 命令行参数是怎么解析的？

```
你在终端输入 python app.py --message "你好" 并按回车：

```
Shell进程（如bash）收到字符串后：

```
Shell执行 fork() 时：

```
子进程执行 execv("/usr/bin/python", argv)：

```
Python解释器启动后：

```
app.py 中：

```bash
命令 [子命令] [选项/参数]

**问题**：终端如何解析命令和参数？python app.py --message "你好" 是怎么执行的？

**回答**：

**第1步：键盘输入被Shell接收**

1. 键盘硬件产生扫描码
2. 键盘驱动把扫描码转换成字符
3. 终端模拟器把字符显示在屏幕上
4. 按回车时，终端把整行字符串发送给Shell进程
```

**第2步：Shell解析字符串**

// bash源码（简化）
void execute_command(char *line) {
    // 1. 词法分析：把字符串拆分成token
    char **argv = tokenize(line);
    // argv = ["python", "app.py", "--message", "你好"]
  
    // 2. 检查是否是内置命令（cd、export等）
    if (is_builtin(argv[0])) {
        execute_builtin(argv);
        return;
    }
  
    // 3. 查找外部命令
    char *path = find_in_path(argv[0]);
    // path = "/usr/bin/python"
  
    // 4. 创建子进程执行
    pid_t pid = fork();  // 复制当前进程
    if (pid == 0) {
        // 子进程
        execv(path, argv);  // 替换为python程序
    } else {
        // 父进程（Shell）
        waitpid(pid, NULL, 0);  // 等待子进程结束
    }
}
```

**第3步：fork()系统调用**

// 系统调用
pid_t fork(void);

CPU执行流程：

1. Shell程序执行 fork 系统调用：
   MOV EAX, 57    ; fork的系统调用号
   INT 0x80       ; 触发系统调用

2. 内核的 fork 实现：
   int sys_fork() {
       // 复制当前进程
       struct task_struct *child = copy_process(current);
   
       // 子进程返回0
       // 父进程返回子进程PID
       if (current == child) {
           return 0;  // 子进程
       } else {
           return child->pid;  // 父进程
       }
   }

3. fork() 返回后：
   - 有两个进程在运行：父进程（Shell）和子进程
   - 两个进程的代码、数据、栈都相同
   - 唯一区别：返回值不同
```

**第4步：exec()系统调用**

// execv 的作用：用新程序替换当前进程

int execv(const char *path, char *const argv[]) {
    // 1. 读取可执行文件
    struct elf_header *elf = read_file(path);
  
    // 2. 检查文件格式
    if (elf->magic != ELF_MAGIC) {
        return -1;
    }
  
    // 3. 加载程序段到内存
    for (each segment in elf) {
        void *addr = mmap(segment->vaddr, segment->size);
        memcpy(addr, segment->data, segment->size);
    }
  
    // 4. 设置参数
    // argv 被放入新程序的栈中
    // argc 也被放入栈中
  
    // 5. 跳转到入口点
    // CPU开始执行python解释器的代码
    jump_to(elf->entry_point);
}

exec() 后：
- 子进程不再是Shell的副本
- 子进程现在是Python解释器
- argv 被传递给新程序
```

**第5步：Python解释器接收参数**

// Python解释器入口
int main(int argc, char **argv) {
    // argc = 4
    // argv = ["python", "app.py", "--message", "你好"]
  
    // 1. 初始化Python运行时
    Py_Initialize();
  
    // 2. 创建 sys.argv
    PyObject *sys_argv = PyList_New(argc);
    for (int i = 0; i < argc; i++) {
        PyList_SetItem(sys_argv, i, PyUnicode_FromString(argv[i]));
    }
    PySys_SetObject("argv", sys_argv);
  
    // 3. 执行脚本
    FILE *fp = fopen(argv[1], "r");  // 打开 app.py
    PyRun_SimpleFile(fp, argv[1]);
  
    // 4. 清理并退出
    Py_Finalize();
    return 0;
}
```

**第6步：Python脚本访问参数**

import sys
print(sys.argv)

sys.argv 是一个列表：
["python", "app.py", "--message", "你好"]

Python脚本可以：
- sys.argv[1] 获取脚本名
- sys.argv[2:] 获取其他参数
- 使用 argparse 库解析参数
```

#### CLI命令的通用模式

**问题**：所有命令行工具的参数格式有什么规律？

**回答**：

# git
git commit -m "message"
git push origin main

# python
python -m venv myenv
python script.py --input data.csv

# npm
npm install express
npm run build

# docker
docker container ls
docker run -d nginx
```

## API 格式、Base URL 与常见报错排查

### 1. Base URL、完整 endpoint、Chat Completions 和 Responses API 怎么区分？

| 名称          | 含义                | 示例                                        |
| ------------- | ------------------- | ------------------------------------------- |
| Base URL      | API 服务的根地址    | https://api.example.com                     |
| Endpoint      | 具体接口路径        | /v1/chat/completions                        |
| 完整 endpoint | Base URL + Endpoint | https://api.example.com/v1/chat/completions |

**合并后的问题：**
配置模型客户端时，Base URL、完整 endpoint、Chat Completions、Responses API 经常混在一起。它们到底怎么区分？为什么路径填错会导致 404、400 或协议错误？

**答案：**

先分清两个概念：

---

### 3. 401、404、400、502、503、stream disconnected 分别是什么意思？

```text
1. key 填错。
2. key 属于另一个服务。
3. key 被删除、禁用、过期或重置。
4. Authorization 格式不对，例如漏了 Bearer。
```

```text
1. Base URL 填成了完整 endpoint。
2. 服务不支持 /v1/responses，却请求了 Responses API。
3. 反代路径没有正确转发。
4. API 类型选错，客户端自动拼出了错误路径。
```

```text
1. Chat Completions 接口里发了 Responses API 的 input/reasoning/text 字段。
2. Responses API 里发了 messages。
3. model 名称写错。
4. response_format 或工具调用字段不被当前服务支持。
```

| 报错                                  | 主要含义               | 更可能的问题位置                        | 处理方向                                                      |
| ------------------------------------- | ---------------------- | --------------------------------------- | ------------------------------------------------------------- |
| 401 Unauthorized / INVALID_API_KEY    | 鉴权失败               | key、服务商、代理鉴权层                 | 检查 key 类型、是否过期、是否填错服务                         |
| 404 Not Found / openresty             | 路径不存在             | Base URL、endpoint、API 类型            | 检查是否把完整 endpoint 填进 Base URL，或请求了不支持的路径   |
| 400 Invalid request                   | 请求体或参数不合法     | 请求字段、模型名、API 协议              | 检查 messages/input、model、response_format、reasoning 等字段 |
| 502 Bad Gateway                       | 网关拿不到上游正常响应 | 本地代理、反代、网关                    | 看代理日志，绕过代理直连测试                                  |
| 503 Service Unavailable               | 服务暂时不可用         | 服务商、sub2api、上游账号池             | 等待恢复或换账号/模型/服务商                                  |
| 503 No available accounts             | 没有可用上游账号       | sub2api 后台账号池                      | 找服务商或更换可用账号池                                      |
| stream disconnected before completion | 流式响应中途断开       | 流式链路、代理、CDN、账号池、客户端超时 | 关闭 stream 测试，检查超时和代理链路                          |

**合并后的问题：**
模型 API 调用时出现 401、404、400、502、503、stream disconnected before completion，这些报错分别说明哪一层出了问题？应该怎么处理？

**答案：**

先看总表：

401 的重点是“你是谁”没通过。常见原因：

404 的重点是“这个地址不存在”。常见原因：

400 的重点是“地址找到了，但参数不对”。常见原因：

502 的重点是“网关和上游之间断了”。如果你本地用了 127.0.0.1 代理，要先绕过它直连服务商测试。能直连成功，说明问题在本地代理或反代；直连也失败，才继续看服务商。

503 的重点是“服务现在不可用”。如果是 sub2api 的 No available accounts，本质通常不是你本地代码错，而是上游账号池没有可用账号。

### 4. 不同厂商的大模型 API 请求格式有什么差异？

## SDK、库、框架与插件系统

### 1. SDK 是什么意思？既然有库，为什么还需要 SDK？

```text
Software Development Kit
软件开发工具包
```

```text
核心库
认证封装
错误处理
重试机制
参数校验
示例代码
文档
工具链
调试工具
部署说明
安全模块
版本兼容处理
```

```text
请求地址
认证头
JSON 序列化
错误码
超时
限流
重试
流式响应
连接池
版本兼容
```

```text
库解决“有这个功能可用”，SDK 解决“怎么快速、正确、安全地接入这个平台”。
```

| 对比项       | 库           | SDK                  |
| ------------ | ------------ | -------------------- |
| 核心定位     | 提供单个功能 | 提供完整接入方案     |
| 范围         | 窄           | 宽                   |
| 是否特定平台 | 通常更通用   | 通常面向特定平台     |
| 文档和示例   | 不一定完整   | 通常配套完整         |
| 错误处理     | 你自己写     | SDK 通常内置         |
| 维护责任     | 主要靠你集成 | 官方承担大量兼容维护 |

**合并后的问题：**
代码领域里 SDK 是什么？不是已经有库了吗，为什么还需要 SDK？

**答案：**
SDK 全称是：

它不是单纯的库，而是一整套帮助你快速接入某个平台、服务、系统的开发包。

库通常只提供功能函数。
SDK 提供的是完整接入方案。

一个 SDK 通常包括：

比如你调用一个大模型 API。

如果只用普通 HTTP 库，你要自己处理：

如果用官方 SDK，很多都已经封装好了。

所以区别是：

一句话：

### 5. 插件系统完整总结：主程序、插件、注册表、容器、工厂函数和 Agent 插件怎么串起来？

```text
主程序提供：
├── _PLUGIN_REGISTRY（全局注册表）
├── @define_plugin_entry（装饰器，往注册表里写）
├── OpenClawPluginApi（容器）
└── PluginEntry（插件基类规范）

```python
# 插件开发者从主程序框架 import 需要的东西
from openclaw import define_plugin_entry, PluginEntry, OpenClawPluginApi

**合并后的问题：**
插件系统到底是怎么运行的？主程序、插件、装饰器、注册表、容器、工厂函数和 Agent 插件之间是什么关系？

**答案：**

#### 一、插件是什么

插件是一个遵守主程序规范的普通类，通过装饰器自动登记，通过 `register()` 方法把功能交给主程序。

插件系统的本质是：主程序提供骨架，插件填充具体功能，主程序代码永远不需要改，也永远不需要知道任何插件的名字。

#### 二、主程序提供的规范

主程序提供了插件开发者需要的所有规范性的东西：

插件开发者只需要：
├── import 这些东西
├── 写自己的 register() 逻辑
└── 把文件丢进 plugins/ 目录
```

插件文件头部就是从主程序框架里 import 这些规范：

# 然后用主程序提供的装饰器
@define_plugin_entry(id="my-plugin", name="My Plugin", kind="skill")
class MyPlugin(PluginEntry):
    def register(self, api: OpenClawPluginApi):
        ...
```

生命周期节点触发 Hook：

```python
# 装饰器内部本质上就是这一件事
_PLUGIN_REGISTRY.append({
    "id": "memory-core",
    "cls": MemoryCorePlugin   # 存的是类本身，不是实例
})
```

```python
class OpenClawPluginApi:
    def __init__(self):
        self._tools = []      # 工具列表
        self._hooks = {}      # 钩子字典 {"事件名": [handler1, handler2]}
        self._cli = []        # CLI 命令列表
        self._memory = []     # 记忆能力列表

```text
下载插件文件 -> 丢进 ./plugins/ 目录 -> 重启主程序
```

```python
# 主程序只有这段死代码，永远不需要改
for filename in os.listdir("./plugins"):
    importlib.import_module(filename)   # 无脑 import 进来
```

```python
api = OpenClawPluginApi()   # 创建空容器

```python
def register(self, api):
    api.register_tool(create_memory_search_tool)   # 工具
    api.register_memory_capability(...)            # 记忆能力
    api.register_cli(register_memory_cli)          # CLI 命令
    api.register_hook("before_save", my_handler)   # 钩子
```

```python
def register(self, api):
    api.register_tool(
        factory=create_blur_tool,
        options=ToolOptions(
            ui=Button(label="模糊")   # 按钮是插件自己给的
        )
    )
```

```python
for tool in api._tools:
    button = tool.options.ui
    button.on_click = lambda: tool.execute()   # 渲染时就绑定好了
    toolbar.add_button(button)
```

```text
按钮长什么样 -> 插件决定
按钮放在哪个位置 -> 主程序决定
点击后执行什么 -> 渲染阶段绑定好了，主程序不需要认识“模糊”这个名字
```

```text
用户点击“模糊”按钮
    ↓
触发 button.on_click
    ↓
执行 tool.execute()   ← 渲染时就绑好了，不需要再去找
```

```python
# 主程序固定写好的节点
def save_file(file):
    emit_hook("before_save", file)    # 触发所有监听这个事件的 handler
    do_actual_save(file)
    emit_hook("after_save", file)

#### 三、装饰器的作用

`@define_plugin_entry` 是主程序提供的装饰器，因为它操作的是主程序的全局注册表 `_PLUGIN_REGISTRY`。

这个文件被 import 的瞬间，装饰器立刻执行，把这个类存进全局注册表：

存的是类本身，不是实例，此时什么功能都没有执行。

#### 四、容器是什么

容器就是 `OpenClawPluginApi` 对象，本质上只是几个普通的列表和字典：

    def register_tool(self, factory, options):
        self._tools.append((factory, options))  # 就是往列表里塞

    def register_hook(self, name, handler):
        self._hooks.setdefault(name, []).append(handler)
```

`register_*` 方法本质上只是往容器里添加东西，没有任何魔法。

#### 五、完整运行流程

第一阶段：你下载插件。

必须重启是因为插件加载是启动时的一次性行为，主程序运行起来之后不再扫描目录。这就是为什么所有软件装了插件都要刷新或重启的原因。

如果不需要重启，是因为额外实现了热加载，主程序持续监听目录变化，发现新文件就动态 import，这是额外的工程，不是默认行为。

第二阶段：扫描目录，import 插件。

主程序认的是目录，不认具体插件名字。import 的副作用就是触发装饰器，让插件类自动登记进注册表。

第三阶段：实例化插件，调用 `register()`。

for entry in _PLUGIN_REGISTRY:
    plugin = entry["cls"]()    # 实例化
    plugin.register(api)       # 插件把功能写进容器
```

插件在 `register()` 里把自己所有的东西都交出来：

所有插件 `register()` 完毕后，容器里有了所有功能，但还没有被执行。

第四阶段：渲染 UI。

插件在 `register()` 时，主动把 UI 元素一起交给主程序：

主程序只看容器里有什么，就渲染什么，同时把点击事件绑定好：

分工是：

第五阶段：运行时触发。

用户点击按钮：

主程序不会在运行时写 `find_tool_by_name("模糊")` 这种代码，因为按钮和功能在渲染阶段就已经捆绑成一对了。

# emit_hook 就是遍历容器挨个调用
def emit_hook(name, payload):
    for handler in self._hooks.get(name, []):
        handler(payload)
```

```python
# 不这样做
api.register_tool(MemorySearchTool())   # 固定死了，无法按上下文定制

插件不决定什么时候被调用，只决定被调用时做什么。

#### 六、为什么注册的是工厂函数而不是实例

# 而是传工厂函数
api.register_tool(create_memory_search_tool)

# 调用时：tool = create_memory_search_tool(context)
# context 包含当前会话、用户信息等动态数据
```

工具需要在运行时根据上下文动态创建，不能提前固定。

---
