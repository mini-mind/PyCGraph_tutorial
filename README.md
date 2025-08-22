# PyCGraph 简明教程

[English](README_en.md) | 中文

> **源库地址**: [https://github.com/ChunelFeng/CGraph](https://github.com/ChunelFeng/CGraph)

## 什么是 CGraph？

CGraph 是一个高性能的**有向无环图（DAG）计算框架**，专门用于构建和执行复杂的数据处理流水线。它的 Python 绑定 PyCGraph 让 Python 开发者能够轻松构建高效的并行计算图。

### 为什么使用 CGraph？

- **高性能**: 基于 C++ 实现，支持多线程并行执行
- **易用性**: 简洁的 Python API，声明式编程风格
- **灵活性**: 支持复杂的依赖关系、条件执行、参数传递
- **可扩展**: 丰富的扩展机制（切面、事件、自定义节点）

### 适用场景

- 数据处理流水线
- 机器学习工作流
- 批处理任务调度
- 复杂业务逻辑编排

## 目录

1. [基础概念](#基础概念)
2. [Hello World](#hello-world)
3. [简单流水线](#简单流水线)
4. [集群节点](#集群节点)
5. [参数传递](#参数传递)
6. [条件执行](#条件执行)
7. [切面编程](#切面编程)
8. [事件处理](#事件处理)
9. [高级特性](#高级特性)

## 基础概念

### 核心组件详解

#### GNode（图节点）
- **作用**: 计算图中的基本执行单元，类似于函数或任务
- **特点**: 每个节点代表一个具体的计算或操作
- **生命周期**: `init()` → `run()` → `destroy()`
- **示例**: 数据读取节点、数据处理节点、结果输出节点

#### GPipeline（管道）
- **作用**: 整个计算图的管理器和执行引擎
- **功能**:
  - 注册节点和依赖关系
  - 调度节点执行顺序
  - 管理并行执行
  - 处理错误和状态
- **使用流程**: 创建 → 注册节点 → 初始化 → 运行 → 销毁

#### GCluster（集群）
- **作用**: 将多个节点组合成一个逻辑单元
- **优势**:
  - 简化复杂图的管理
  - 支持集群级别的并行控制
  - 可以作为一个整体添加切面或条件

#### GParam（参数）
- **作用**: 节点间共享数据的容器
- **特点**:
  - 线程安全（支持 lock/unlock）
  - 生命周期管理
  - 支持复杂数据类型

#### GCondition（条件）
- **作用**: 根据运行时条件选择执行不同的节点分支
- **应用**: 实现 if-else 逻辑、动态路由

#### GAspect（切面）
- **作用**: 在节点执行前后插入额外逻辑
- **应用**: 日志记录、性能监控、资源管理

#### GEvent（事件）
- **作用**: 异步事件处理机制
- **应用**: 状态通知、监控告警、解耦组件

### 状态管理

#### CStatus（状态对象）
- **作用**: 表示操作的执行状态
- **属性**:
  - 错误码（0 表示成功，负数表示错误）
  - 错误信息字符串
- **方法**:
  - `isErr()`: 检查是否有错误
  - `getCode()`: 获取错误码
  - `getInfo()`: 获取错误信息

## Hello World

让我们从最简单的例子开始，了解 CGraph 的基本使用方法。

### 第一步：创建自定义节点

首先，我们需要创建一个自定义的计算节点。所有的计算节点都必须继承自 `GNode` 类：

```python
from PyCGraph import GNode, CStatus

class HelloCGraphNode(GNode):
    def run(self):
        """
        run() 方法是节点的核心逻辑
        - 这里实现具体的计算或处理逻辑
        - 必须返回 CStatus 对象表示执行状态
        """
        print("Hello, PyCGraph.")
        return CStatus()  # 返回成功状态
```

**关键点说明：**
- `run()` 方法：节点的主要执行逻辑，类似于线程的 run 方法
- `CStatus()`：表示成功执行，等价于 `CStatus(0, "")`
- 如果出错，可以返回 `CStatus(-1, "错误信息")`

### 第二步：构建和执行管道

```python
from PyCGraph import GPipeline
from MyGNode.HelloCGraphNode import HelloCGraphNode

def tutorial_hello_cgraph():
    # 1. 创建管道对象
    pipeline = GPipeline()

    # 2. 注册节点到管道
    # registerGElement(节点实例, 依赖集合, 节点名称)
    pipeline.registerGElement(HelloCGraphNode())

    # 3. 一键执行：初始化 → 运行 → 销毁
    pipeline.process()

if __name__ == '__main__':
    tutorial_hello_cgraph()
```

**执行流程说明：**
1. **创建管道**：`GPipeline()` 创建一个空的执行管道
2. **注册节点**：将自定义节点添加到管道中
3. **执行管道**：`process()` 方法会自动完成初始化、运行、销毁的完整流程

**输出结果：**
```
Hello, PyCGraph.
```

### process() vs 手动控制

`process()` 是一个便捷方法，等价于：

```python
pipeline.init()    # 初始化所有节点
pipeline.run()     # 执行一次
pipeline.destroy() # 清理资源
```

对于需要多次运行或精细控制的场景，建议使用手动控制方式。

## 简单流水线

在实际应用中，我们通常需要构建具有复杂依赖关系的计算图。让我们学习如何创建一个包含多个节点和依赖关系的流水线。

### 依赖关系图

我们要构建的计算图结构如下：

```
    A (起始节点)
   / \
  B   C (并行执行)
   \ /
    D (汇聚节点)
```

这个图表示：
- A 节点首先执行（无依赖）
- B 和 C 节点等待 A 完成后并行执行
- D 节点等待 B 和 C 都完成后执行

### 节点实现示例

首先看看基础节点的实现：

```python
# MyNode1.py
from datetime import datetime
import time
from PyCGraph import GNode, CStatus

class MyNode1(GNode):
    def run(self):
        print(f"[{datetime.now()}] {self.getName()}, 进入 MyNode1 执行. 休眠 1 秒...")
        time.sleep(1)  # 模拟计算耗时
        return CStatus()

# MyNode2.py
class MyNode2(GNode):
    def run(self):
        print(f"[{datetime.now()}] {self.getName()}, 进入 MyNode2 执行. 休眠 1 秒...")
        time.sleep(1)  # 模拟计算耗时
        return CStatus()
```

### 构建流水线

```python
from PyCGraph import GPipeline, CStatus
from MyGNode.MyNode1 import MyNode1
from MyGNode.MyNode2 import MyNode2

def tutorial_simple():
    # 1. 创建管道和节点实例
    pipeline = GPipeline()
    a, b, c, d = MyNode1(), MyNode2(), MyNode1(), MyNode2()

    # 2. 注册节点及其依赖关系
    # registerGElement(节点, 依赖集合, 节点名称, 循环次数)
    pipeline.registerGElement(a, set(), "nodeA")        # a 无依赖，首先执行
    pipeline.registerGElement(b, {a}, "nodeB")          # b 依赖 a，等 a 完成后执行
    pipeline.registerGElement(c, {a}, "nodeC")          # c 依赖 a，与 b 并行执行
    pipeline.registerGElement(d, {b, c}, "nodeD")       # d 依赖 b 和 c，等两者都完成

    # 3. 手动控制执行流程（推荐用于生产环境）
    status = pipeline.init()
    if status.isErr():
        print(f'管道初始化失败: 错误码={status.getCode()}, 错误信息={status.getInfo()}')
        return

    # 4. 多次运行同一个管道
    for i in range(3):
        print(f"\n=== 第 {i+1} 次执行开始 ===")
        status = pipeline.run()
        print(f"=== 第 {i+1} 次执行完成，状态码: {status.getCode()} ===")

    # 5. 清理资源
    pipeline.destroy()
```

### 关键概念解释

#### 依赖集合
- `set()`：空集合，表示无依赖
- `{a}`：依赖节点 a
- `{b, c}`：同时依赖节点 b 和 c（必须都完成）

#### 执行顺序
CGraph 会自动根据依赖关系确定执行顺序：
1. 首先执行无依赖的节点（A）
2. 然后执行依赖已完成节点的节点（B、C 并行）
3. 最后执行所有依赖都满足的节点（D）

#### 错误处理
- 始终检查 `status.isErr()` 来判断是否有错误
- 使用 `status.getCode()` 和 `status.getInfo()` 获取详细错误信息
- 错误码 0 表示成功，负数表示错误

#### 多次运行
- `init()` 只需调用一次，用于初始化所有节点
- `run()` 可以多次调用，每次都会完整执行一遍流水线
- `destroy()` 在最后调用，清理所有资源

### 预期输出

```text
=== 第 1 次执行开始 ===
[2025-01-01 10:00:00] nodeA, 进入 MyNode1 执行. 休眠 1 秒...
[2025-01-01 10:00:01] nodeB, 进入 MyNode2 执行. 休眠 1 秒...
[2025-01-01 10:00:01] nodeC, 进入 MyNode1 执行. 休眠 1 秒...
[2025-01-01 10:00:02] nodeD, 进入 MyNode2 执行. 休眠 1 秒...
=== 第 1 次执行完成，状态码: 0 ===
```

注意 B 和 C 几乎同时开始执行，体现了并行特性。

## 集群节点

当我们需要将多个相关的节点组合在一起作为一个逻辑单元时，可以使用 `GCluster`（集群）。集群可以简化复杂图的管理，并提供统一的控制接口。

**注意**：GCluster 是 GRegion 的弱化版本，内部将节点按照添加顺序依次执行。如果需要更复杂的节点组合逻辑，建议优先使用 GRegion。

### 什么是集群？

集群是将多个节点打包成一个逻辑单元的机制：
- **逻辑分组**：相关节点的逻辑组合
- **统一管理**：可以对整个集群设置并行度、添加切面等
- **简化依赖**：其他节点可以依赖整个集群，而不需要依赖集群内的每个节点

### 集群的执行模式

集群内的节点执行方式：
- **顺序执行**：集群内的节点按照添加顺序依次执行
- **无依赖关系**：集群内节点之间没有依赖关系
- **整体完成**：只有当集群内所有节点都完成时，集群才算完成

### 示例：构建集群流水线

我们要构建的计算图结构：

```text
    A (单节点)
   / \
  B   C (B是集群，包含3个节点；C是单节点)
   \ /
    D (单节点)
```

```python
from PyCGraph import GPipeline, GCluster
from MyGNode.MyNode1 import MyNode1
from MyGNode.MyNode2 import MyNode2

def tutorial_cluster():
    # 1. 创建集群内的节点
    b1 = MyNode1("nodeB1")           # 集群内节点1
    b2 = MyNode1("nodeB2")           # 集群内节点2
    b3 = MyNode2("nodeB3")           # 集群内节点3

    # 2. 创建集群（将多个节点组合）
    b_cluster = GCluster([b1, b2, b3])  # 注意：传入的是节点列表

    # 3. 创建其他独立节点
    pipeline = GPipeline()
    a = MyNode1()                    # 前置节点
    c = MyNode2()                    # 并行节点
    d = MyNode1()                    # 后置节点

    # 4. 注册节点和集群到管道
    pipeline.registerGElement(a, set(), "nodeA")
    pipeline.registerGElement(b_cluster, {a}, "clusterB", 2)  # 集群循环执行2次
    pipeline.registerGElement(c, {a}, "nodeC")
    pipeline.registerGElement(d, {b_cluster, c}, "nodeD", 2)

    # 5. 执行
    pipeline.process()
```

### 关键参数说明

#### 集群创建
```python
b_cluster = GCluster([b1, b2, b3])
```
- 传入节点列表，不是依赖关系
- 列表中的节点会被组合成一个逻辑单元

#### 循环次数控制
```python
pipeline.registerGElement(b_cluster, {a}, "clusterB", 2)
```
- 第4个参数 `2` 表示集群的循环执行次数
- 意味着整个集群会执行2次（每次都是顺序执行集群内的所有节点）
- 默认值为1，表示只执行一次

### 执行流程分析

1. **A 节点执行**：首先执行，无依赖
2. **B 集群和 C 节点并行执行**：
   - B 集群内的 b1, b2, b3 按顺序执行（执行2次）
   - C 节点同时开始执行
3. **D 节点执行**：等待 B 集群和 C 节点都完成

### 集群的优势

#### 1. 简化依赖管理
```python
# 不使用集群：需要依赖集群内的每个节点
pipeline.registerGElement(d, {b1, b2, b3, c}, "nodeD")

# 使用集群：只需依赖集群本身
pipeline.registerGElement(d, {b_cluster, c}, "nodeD")
```

#### 2. 统一控制
```python
# 可以对整个集群设置属性
b_cluster.addGAspect(MyTimerAspect())  # 为整个集群添加切面
```

#### 3. 逻辑清晰
- 将相关功能的节点组合在一起
- 提高代码的可读性和维护性

### 预期输出

```text
[时间戳] nodeA, 进入 MyNode1 执行...
[时间戳] nodeB1, 进入 MyNode1 执行...  # 集群内节点按顺序执行
[时间戳] nodeB2, 进入 MyNode1 执行...  # 顺序执行
[时间戳] nodeB3, 进入 MyNode2 执行...  # 顺序执行
[时间戳] nodeB1, 进入 MyNode1 执行...  # 第2次循环开始
[时间戳] nodeB2, 进入 MyNode1 执行...  # 第2次循环
[时间戳] nodeB3, 进入 MyNode2 执行...  # 第2次循环
[时间戳] nodeC, 进入 MyNode2 执行...   # 与集群并行
[时间戳] nodeD, 进入 MyNode1 执行...   # 等集群和C都完成
```

## 参数传递

在复杂的计算图中，节点之间经常需要共享数据。CGraph 提供了 `GParam` 机制来实现线程安全的参数传递和共享。

### 参数传递的核心概念

#### 为什么需要参数传递？
- **数据共享**：多个节点需要访问相同的数据
- **状态保持**：在多次运行之间保持状态
- **线程安全**：在并行执行环境中安全地共享数据
- **生命周期管理**：自动管理参数的创建和销毁

#### 参数的生命周期
1. **创建**：在某个节点的 `init()` 方法中创建
2. **共享**：其他节点通过参数名访问
3. **使用**：在节点的 `run()` 方法中读写
4. **销毁**：管道销毁时自动清理

### 第一步：定义参数类

所有的参数类都必须继承自 `GParam`：

```python
from PyCGraph import GParam, CStatus

class MyParam(GParam):
    """
    自定义参数类，用于在节点间共享数据
    """
    def __init__(self):
        super().__init__()
        self.value = 0      # 业务数据1
        self.count = 0      # 业务数据2
        self.data = []      # 可以包含任意复杂的数据结构

    def reset(self, curStatus: CStatus):
        """
        重置方法：在每次管道运行前调用
        用于清理或重置参数状态
        """
        self.value = 0
        self.data.clear()
        # 注意：通常不重置 count，因为它可能需要跨运行保持
```

**关键点说明：**
- 继承 `GParam` 类
- 可以包含任意类型的数据成员
- `reset()` 方法用于在每次运行前重置状态

### 第二步：创建参数的节点（写节点）

```python
from PyCGraph import GNode, CStatus
from MyParams.MyParam import MyParam

class MyWriteParamNode(GNode):
    """
    负责创建和写入参数的节点
    """
    def init(self):
        """
        在初始化阶段创建参数
        createGParam(参数实例, 参数名称)
        """
        return self.createGParam(MyParam(), "param1")

    def run(self):
        """
        在运行阶段修改参数
        """
        # 1. 获取参数对象
        param: MyParam = self.getGParam("param1")

        # 2. 加锁（确保线程安全）
        param.lock()

        try:
            # 3. 修改参数数据
            param.count += 1
            param.value += 10
            param.data.append(f"数据_{param.count}")

            print(f'[{self.getName()}] 写入数据 - value: {param.value}, count: {param.count}')
            print(f'[{self.getName()}] 数据列表: {param.data}')

        finally:
            # 4. 解锁（确保即使出错也能解锁）
            param.unlock()

        return CStatus()
```

### 第三步：读取参数的节点（读节点）

```python
class MyReadParamNode(GNode):
    """
    负责读取参数的节点
    """
    def run(self):
        """
        读取共享参数的数据
        """
        # 1. 获取参数对象（通过参数名）
        param: MyParam = self.getGParam("param1")

        # 2. 加锁（读操作也需要加锁）
        param.lock()

        try:
            # 3. 读取参数数据
            current_value = param.value
            current_count = param.count
            data_copy = param.data.copy()  # 复制数据避免外部修改

            print(f'[{self.getName()}] 读取数据 - value: {current_value}, count: {current_count}')
            print(f'[{self.getName()}] 数据列表长度: {len(data_copy)}')

        finally:
            # 4. 解锁
            param.unlock()

        return CStatus()
```

### 第四步：构建参数传递流水线

```python
from PyCGraph import GPipeline
from MyGNode.MyWriteParamNode import MyWriteParamNode
from MyGNode.MyReadParamNode import MyReadParamNode

def tutorial_param():
    """
    演示参数传递的完整流水线
    """
    pipeline = GPipeline()

    # 创建节点实例
    write_node = MyWriteParamNode()  # 创建参数的节点
    read_node1 = MyReadParamNode()   # 读取参数的节点1
    read_node2 = MyReadParamNode()   # 读取参数的节点2
    write_node2 = MyWriteParamNode() # 再次写入的节点

    # 注册节点和依赖关系
    pipeline.registerGElement(write_node, set(), "writeNodeA")      # 首先创建参数
    pipeline.registerGElement(read_node1, {write_node}, "readNodeB") # 读取参数
    pipeline.registerGElement(read_node2, {write_node}, "readNodeC") # 并行读取
    pipeline.registerGElement(write_node2, {read_node1, read_node2}, "writeNodeD") # 再次修改

    # 执行流水线
    pipeline.init()

    for i in range(3):
        print(f"\n=== 第 {i+1} 次运行 ===")
        status = pipeline.run()
        print(f"运行状态: {status.getCode()}")

    pipeline.destroy()
```

### 线程安全的重要性

#### 为什么需要加锁？
```python
# 错误示例：不加锁的并发访问
param.value += 1  # 在并行环境中可能导致数据竞争

# 正确示例：加锁保护
param.lock()
param.value += 1  # 线程安全的操作
param.unlock()
```

#### 最佳实践
```python
# 推荐使用 try-finally 确保解锁
param.lock()
try:
    # 对参数的所有操作
    param.value += 1
    param.data.append("new_item")
finally:
    param.unlock()  # 确保即使出错也能解锁
```

## 条件执行

在实际应用中，我们经常需要根据运行时的条件来决定执行哪个分支。CGraph 提供了 `GCondition` 机制来实现类似 if-else 的条件控制。

### 条件执行的核心概念

#### 什么是条件执行？
条件执行允许在运行时根据特定条件选择执行不同的节点分支：
- **动态选择**：根据运行时状态决定执行路径
- **分支控制**：实现类似 if-else、switch-case 的逻辑
- **资源优化**：只执行必要的分支，节省计算资源

#### 条件执行的工作原理
1. **候选节点**：定义多个可能执行的节点
2. **选择逻辑**：实现 `choose()` 方法返回要执行的节点索引
3. **动态执行**：根据选择结果只执行对应的节点

### 第一步：简单条件实现

```python
from PyCGraph import GCondition

class MyCondition(GCondition):
    """
    简单的条件选择器
    """
    def choose(self):
        """
        选择要执行的节点
        返回值：节点在候选列表中的索引（从0开始）
        """
        # 这里可以实现任意的选择逻辑
        import random
        choice = random.randint(0, 2)  # 随机选择0、1、2中的一个
        print(f"[MyCondition] 随机选择执行节点索引: {choice}")
        return choice
```

### 第二步：基于参数的条件

```python
class MyParamCondition(GCondition):
    """
    基于共享参数的条件选择器
    """
    def choose(self):
        """
        根据共享参数的值来选择执行分支
        """
        # 获取共享参数
        param: MyParam = self.getGParam("param1")

        param.lock()
        try:
            current_value = param.value

            # 根据参数值选择不同的执行分支
            if current_value < 10:
                choice = 0  # 执行第一个节点
                reason = f"value({current_value}) < 10，执行分支0"
            elif current_value < 20:
                choice = 1  # 执行第二个节点
                reason = f"10 <= value({current_value}) < 20，执行分支1"
            else:
                choice = 2  # 执行第三个节点
                reason = f"value({current_value}) >= 20，执行分支2"

            print(f"[MyParamCondition] {reason}")
            return choice

        finally:
            param.unlock()
```

### 第三步：创建条件节点

```python
# 为不同分支创建不同的节点
class ConditionNodeA(GNode):
    def run(self):
        print(f"[{self.getName()}] 执行分支A - 处理小数值情况")
        return CStatus()

class ConditionNodeB(GNode):
    def run(self):
        print(f"[{self.getName()}] 执行分支B - 处理中等数值情况")
        return CStatus()

class ConditionNodeC(GNode):
    def run(self):
        print(f"[{self.getName()}] 执行分支C - 处理大数值情况")
        return CStatus()
```

### 第四步：构建条件执行流水线

```python
from PyCGraph import GPipeline
from MyGNode.MyWriteParamNode import MyWriteParamNode
from MyGNode.MyReadParamNode import MyReadParamNode
from MyGCondition.MyCondition import MyCondition
from MyGCondition.MyParamCondition import MyParamCondition

def tutorial_condition():
    """
    演示条件执行的完整流水线
    """
    # 1. 创建条件分支的候选节点
    b0 = ConditionNodeA("conditionNodeB0")  # 分支0
    b1 = ConditionNodeB("conditionNodeB1")  # 分支1
    b2 = ConditionNodeC("conditionNodeB2")  # 分支2

    # 2. 创建条件选择器（传入候选节点列表）
    b_condition = MyCondition([b0, b1, b2])

    # 3. 创建另一组条件分支
    d0 = MyNode1("conditionNodeD0")
    d1 = MyNode1("conditionNodeD1")
    d2 = MyNode1("conditionNodeD2")

    # 4. 创建基于参数的条件选择器
    d_condition = MyParamCondition([d0, d1, d2])

    # 5. 构建完整流水线
    pipeline = GPipeline()
    write_node = MyWriteParamNode()  # 创建和修改参数
    read_node = MyReadParamNode()    # 读取参数

    # 6. 注册节点和依赖关系
    pipeline.registerGElement(write_node, set(), "writeNodeA")           # 首先写入参数
    pipeline.registerGElement(b_condition, {write_node}, "conditionB")   # 基于参数选择分支
    pipeline.registerGElement(read_node, {b_condition}, "readNodeC")     # 读取参数状态
    pipeline.registerGElement(d_condition, {read_node}, "conditionD")    # 再次条件选择

    # 7. 执行多次观察不同的选择结果
    pipeline.init()
    for i in range(5):
        print(f"\n=== 第 {i+1} 次运行 ===")
        status = pipeline.run()
        print(f"运行状态: {status.getCode()}")

    pipeline.destroy()
```

### 条件执行的高级特性

#### 1. 条件嵌套
```python
# 可以在条件分支内再嵌套条件
inner_condition = MyCondition([node1, node2])
outer_condition = MyCondition([inner_condition, node3, node4])
```

#### 2. 动态条件
```python
class DynamicCondition(GCondition):
    def choose(self):
        # 可以基于外部状态、时间、随机数等做选择
        import datetime
        hour = datetime.datetime.now().hour
        if hour < 12:
            return 0  # 上午执行分支0
        else:
            return 1  # 下午执行分支1
```

#### 3. 条件与集群结合
```python
# 条件的候选项也可以是集群
cluster1 = GCluster([node1, node2])
cluster2 = GCluster([node3, node4])
condition = MyCondition([cluster1, cluster2])
```

### 使用场景示例

#### 1. 数据处理分支
```python
class DataProcessCondition(GCondition):
    def choose(self):
        data_size = self.getGParam("data_info").size
        if data_size < 1000:
            return 0  # 小数据集处理
        elif data_size < 10000:
            return 1  # 中等数据集处理
        else:
            return 2  # 大数据集处理
```

#### 2. 错误处理分支
```python
class ErrorHandlingCondition(GCondition):
    def choose(self):
        error_count = self.getGParam("error_info").count
        if error_count == 0:
            return 0  # 正常处理流程
        elif error_count < 5:
            return 1  # 轻微错误处理
        else:
            return 2  # 严重错误处理
```

### 注意事项

1. **索引范围**：`choose()` 返回的索引必须在候选节点列表范围内
2. **线程安全**：在 `choose()` 方法中访问共享参数时要加锁
3. **性能考虑**：条件选择逻辑应该尽量简单快速
4. **调试友好**：建议在选择时打印日志，便于调试和监控

## 切面编程

切面编程（AOP）是一种编程范式，允许在不修改原有代码的情况下，在程序执行的特定点插入额外的逻辑。CGraph 的 `GAspect` 提供了强大的切面支持。

### 切面编程的应用场景

- **性能监控**：统计节点执行时间
- **日志记录**：记录节点的执行状态
- **资源管理**：在执行前后管理资源
- **错误处理**：统一的异常处理逻辑
- **权限检查**：执行前的权限验证

### 创建定时器切面

```python
from PyCGraph import GAspect, CStatus
import time

class MyTimerAspect(GAspect):
    """
    定时器切面：监控节点执行时间
    """
    _start_time = None
    
    def beginRun(self):
        """
        在节点执行前调用
        """
        self._start_time = time.time()
        print(f"[TimerAspect] {self.getName()} 开始执行...")
        return CStatus()

    def finishRun(self, curStatus: CStatus):
        """
        在节点执行后调用
        """
        span = time.time() - self._start_time
        print(f"[TimerAspect] {self.getName()} 执行完成，耗时: {span:.3f}秒")
        return
```

### 创建日志切面

```python
class MyLogAspect(GAspect):
    """
    日志切面：记录详细的执行信息
    """
    def beginRun(self):
        print(f"[LogAspect] === {self.getName()} 准备执行 ===")
        print(f"[LogAspect] 当前时间: {time.strftime('%Y-%m-%d %H:%M:%S')}")
        return CStatus()

    def finishRun(self, curStatus: CStatus):
        print(f"[LogAspect] === {self.getName()} 执行结束 ===")
        return
```

### 使用切面

```python
from PyCGraph import GPipeline, GCluster
from MyGNode.MyNode1 import MyNode1
from MyGNode.MyNode2 import MyNode2
from MyGAspect.MyTimerAspect import MyTimerAspect
from MyGAspect.MyLogAspect import MyLogAspect

def tutorial_aspect():
    """
    演示切面编程的使用
    """
    # 1. 创建节点和集群
    a = MyNode1()
    b1, b2 = MyNode2('nodeB1'), MyNode1('nodeB2')
    b_cluster = GCluster([b1, b2])
    c = MyNode2()

    # 2. 为节点添加切面
    a.addGAspect(MyTimerAspect())           # 为单个节点添加定时器切面
    a.addGAspect(MyLogAspect())             # 同一个节点可以添加多个切面

    b_cluster.addGAspect(MyTimerAspect())   # 为整个集群添加切面

    c.addGAspect(MyTimerAspect())           # 为另一个节点添加切面

    # 3. 构建和执行管道
    pipeline = GPipeline()
    pipeline.registerGElement(a, set(), 'nodeA')
    pipeline.registerGElement(b_cluster, {a}, 'clusterB')
    pipeline.registerGElement(c, {b_cluster}, 'nodeC')

    pipeline.process()
```

### 切面的执行顺序

当一个节点有多个切面时，执行顺序为：
1. 按添加顺序执行所有切面的 `beginRun()`
2. 执行节点的 `run()` 方法
3. 按相反顺序执行所有切面的 `finishRun()`

```python
# 添加顺序：Timer -> Log
node.addGAspect(MyTimerAspect())
node.addGAspect(MyLogAspect())

# 执行顺序：
# 1. MyTimerAspect.beginRun()
# 2. MyLogAspect.beginRun()
# 3. node.run()
# 4. MyLogAspect.finishRun()
# 5. MyTimerAspect.finishRun()
```

## 事件处理

CGraph 提供了强大的事件机制，允许节点在执行过程中触发异步事件，实现松耦合的组件通信。

### 事件机制的核心概念

#### 事件的作用
- **解耦通信**：节点间的松耦合通信
- **异步处理**：事件处理不阻塞主流程
- **状态通知**：通知系统状态变化
- **监控告警**：实现监控和告警机制

#### 事件的工作流程
1. **定义事件**：创建继承自 `GEvent` 的事件类
2. **注册事件**：将事件注册到管道中
3. **触发事件**：在节点中触发事件
4. **处理事件**：事件处理器异步执行

### 第一步：定义事件处理器

```python
from PyCGraph import GEvent, CStatus

class MyPrintEvent(GEvent):
    """
    简单的打印事件处理器
    """
    def trigger(self, _):
        """
        事件触发时的处理逻辑
        """
        print(f"[PrintEvent] 收到事件")
        return CStatus()

class MyLogEvent(GEvent):
    """
    日志记录事件处理器
    """
    def trigger(self, _):
        import datetime
        timestamp = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        print(f"[LogEvent] {timestamp} - 事件记录")
        return CStatus()
```

### 第二步：创建事件触发节点

```python
from PyCGraph import GNode, CStatus

class MyEventNode(GNode):
    """
    能够触发事件的节点
    """
    def run(self):
        node_name = self.getName()
        print(f"[{node_name}] 开始执行，准备触发事件...")

        # 触发事件 - 使用正确的API
        from PyCGraph import GEventType
        self.notify("print_event", GEventType.SYNC)
        self.notify("log_event", GEventType.SYNC)

        # 根据条件触发不同事件
        import random
        if random.random() > 0.5:
            self.notify("print_event", GEventType.SYNC)

        print(f"[{node_name}] 执行完成")
        return CStatus()
```

### 第三步：构建事件驱动的流水线

```python
from PyCGraph import GPipeline
from MyGNode.MyNode1 import MyNode1
from MyGNode.MyEventNode import MyEventNode
from MyGNode.MyWriteParamNode import MyWriteParamNode
from MyGEvent.MyPrintEvent import MyPrintEvent
from MyGEvent.MyLogEvent import MyLogEvent

def tutorial_event():
    """
    演示事件处理的完整流水线
    """
    # 1. 创建节点
    pipeline = GPipeline()
    write_node = MyWriteParamNode()     # 参数写入节点
    event_node1 = MyEventNode()         # 事件触发节点1
    normal_node = MyNode1()             # 普通节点
    event_node2 = MyEventNode()         # 事件触发节点2

    # 2. 注册节点和依赖关系
    pipeline.registerGElement(write_node, set(), "writeNodeA")
    pipeline.registerGElement(event_node1, {write_node}, "eventNodeB")
    pipeline.registerGElement(normal_node, {event_node1}, "normalNodeC")
    pipeline.registerGElement(event_node2, {normal_node}, "eventNodeD")

    # 3. 注册事件处理器
    pipeline.addGEvent(MyPrintEvent(), "print_event")  # 注册打印事件
    pipeline.addGEvent(MyLogEvent(), "log_event")      # 注册日志事件

    # 4. 执行流水线
    print("=== 开始执行事件驱动的流水线 ===")
    pipeline.process()
    print("=== 流水线执行完成 ===")
```

### 事件的高级特性

#### 1. 事件参数传递
```python
# 事件触发时不传递参数，使用notify方法
from PyCGraph import GEventType
self.notify("my_event", GEventType.SYNC)
```

#### 2. 条件事件触发
```python
class ConditionalEventNode(GNode):
    def run(self):
        param = self.getGParam("shared_param")
        param.lock()
        try:
            from PyCGraph import GEventType
            if param.value > 10:
                self.notify("high_value_event", GEventType.SYNC)
            else:
                self.notify("low_value_event", GEventType.SYNC)
        finally:
            param.unlock()
        return CStatus()
```

#### 3. 多事件处理器
```python
# 同一个事件名可以注册多个处理器
pipeline.addGEvent(MyPrintEvent(), "status_event")
pipeline.addGEvent(MyLogEvent(), "status_event")
pipeline.addGEvent(MyEmailEvent(), "status_event")  # 都会被触发
```

### 事件 vs 参数的选择

#### 使用事件的场景：
- 状态通知和监控
- 异步处理不影响主流程
- 松耦合的组件通信
- 日志记录和告警

#### 使用参数的场景：
- 节点间的数据传递
- 状态共享和同步
- 需要数据持久化
- 需要线程安全的数据访问

### 注意事项

1. **异步执行**：事件处理是异步的，不会阻塞主流程
2. **错误隔离**：事件处理的错误不会影响主流程
3. **性能考虑**：频繁的事件触发可能影响性能
4. **调试困难**：异步事件可能增加调试复杂度

## 高级特性

### 多管道并行
```python
def tutorial_multi_pipeline():
    """
    创建多个独立的管道并行执行
    """
    pipeline1 = GPipeline()
    pipeline2 = GPipeline()

    # 分别配置不同的管道
    # pipeline1 处理数据流A
    # pipeline2 处理数据流B

    # 可以并行执行
    import threading

    def run_pipeline1():
        pipeline1.process()

    def run_pipeline2():
        pipeline2.process()

    thread1 = threading.Thread(target=run_pipeline1)
    thread2 = threading.Thread(target=run_pipeline2)

    thread1.start()
    thread2.start()

    thread1.join()
    thread2.join()
```

### 超时控制
```python
def tutorial_timeout():
    """
    设置节点执行超时
    """
    pipeline = GPipeline()
    slow_node = SlowNode()
    pipeline.registerGElement(slow_node, set(), "slowNode")
    
    # 设置节点的超时时间（毫秒）
    slow_node.setTimeout(5000)  # 5秒超时

    # 如果节点执行超过5秒，会自动终止
    pipeline.process()
```

### 动态节点创建
```python
class DynamicNodeFactory:
    """
    动态创建节点的工厂类
    """
    @staticmethod
    def create_processing_nodes(data_types):
        nodes = []
        for data_type in data_types:
            if data_type == "text":
                nodes.append(TextProcessingNode())
            elif data_type == "image":
                nodes.append(ImageProcessingNode())
            elif data_type == "video":
                nodes.append(VideoProcessingNode())
        return nodes
```

## 最佳实践

### 1. 错误处理和状态检查

```python
def robust_pipeline_execution():
    """
    健壮的管道执行模式
    """
    pipeline = GPipeline()
    # ... 注册节点

    # 初始化检查
    status = pipeline.init()
    if status.isErr():
        print(f"初始化失败: {status.getCode()} - {status.getInfo()}")
        return False

    try:
        # 执行检查
        status = pipeline.run()
        if status.isErr():
            print(f"执行失败: {status.getCode()} - {status.getInfo()}")
            return False

        print("执行成功")
        return True

    finally:
        # 确保资源清理
        pipeline.destroy()
```

### 2. 参数的线程安全使用

```python
class SafeParamNode(GNode):
    """
    线程安全的参数使用示例
    """
    def run(self):
        param = self.getGParam("shared_param")

        # 推荐：使用 try-finally 确保解锁
        param.lock()
        try:
            # 所有对参数的操作都在锁保护下
            old_value = param.value
            param.value = old_value + 1
            param.data.append(f"update_{param.value}")

        finally:
            param.unlock()  # 确保解锁

        return CStatus()
```

### 3. 性能优化策略

```python
def optimized_pipeline():
    """
    性能优化的管道配置
    """
    pipeline = GPipeline()

    # 1. 合理设置并行度
    cpu_intensive_node = MyNode1()
    io_intensive_cluster = GCluster([IONode1(), IONode2(), IONode3()])

    # CPU密集型节点：并行度 = CPU核心数
    pipeline.registerGElement(cpu_intensive_node, set(), "cpu_node", 4)

    # IO密集型集群：可以设置更高的并行度
    pipeline.registerGElement(io_intensive_cluster, {cpu_intensive_node}, "io_cluster", 8)

    # 2. 使用切面监控性能
    cpu_intensive_node.addGAspect(MyTimerAspect())
    io_intensive_cluster.addGAspect(MyTimerAspect())

    pipeline.process()
```

### 4. 调试和监控

```python
class DebugAspect(GAspect):
    """
    调试切面：详细记录执行信息
    """
    def beforeRun(self):
        node_name = self.getBelongName()
        print(f"[DEBUG] {node_name} 开始执行")
        print(f"[DEBUG] 当前线程: {threading.current_thread().name}")
        return CStatus()

    def afterRun(self):
        node_name = self.getBelongName()
        print(f"[DEBUG] {node_name} 执行完成")
        return CStatus()

def debug_pipeline():
    """
    调试模式的管道
    """
    pipeline = GPipeline()
    # ... 创建节点

    # 为所有节点添加调试切面
    for node in [a, b, c, d]:
        node.addGAspect(DebugAspect())

    pipeline.process()
```

### 5. 配置管理

```python
class PipelineConfig:
    """
    管道配置管理
    """
    def __init__(self, config_file=None):
        self.config = {
            "timeout": 10000,
            "max_parallel": 4,
            "enable_debug": False,
            "enable_monitoring": True
        }

        if config_file:
            self.load_config(config_file)

    def load_config(self, config_file):
        # 从文件加载配置
        pass

    def create_pipeline(self):
        pipeline = GPipeline()
        pipeline.setTimeout(self.config["timeout"])
        return pipeline
```

### 6. 常见陷阱和解决方案

#### 陷阱1：忘记解锁参数
```python
# 错误：可能导致死锁
def bad_example():
    param.lock()
    if some_condition:
        return CStatus(-1, "error")  # 忘记解锁！
    param.unlock()

# 正确：使用 try-finally
def good_example():
    param.lock()
    try:
        if some_condition:
            return CStatus(-1, "error")
    finally:
        param.unlock()
```

#### 陷阱2：循环依赖
```python
# 错误：会导致死锁
pipeline.registerGElement(a, {b}, "nodeA")
pipeline.registerGElement(b, {a}, "nodeB")  # 循环依赖！

# 正确：确保依赖关系是有向无环图
pipeline.registerGElement(a, set(), "nodeA")
pipeline.registerGElement(b, {a}, "nodeB")
```

#### 陷阱3：条件选择越界
```python
# 错误：可能返回超出范围的索引
class BadCondition(GCondition):
    def choose(self):
        return 5  # 如果候选节点只有3个，会出错！

# 正确：确保返回值在有效范围内
class GoodCondition(GCondition):
    def choose(self):
        candidate_count = len(self.getGElementList())
        return random.randint(0, candidate_count - 1)
```

## 总结

CGraph Python 是一个功能强大的计算图框架，提供了：

### 核心优势
- **高性能**：基于 C++ 的高效执行引擎
- **易用性**：简洁直观的 Python API
- **灵活性**：支持复杂的依赖关系和执行模式
- **可扩展性**：丰富的扩展机制（切面、事件、自定义节点）
- **线程安全**：内置的并发控制和数据保护

### 主要特性
1. **节点和管道**：构建计算图的基础组件
2. **依赖管理**：自动的拓扑排序和执行调度
3. **并行执行**：高效的多线程并行处理
4. **参数传递**：线程安全的数据共享机制
5. **条件控制**：灵活的分支执行逻辑
6. **切面编程**：横切关注点的统一处理
7. **事件机制**：松耦合的异步通信

### 适用场景
- **数据处理流水线**：ETL、数据清洗、特征工程
- **机器学习工作流**：模型训练、验证、推理流程
- **批处理系统**：大规模数据批处理任务
- **业务流程编排**：复杂业务逻辑的自动化执行
- **微服务协调**：服务间的依赖管理和调度

### 学习建议
1. **从简单开始**：先掌握基本的节点和管道概念
2. **逐步深入**：依次学习参数、条件、切面、事件等高级特性
3. **实践为主**：通过实际项目加深理解
4. **关注性能**：学会使用监控和调试工具
5. **遵循最佳实践**：避免常见陷阱，编写健壮的代码

通过掌握这些概念和技巧，您可以使用 CGraph Python 构建高效、可靠、可维护的计算图应用。
