本手册包含 1 份源文件：E:\GithubProject\docs\general\计算机基础知识整合指南\03-数据结构算法与设计模式.md

# 数据结构、算法与设计模式

#### 什么是数据结构？

```
最简单的理解：

操作方式和代价是从组织方式推导出来的，不是独立的部分：
  数组连续存 → 能按下标直接跳 → O(1)
  链表一个串一个 → 只能从头走 → O(n)
  哈希表按hash值放 → 算一下就定位 → O(1)

```
操作系统里的：
  - 页表（Page Table）：数组，用页号（虚拟地址的高位）作为下标，直接查表得到物理地址
  - 进程控制块（PCB）：结构体，存储进程的所有信息
  - 中断向量表：数组，用中断号作为下标，直接查表得到处理程序地址
  - socket表：数组，用端口号作为下标，直接查表得到socket对象

**是什么**：
数据结构 = **数据 + 组织方式**。它决定了数据怎么存、怎么取、怎么增删改查。

你有一堆数据，怎么放？
- 排成一排？ → 数组（Array）
- 一个串一个？ → 链表（Linked List）
- 像字典一样按名字查？ → 哈希表（Hash Map）
- 像树一样分层？ → 树（Tree）
- 像栈一样后进先出？ → 栈（Stack）

每种"放法"就是一种数据结构。

"数据"反而是最不重要的部分——数据结构关心的是结构本身，
不管你存的是整数、字符串还是对象，数组就是数组，链表就是链表。
```

这四个都是数组而不是哈希表，因为key（页号、中断号、端口号）本身就是连续整数，
可以直接当数组下标用，不需要哈希。映射不一定需要哈希，key是连续整数时数组就是最直接的映射。
```

## 数据结构底层、栈、树、指针与设计模式补充

```text
告诉 CPU 数据怎么存、怎么找、怎么操作。
```

```text
base
  ↓
+0   +4   +8   +12
[1]  [2]  [3]  [4]
```

```text
目标地址 = 数组起始地址 + i * 单个元素大小
```

```text
socket 指针所在地址 = port_table 基址 + 端口号 * 指针大小
```

| 数据结构 | 找下一个数据的方式             |
| -------- | ------------------------------ |
| 数组     | 起始地址 + 元素大小 -->偏移量  |
| 链表     | 当前节点里存着下一个节点的地址 |
| 栈       | 只能访问栈顶                   |
| 树       | 当前节点里存子节点地址         |
| 哈希表   | 通过哈希函数算位置             |

| 数组类型                      | 数组元素实际存放的内容 |
| ----------------------------- | ---------------------- |
| `int arr[4]`                | 四个连续的整数值       |
| `struct Node arr[4]`        | 四个连续的结构体       |
| `struct socket *arr[65536]` | 65536 个连续的指针值   |

**问题合并：**
是不是**一个数据存在一个空间里，CPU 读取这个数据，然后怎么读取下一个数据，就是数据结构**？

**内存可以理解成一个超大的格子数组，每个格子有地址。CPU 只能按地址读写数据。**

但是 CPU 自己不知道“下一个数据在哪里”。
“下一个数据在哪里”是数据结构告诉 CPU 的。

不同数据结构告诉 CPU 的方式不同：

所以，数据结构的核心作用就是：

#### 数组、指针数组和下标访问的底层机制是什么？

**问题**：数组是不是一块连续内存？数组里到底存数据还是存指针？用下标访问时，底层是不是查了一张表？

数组的核心特征是：**一段连续内存 + 固定大小元素 + 基址偏移计算**。

数组本身不会自动带有“数据区 + 指针区”的双层结构。数组里存什么，完全取决于元素类型：

普通整数数组的内存模型可以理解为，一个数组的头部就有数组的各种信息，如元素大小：

如果 `int` 占 4 字节，那么访问 `arr[i]` 时，写这个代码，被翻译过来就是有起始地址、然后去数组的头信息查元素大小、然后有计算，核心计算是：

下标只是偏移量的简写，CPU 最终仍然是按地址读写内存。这就是数组随机访问是 `O(1)` 的根本原因。

因此，**端口表这类结构可以用端口号直接当数组下标**：

**数组本体连续，数组元素也连续；但如果数组元素是指针，指针指向的真实对象不要求连续。链表、树、内核 socket 对象等都可能分散在堆内存不同位置，数组里只保存它们的地址。**

### 14. 设计模式是什么？常见设计模式有哪些？

```python
# 模块A需要配置
config1 = AppConfig()       # 读取配置文件，花1秒
config1.get(“db_host”)

| 类型   | 解决什么问题     | 例子                   |
| ------ | ---------------- | ---------------------- |
| 创建型 | 对象怎么创建     | 单例、工厂             |
| 结构型 | 类和对象怎么组合 | 装饰器、适配器、代理   |
| 行为型 | 对象之间怎么交互 | 策略、观察者、模板方法 |

**设计模式不是语法，是前人总结出来的代码组织经验。核心目标是：降低耦合、提高复用、方便扩展。**

经典设计模式分三类：

#### 单例模式（创建型）

**场景**：你做一个 App，需要一个全局配置管理器来存数据库地址、API 密钥等配置。如果到处 new 配置管理器，每个实例各存一份配置，改了一个其他的不知道，就会出问题。

**第一步：不用单例，直接写**

# 模块B也需要配置
config2 = AppConfig()       # 又读了一遍配置文件，又花1秒，浪费！
config2.get(“db_port”)
```

问题一：**重复创建**。配置文件被读了两次，浪费时间和内存。

问题二：**状态不同步**。

问题三：**语义不对**。配置管理器在业务上就应该只有一个，多个实例在逻辑上说不通。

```python
config1.set(“db_host”, “192.168.1.1”)
print(config2.get(“db_host”))  # 还是旧值！两个对象各存一份，改了一个另一个不知道
```

```python
class AppConfig:
    _instance = None        # 类变量，保存唯一的实例

```python
def checkout(order, pay_type):
    if pay_type == “微信”:
        pay_service = WechatPay()
    elif pay_type == “支付宝”:
        pay_service = AliPay()
    elif pay_type == “银行卡”:
        pay_service = BankPay()
    pay_service.pay(order.amount)
```

```python
# 工厂：专门负责创建支付对象
class PaymentFactory:
    @staticmethod
    def create(pay_type):
        if pay_type == “微信”:
            return WechatPay()
        elif pay_type == “支付宝”:
            return AliPay()
        elif pay_type == “银行卡”:
            return BankPay()

**第二步：用单例解决**

    def __new__(cls):
        # 每次 AppConfig() 都会调用 __new__
        if cls._instance is None:
            # 第一次创建：真的 new 一个对象
            cls._instance = super().__new__(cls)
            cls._instance._config = cls._instance._load_config()
        # 之后每次：直接返回已有的对象，不创建新的
        return cls._instance

    def _load_config(self):
        print(“读取配置文件...（只执行一次）”)
        return {“db_host”: “localhost”, “db_port”: 3306}

    def get(self, key):
        return self._config.get(key)

    def set(self, key, value):
        self._config[key] = value
```

**一句话**：配置管理器、日志对象、数据库连接池这类”全局只需要一个”的东西，用单例。

---

#### 工厂(专门负责创建对象的地方)模式（创建型）

**场景**：你做一个外卖 App，用户下单时要扣款。扣款方式有三种：微信支付、支付宝支付、银行卡支付。

##### 第一步：不用工厂(不在外面创建对象)，直接写

最直接的写法，在下单函数里 if-else：

这样写能跑，但有一个问题：**下单函数里混着”创建支付对象”的逻辑**。

下单函数的职责是”处理订单”，但现在它还得知道”微信支付怎么创建、支付宝怎么创建”。

更麻烦的是，如果要新增”Apple Pay”，你得改 checkout 函数，加一个 elif。checkout 函数被改来改去，容易出 bug。

##### 第二步：简单工厂——把”创建”的活儿交给别人

思路很简单：**把 if-else 创建对象的逻辑，搬到一个专门的工厂类里**。

# 下单函数：只管下单，不关心支付对象怎么来的
def checkout(order, pay_type):
    pay_service = PaymentFactory.create(pay_type)  # 找工厂要
    pay_service.pay(order.amount)
```

- checkout 函数变简单了，它不关心支付对象怎么创建
- 创建逻辑集中在一个地方，改起来方便

```python
from abc import ABC, abstractmethod

好处：

但还有问题：如果要新增”Apple Pay”，你还是得改 PaymentFactory 的 if-else。**工厂代码被改了，有可能改出 bug**。

##### 第三步：工厂方法——连工厂都不用改了

思路是：**不用一个工厂包办所有，而是每种支付方式有自己的工厂**。

# 工厂的”模板”：规定每个工厂必须能创建支付对象
class PaymentFactory(ABC):
    @abstractmethod
    def create(self): pass

# 微信支付工厂：专门创建微信支付
class WechatPayFactory(PaymentFactory):
    def create(self):
        return WechatPay()

# 支付宝工厂：专门创建支付宝
class AliPayFactory(PaymentFactory):
    def create(self):
        return AliPay()

# 银行卡工厂：专门创建银行卡支付
class BankPayFactory(PaymentFactory):
    def create(self):
        return BankPay()
```

核心问题：**所有优惠规则堆在一个函数里，互相耦合，改一个可能影响另一个。**

使用时：

```python
# 用户选了微信支付 → 用微信工厂创建
factory = WechatPayFactory()
pay_service = factory.create()
pay_service.pay(100)
```

```python
# 只需要新增一个工厂，不改任何已有代码
class ApplePayFactory(PaymentFactory):
    def create(self):
        return ApplePay()
```

```
第一步 直接写：
  checkout 里 if-else 创建对象
  问题：下单逻辑和创建逻辑混在一起，新增支付方式要改 checkout

#### 策略模式（行为型）

```python
def calculate_price(order, user):
    if user.is_new:
        return order.total * 0.8
    elif user.is_vip:
        return order.total * 0.9
    elif order.total >= 100:
        return order.total - 20
    elif user.coupon:
        return order.total - user.coupon.value
    else:
        return order.total
```

```text
产品经理说：加一个”周末特惠” → 改函数，加 elif
产品经理说：加一个”节日活动” → 又改函数，加 elif
产品经理说：加一个”邀请好友折扣” → 又改函数，加 elif

```python
# 每个策略类只管自己的计算逻辑
class NewUserDiscount:
    def calculate(self, order, user):
        return order.total * 0.8

```python
class Order:
    def __init__(self, total, strategy):
        self.total = total
        self.strategy = strategy     # 传入一个策略对象

使用：

```python
order = Order(200, NewUserDiscount())
print(order.checkout())  # 160（新人8折）

```python
# 只需要新建一个类，不改任何已有代码
class WeekendDiscount:
    def calculate(self, order, user):
        return order.total * 0.85   # 周末85折

**新增 Apple Pay 怎么办？**

这就是工厂方法的核心：**新增产品时，只加新代码，不改旧代码**。旧代码不动，就不会引入新 bug。

##### 三种方式对比

第二步 简单工厂：
  把创建逻辑搬到 PaymentFactory 里
  好处：checkout 不关心创建了
  问题：新增支付方式还是要改工厂的 if-else

第三步 工厂方法：
  每种支付方式一个工厂，各自创建各自的
  好处：新增支付方式只加新工厂，不改任何已有代码
```

**一句话**：**不想在业务代码里写 if-else 创建对象，就把创建逻辑交给工厂。简单工厂够用就用简单工厂，需要频繁新增产品就用工厂方法。**

---

**场景**：你做一个外卖平台，订单结算时要算优惠价。优惠规则有：新人 8 折、VIP 9 折、满 100 减 20、优惠券抵扣。

**第一步：不用策略，直接 if-else**

这样写能跑，但问题来了：

每次改同一个函数，这个函数越来越长，越来越难看懂。
而且改的时候可能不小心把已有的逻辑改坏了。
```

**第二步：用策略模式，把每个规则拆成独立的类**

class VipDiscount:
    def calculate(self, order, user):
        return order.total * 0.9

class FullReduction:
    def calculate(self, order, user):
        if order.total >= 100:
            return order.total - 20
        return order.total

class CouponDiscount:
    def calculate(self, order, user):
        return order.total - user.coupon.value
```

**第三步：在结算时选择用哪个策略**

    def checkout(self):
        return self.strategy.calculate(self, None)
```

order = Order(200, VipDiscount())
print(order.checkout())  # 180（VIP 9折）

order = Order(200, FullReduction())
print(order.checkout())  # 180（满100减20）
```

**新增”周末特惠”怎么办？**

# 直接用
order = Order(200, WeekendDiscount())
print(order.checkout())  # 170
```

核心问题：**事件的发布者（支付）和事件的接收者（短信、商家、仓库）直接耦合了。**

问题二：新增需求时要改支付代码。
  产品经理说：支付成功后通知数据分析平台 → 改 pay_success
  产品经理说：支付成功后发 Push 通知 → 又改 pay_success
  每次改支付代码，风险越来越高。

问题三：支付模块不需要知道”通知谁”。
  支付模块的职责是”处理支付”，不是”通知全世界”。
```

```text
不用策略：所有规则堆在一个函数里，改一个动全身
用策略：每个规则是独立的类，互不影响，想新增就新增

```python
def pay_success(order):
    print(“支付成功！”)
    # 直接调用所有下游模块
    sms_service.send(order.user.phone, “支付成功”)
    merchant_service.notify(order.merchant_id, “有新订单”)
    warehouse_service.prepare(order.items)
    points_service.add(order.user.id, order.total // 10)
```

```text
问题一：支付模块和短信、商家、仓库、积分四个模块绑死了。
  支付模块 import 了这四个模块，任何一个挂了，支付也可能受影响。

```python
# 观察者接口：所有接收者都要实现这个方法
class OrderObserver:
    def on_pay_success(self, order):
        pass

**策略模式的本质**：

就像餐厅的菜单：
  不用策略 = 把所有菜的做法写在一张纸上，改一个菜要翻整张纸
  用策略 = 每个菜一张卡片，改一个菜只动那张卡片
```

**一句话**：同一个动作有多种实现方式，且需要灵活切换和独立扩展，用策略。

---

#### 观察者模式（行为型）

**场景**：你做一个电商系统。用户下单支付成功后，系统要做四件事：发短信通知用户、通知商家发货、通知仓库备货、给用户加积分。

**第一步：不用观察者，直接在支付代码里调用**

这样写能跑，但问题很多：

**第二步：用观察者模式，发布者和订阅者解耦**

先把各种”接收者”统一成一个接口：

# 具体观察者：短信
class SmsObserver(OrderObserver):
    def on_pay_success(self, order):
        print(f”发送短信给 {order.user.phone}：支付成功”)

# 具体观察者：商家
class MerchantObserver(OrderObserver):
    def on_pay_success(self, order):
        print(f”通知商家 {order.merchant_id}：有新订单，请发货”)

# 具体观察者：仓库
class WarehouseObserver(OrderObserver):
    def on_pay_success(self, order):
        print(f”通知仓库：准备 {len(order.items)} 件商品”)

# 具体观察者：积分
class PointsObserver(OrderObserver):
    def on_pay_success(self, order):
        print(f”给用户 {order.user.id} 加 {order.total // 10} 积分”)
```

```python
class EventCenter:
    def __init__(self):
        self._listeners = {}

```python
# 启动时，各模块自己注册
event_center = EventCenter()
event_center.subscribe(“pay_success”, SmsObserver())
event_center.subscribe(“pay_success”, MerchantObserver())
event_center.subscribe(“pay_success”, WarehouseObserver())
event_center.subscribe(“pay_success”, PointsObserver())

再创建一个”事件中心”，管理谁在监听：

    def subscribe(self, event_type, observer):
        “””订阅事件：谁想监听什么事件”””
        if event_type not in self._listeners:
            self._listeners[event_type] = []
        self._listeners[event_type].append(observer)

    def publish(self, event_type, data):
        “””发布事件：通知所有订阅者”””
        for observer in self._listeners.get(event_type, []):
            observer.on_pay_success(data)
```

支付代码只管发布事件，不关心谁在监听：

# 支付成功时，只发一个事件
def pay_success(order):
    print(“支付成功！”)
    event_center.publish(“pay_success”, order)  # 通知所有订阅者
```

```python
# 新建一个观察者，注册一下，不改支付代码
class AnalyticsObserver(OrderObserver):
    def on_pay_success(self, order):
        print(f”记录数据：用户{order.user.id}消费{order.total}元”)

```text
不用观察者：支付代码直接调用所有下游模块 → 紧耦合，改一个动全身
用观察者：支付代码只发事件，各模块自己订阅 → 松耦合，各改各的

```python
class RedTea:
    def cost(self): return 10

**新增”通知数据分析”怎么办？**

event_center.subscribe(“pay_success”, AnalyticsObserver())
```

**观察者模式的本质**：

就像微信群发：
  不用观察者 = 你每次发消息要手动 @所有人，加一个人就要多 @一个
  用观察者 = 你只管往群里发，谁在群里谁收到，加人退人都不用你管
```

**一句话**：一个事件发生后，多个对象需要响应，且发布者不想知道接收者是谁，用观察者。

---

#### 装饰器模式（结构型，就是类中有类，一层层包装）

**场景**：你做一个奶茶店系统。奶茶有基础款（红茶 10 元、绿茶 8 元），然后顾客可以加牛奶（+3）、加糖（+1）、加珍珠（+2）、加椰果（+2）。

**第一步：用继承来做**

每种组合写一个类：

class RedTeaWithMilk(RedTea):
    def cost(self): return 13

class RedTeaWithSugar(RedTea):
    def cost(self): return 11

class RedTeaWithMilkAndSugar(RedTea):
    def cost(self): return 14

class RedTeaWithMilkSugarAndPearl(RedTea):
    def cost(self): return 16

# 绿茶也一样...
class GreenTea:
    def cost(self): return 8

核心思路：**每加一种配料，就用一个新的对象把原对象包起来，价格叠加。**

问题很明显：

```text
4种配料 × 2种茶底 = 要写多少类？
  0种配料：红茶、绿茶（2个）
  1种配料：红茶加牛奶、红茶加糖、红茶加珍珠、红茶加椰果、绿茶加牛奶...（8个）
  2种配料：红茶加牛奶加糖、红茶加牛奶加珍珠...（更多）
  3种配料、4种配料...
  
组合爆炸，根本写不完。
而且每新增一种配料（比如加芋圆），要改所有相关的类。
```

```python
# 基础奶茶（和配料共用同一个接口：都有 cost() 方法）
class MilkTea:
    def __init__(self, name, price):
        self._name = name
        self._price = price

class GreenTeaWithMilk(GreenTea):
    def cost(self): return 11
# ...继续写下去
```

**第二步：用装饰器，一层一层包**

    def cost(self):
        return self._price

    def description(self):
        return self._name

# 配料装饰器（每个配料都包一层）
class AddMilk:
    def __init__(self, tea):
        self._tea = tea           # 包住原来的奶茶

使用时，想加什么就包什么：

```python
# 红茶，什么都不加
tea = MilkTea("红茶", 10)
print(tea.description(), tea.cost())   # 红茶 10

    def cost(self):
        return self._tea.cost() + 3      # 原价 + 牛奶3元

    def description(self):
        return self._tea.description() + " + 牛奶"

class AddSugar:
    def __init__(self, tea):
        self._tea = tea

    def cost(self):
        return self._tea.cost() + 1

    def description(self):
        return self._tea.description() + " + 糖"

class AddPearl:
    def __init__(self, tea):
        self._tea = tea

    def cost(self):
        return self._tea.cost() + 2

    def description(self):
        return self._tea.description() + " + 珍珠"
```

# 红茶 + 牛奶
tea = MilkTea("红茶", 10)
tea = AddMilk(tea)
print(tea.description(), tea.cost())   # 红茶 + 牛奶 13

# 红茶 + 牛奶 + 糖 + 珍珠
tea = MilkTea("红茶", 10)
tea = AddMilk(tea)
tea = AddSugar(tea)
tea = AddPearl(tea)
print(tea.description(), tea.cost())   # 红茶 + 牛奶 + 糖 + 珍珠 16

# 绿茶 + 椰果
tea = MilkTea("绿茶", 8)
tea = AddPearl(tea)  # 椰果复用珍珠的价格逻辑，或新建 AddCoconut
print(tea.description(), tea.cost())   # 绿茶 + 珍珠 10
```

```python
# 新建一个类，不改任何已有代码
class AddTaro:
    def __init__(self, tea):
        self._tea = tea

```text
不用装饰器：每种组合写一个类 → 组合爆炸
用装饰器：基础对象 + 配料对象层层包装 → 想怎么组合就怎么组合

```text
你原来写的代码到处都是：
  payment = Alipay()
  payment.pay(“ORDER001”, 100)     → 返回 True/False

接口完全不同：
  支付宝：一个 pay() 方法搞定，返回布尔值
  微信：要先 create_order 拿到订单号，再 query 查状态，返回字符串
```

```python
# 支付宝SDK（原来用的，接口长这样）
class AlipaySDK:
    def pay(self, order_id, amount):
        print(f”支付宝：订单{order_id}，金额{amount}”)
        return True   # 返回布尔值

**新增配料"加芋圆（+3 元）"怎么办？**

    def cost(self):
        return self._tea.cost() + 3

    def description(self):
        return self._tea.description() + " + 芋圆"
```

**装饰器的本质**：

就像穿衣服：
  不用装饰器 = 为每种穿搭写一套衣服（短袖+短裤、短袖+长裤、长袖+短裤...）
  用装饰器 = 先穿T恤，冷了加外套，再冷加围巾，想脱就脱，自由组合
```

**一句话**：不改原对象代码，外面一层一层包新功能，自由组合，用装饰器。

---

#### 适配器模式（结构型）(就是把不同的方法，统一起来，用一个适配器，这个逻辑的方法调用都一样)

**场景**：你的系统对接了支付宝 SDK，整个业务代码都在用 `payment.pay(order_id, amount)` 这个接口。现在老板说要接入微信支付，但微信支付的接口和支付宝完全不同。

**第一步：直接换？改不动。**

现在要换成微信SDK，但微信SDK的接口是这样的：
  wechat = WechatPaySDK()
  wechat.create_order(100, “订单描述”)  → 返回订单号（字符串）
  wechat.query(“WX20230501001”)         → 返回 “SUCCESS” 或 “FAIL”

如果直接换，你要改所有调用 `payment.pay()` 的地方。几十个文件都要改，风险巨大。

**第二步：用适配器，做一个”翻译层”**

适配器做的事情就是：**把微信的接口”翻译”成和支付宝一样的接口，业务代码一行都不用改。**

# 微信SDK（新接入的，接口完全不同）
class WechatPaySDK:
    def create_order(self, amount, desc):
        print(f”微信：创建订单，金额{amount}，描述{desc}”)
        return “WX20230501001”   # 返回订单号

    def query(self, order_no):
        print(f”微信：查询订单{order_no}状态”)
        return “SUCCESS”   # 返回字符串

# 适配器：让微信SDK看起来像支付宝SDK
class WechatPayAdapter:
    def __init__(self):
        self.wechat = WechatPaySDK()   # 内部持有微信SDK

```python
# 原来的代码：
payment = AlipaySDK()
payment.pay(“ORDER001”, 100)

    def pay(self, order_id, amount):
        # 把支付宝风格的调用，翻译成微信风格的调用
        order_no = self.wechat.create_order(amount, f”订单{order_id}”)
        result = self.wechat.query(order_no)
        return result == “SUCCESS”     # 把微信的返回值转成布尔值，和支付宝一致
```

**第三步：业务代码一行不用改**

# 换成微信，只改一行：
payment = WechatPayAdapter()    # 只换了这一行
payment.pay(“ORDER001”, 100)    # 调用方式完全一样，返回值也一样
```

问题：

```text
就像充电器转接头：
  你的手机是 Type-C 接口
  充电线是 USB 接口
  直接插？插不进去
  加一个转接头 → 适配器
  手机不用改，充电线不用改，转接头负责转换

```python
class UserService:
    def __init__(self):
        self.cache = {}

```text
UserService 原本只负责查询，职责很纯粹。
现在混进了缓存逻辑，职责变复杂了。
如果以后还要加日志、加权限、加限流，这个类会越来越臃肿。
而且改了原函数，万一缓存逻辑有 bug，连查询都用不了了。
```

```python
# 原始服务：只负责查数据库，不关心其他事
class UserService:
    def get_user(self, user_id):
        print(f”查询数据库: user_id={user_id}”)
        time.sleep(1)   # 模拟慢查询
        return {“id”: user_id, “name”: “张三”}

**适配器的本质**：

代码里也一样：
  你的业务代码调 payment.pay()
  微信SDK提供 create_order() + query()
  直接换？改不动
  加一个 WechatPayAdapter → 适配器
  业务代码不用改，微信SDK不用改，适配器负责翻译
```

**一句话**：已有接口不能改，新接口又不一样，中间加一个”翻译层”，用适配器。

---

#### 代理模式（结构型）

**场景**：你有一个查询用户信息的函数，直接查数据库，很慢。你想加缓存：第一次查数据库，之后查缓存。但你不想改原来的函数代码（可能是别人写的，或者改了怕出 bug）。

**第一步：不用代理，直接改原函数**

    def get_user(self, user_id):
        if user_id in self.cache:
            return self.cache[user_id]     # 缓存命中
        print(f”查询数据库: user_id={user_id}”)
        time.sleep(1)
        result = {“id”: user_id, “name”: “张三”}
        self.cache[user_id] = result
        return result
```

**第二步：用代理，不改原代码，外面包一层**

# 代理：和原始服务有同样的接口（都有 get_user 方法）
class UserServiceProxy:
    def __init__(self, real_service):
        self._real_service = real_service   # 持有原始服务的引用
        self._cache = {}

使用时，把代理当作原始服务来用：

```python
# 创建代理，传入原始服务
service = UserServiceProxy(UserService())

    def get_user(self, user_id):
        # 第一道关：查缓存
        if user_id in self._cache:
            print(f”[代理] 缓存命中: user_id={user_id}”)
            return self._cache[user_id]

        # 缓存没有，才调用原始服务查数据库
        print(f”[代理] 缓存未命中，查询数据库”)
        result = self._real_service.get_user(user_id)
        self._cache[user_id] = result
        return result
```

# 调用方完全不知道背后有代理，接口一样
service.get_user(1)   # [代理] 缓存未命中，查询数据库（慢）
service.get_user(1)   # [代理] 缓存命中: user_id=1（快，不查数据库）
service.get_user(2)   # [代理] 缓存未命中，查询数据库（慢）
service.get_user(2)   # [代理] 缓存命中: user_id=2（快）
```

```python
# 权限代理：检查有没有权限
class AuthProxy:
    def __init__(self, service, current_user):
        self._service = service
        self._current_user = current_user

**代理的扩展：同一个原始服务，可以叠加多种代理**

    def get_user(self, user_id):
        if self._current_user[“role”] != “admin”:
            raise PermissionError(“只有管理员能查用户信息”)
        return self._service.get_user(user_id)

# 日志代理：记录每次调用
class LogProxy:
    def __init__(self, service):
        self._service = service

    def get_user(self, user_id):
        print(f”[日志] 调用 get_user({user_id})，时间: {datetime.now()}”)
        result = self._service.get_user(user_id)
        print(f”[日志] 返回: {result}”)
        return result

# 叠加使用：日志 → 权限 → 缓存 → 原始服务
service = UserService()                          # 原始
service = UserServiceProxy(service)              # 加缓存
service = AuthProxy(service, {“role”: “admin”})  # 加权限
service = LogProxy(service)                      # 加日志

问题：

```text
不用代理：所有逻辑堆在一个类里，改来改去
用代理：原始服务只管核心逻辑，代理在外面加辅助逻辑（缓存、权限、日志）

```python
class CsvImporter:
    def import_data(self, file_path):
        raw = open(file_path).read()                    # 第1步：读文件
        data = [line.split(“,”) for line in raw.split(“\n”)]  # 第2步：解析CSV
        if not data:                                     # 第3步：校验
            raise ValueError(“数据为空”)
        print(f”写入数据库: {len(data)} 条”)              # 第4步：入库

```text
第1、3、4步的代码完全一样，复制粘贴了三份。
如果要改校验逻辑（比如加一个”数据量不能超过10万条”的检查），
三个类都要改，改漏一个就出 bug。
```

```python
from abc import ABC, abstractmethod

```python
# CSV格式：解析方式是按逗号分割
class CsvImporter(DataImporter):
    def parse_data(self, raw):
        lines = raw.strip().split(“\n”)
        return [line.split(“,”) for line in lines]

service.get_user(1)  # 先记日志 → 检查权限 → 查缓存 → 查数据库
```

**代理的本质**：

就像明星和经纪人：
  明星只管唱歌演戏（核心业务）
  经纪人负责接活、排期、过滤不合理要求（代理逻辑）
  粉丝找明星，要先过经纪人这一关
```

**一句话**：不改原对象代码，在外面包一层做缓存、权限、日志等辅助逻辑，用代理。

---

#### 模板方法模式（行为型）

**场景**：你做一个数据导入系统，要支持导入 CSV、JSON、Excel 三种格式。导入流程都一样：读文件 → 解析数据 → 校验数据 → 写入数据库。三种格式只有”解析数据”这一步不同，其他三步完全一样。

**第一步：不用模板方法，每种格式写一套完整代码**

class JsonImporter:
    def import_data(self, file_path):
        raw = open(file_path).read()                    # 第1步：读文件（重复！）
        data = json.loads(raw)                           # 第2步：解析JSON
        if not data:                                     # 第3步：校验（重复！）
            raise ValueError(“数据为空”)
        print(f”写入数据库: {len(data)} 条”)              # 第4步：入库（重复！）
```

**第二步：用模板方法，把重复的步骤提到父类**

class DataImporter(ABC):
    # 模板方法：定义固定流程，子类不能改这个流程
    def import_data(self, file_path):
        raw = self.read_file(file_path)       # 第1步：读文件（固定，写在父类）
        data = self.parse_data(raw)           # 第2步：解析（可变，子类实现）
        self.validate(data)                   # 第3步：校验（固定，写在父类）
        self.save_to_db(data)                 # 第4步：入库（固定，写在父类）

    # 固定步骤：直接写在父类，所有子类共用
    def read_file(self, path):
        with open(path, 'r') as f:
            return f.read()

    def validate(self, data):
        if not data:
            raise ValueError(“数据为空”)
        if len(data) > 100000:
            raise ValueError(“数据量超过10万条，拒绝导入”)

    def save_to_db(self, data):
        print(f”写入数据库: {len(data)} 条”)

    # 可变步骤：抽象方法，子类必须实现
    @abstractmethod
    def parse_data(self, raw): pass
```

每种子类只实现”解析”这一步：

# JSON格式：解析方式是 json.loads
class JsonImporter(DataImporter):
    def parse_data(self, raw):
        return json.loads(raw)

# Excel格式：解析方式是用 openpyxl 读取
class ExcelImporter(DataImporter):
    def parse_data(self, raw):
        import openpyxl
        wb = openpyxl.load_workbook(raw)
        return [[cell.value for cell in row] for row in wb.active.rows]
```

使用：

```python
CsvImporter().import_data(“users.csv”)
JsonImporter().import_data(“users.json”)
ExcelImporter().import_data(“users.xlsx”)
```

```python
# 父类里改一下，所有子类自动生效
def validate(self, data):
    if not data:
        raise ValueError(“数据为空”)
    if len(data) > 500000:       # 从10万改成50万
        raise ValueError(“数据量过大”)
```

```text
不用模板方法：每个子类把所有步骤都写一遍 → 大量重复代码
用模板方法：固定步骤写在父类，可变步骤留给子类 → 重复代码只有一份

| 模式     | 类型   | 一句话记忆                     | 典型场景             |
| -------- | ------ | ------------------------------ | -------------------- |
| 单例     | 创建型 | 全局只要一个对象               | 配置管理器、连接池   |
| 工厂     | 创建型 | 封装 new，调用方不关心创建细节 | 支付方式选择、UI组件 |
| 策略     | 行为型 | 同一个动作，多种算法           | 优惠计算、排序方式   |
| 观察者   | 行为型 | 一个变了，自动通知别人         | 事件系统、消息推送   |
| 装饰器   | 结构型 | 不改原对象，外面加功能         | 奶茶加料、IO流包装   |
| 适配器   | 结构型 | 接口不兼容，中间转换           | 老系统对接、SDK适配  |
| 代理     | 结构型 | 不直接访问，通过中介           | 缓存、权限、日志     |
| 模板方法 | 行为型 | 流程固定，细节可变             | 数据导入、奶茶制作   |

**如果要改校验逻辑，只改父类一处**：

**模板方法的本质**：

就像餐厅出餐流程：
  所有菜都要：备料 → 烹饪 → 摆盘 → 上菜
  “备料、摆盘、上菜”每道菜都一样（父类）
  “烹饪”每道菜不同（子类实现）
  模板方法就是把固定流程定死，只让子类填空
```

**一句话**：多个类的流程一样，只有其中某几步不同，用模板方法把固定流程定死在父类，可变步骤交给子类填空。

---

#### 快速区分：这些模式到底有什么不同

最容易混淆的三组：

**工厂 vs 策略**：工厂管”怎么创建对象”，策略管”怎么切换行为”。工厂返回一个对象给你用，策略是传一个算法进去让别人用。

**装饰器 vs 代理 vs 适配器**：三者都是”包一层”，但目的不同——装饰器增强功能（加配料），代理控制访问（加缓存/权限），适配器转换接口（翻译）。

**策略 vs 模板方法**：策略是整个算法可以替换（传什么用什么），模板方法是流程骨架固定、只替换其中几步（子类只实现差异部分）。
