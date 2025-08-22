# PyCGraph Concise Tutorial

English | [中文](README.md)

> **Source Repository**: [https://github.com/ChunelFeng/CGraph](https://github.com/ChunelFeng/CGraph)

## What is CGraph?

CGraph is a high-performance **Directed Acyclic Graph (DAG) computing framework** specifically designed for building and executing complex data processing pipelines. Its Python binding PyCGraph enables Python developers to easily construct efficient parallel computation graphs.

### Why Use CGraph?

- **High Performance**: Based on C++ implementation with multi-threaded parallel execution support
- **Ease of Use**: Concise Python API with declarative programming style
- **Flexibility**: Supports complex dependency relationships, conditional execution, and parameter passing
- **Extensibility**: Rich extension mechanisms (aspects, events, custom nodes)

### Applicable Scenarios

- Data processing pipelines
- Machine learning workflows
- Batch processing task scheduling
- Complex business logic orchestration

## Table of Contents

1. [Basic Concepts](#basic-concepts)
2. [Hello World](#hello-world)
3. [Simple Pipeline](#simple-pipeline)
4. [Cluster Nodes](#cluster-nodes)
5. [Parameter Passing](#parameter-passing)
6. [Conditional Execution](#conditional-execution)
7. [Aspect-Oriented Programming](#aspect-oriented-programming)
8. [Event Handling](#event-handling)
9. [Advanced Features](#advanced-features)

## Basic Concepts

### Core Components Explained

#### GNode (Graph Node)
- **Purpose**: Basic execution unit in the computation graph, similar to functions or tasks
- **Characteristics**: Each node represents a specific computation or operation
- **Lifecycle**: `init()` → `run()` → `destroy()`
- **Examples**: Data reading nodes, data processing nodes, result output nodes

#### GPipeline (Pipeline)
- **Purpose**: Manager and execution engine for the entire computation graph
- **Functions**:
  - Register nodes and dependency relationships
  - Schedule node execution order
  - Manage parallel execution
  - Handle errors and states
- **Usage Flow**: Create → Register nodes → Initialize → Run → Destroy

#### GCluster (Cluster)
- **Purpose**: Combine multiple nodes into a logical unit
- **Advantages**:
  - Simplify management of complex graphs
  - Support cluster-level parallel control
  - Can add aspects or conditions as a whole

#### GParam (Parameter)
- **Purpose**: Container for sharing data between nodes
- **Characteristics**:
  - Thread-safe (supports lock/unlock)
  - Lifecycle management
  - Supports complex data types

#### GCondition (Condition)
- **Purpose**: Select different node branches based on runtime conditions
- **Applications**: Implement if-else logic, dynamic routing

#### GAspect (Aspect)
- **Purpose**: Insert additional logic before and after node execution
- **Applications**: Logging, performance monitoring, resource management

#### GEvent (Event)
- **Purpose**: Asynchronous event handling mechanism
- **Applications**: Status notifications, monitoring alerts, component decoupling

### State Management

#### CStatus (Status Object)
- **Purpose**: Represents the execution status of operations
- **Properties**:
  - Error code (0 indicates success, negative numbers indicate errors)
  - Error message string
- **Methods**:
  - `isErr()`: Check if there's an error
  - `getCode()`: Get error code
  - `getInfo()`: Get error information

## Hello World

Let's start with the simplest example to understand the basic usage of CGraph.

### Step 1: Create a Custom Node

First, we need to create a custom computation node. All computation nodes must inherit from the `GNode` class:

```python
from PyCGraph import GNode, CStatus

class HelloCGraphNode(GNode):
    def run(self):
        """
        The run() method is the core logic of the node
        - Implement specific computation or processing logic here
        - Must return a CStatus object representing execution status
        """
        print("Hello, PyCGraph.")
        return CStatus()  # Return success status
```

**Key Points:**
- `run()` method: Main execution logic of the node, similar to the run method of a thread
- `CStatus()`: Represents successful execution, equivalent to `CStatus(0, "")`
- If an error occurs, you can return `CStatus(-1, "error message")`

### Step 2: Build and Execute Pipeline

```python
from PyCGraph import GPipeline
from MyGNode.HelloCGraphNode import HelloCGraphNode

def tutorial_hello_cgraph():
    # 1. Create pipeline object
    pipeline = GPipeline()

    # 2. Register node to pipeline
    # registerGElement(node_instance, dependency_set, node_name)
    pipeline.registerGElement(HelloCGraphNode())

    # 3. One-click execution: initialize → run → destroy
    pipeline.process()

if __name__ == '__main__':
    tutorial_hello_cgraph()
```

**Execution Flow Explanation:**
1. **Create Pipeline**: `GPipeline()` creates an empty execution pipeline
2. **Register Node**: Add custom node to the pipeline
3. **Execute Pipeline**: The `process()` method automatically completes the full flow of initialization, execution, and cleanup

**Output Result:**
```
Hello, PyCGraph.
```

### process() vs Manual Control

`process()` is a convenient method, equivalent to:

```python
pipeline.init()    # Initialize all nodes
pipeline.run()     # Execute once
pipeline.destroy() # Clean up resources
```

For scenarios requiring multiple runs or fine-grained control, manual control is recommended.

## Simple Pipeline

In practical applications, we usually need to build computation graphs with complex dependency relationships. Let's learn how to create a pipeline with multiple nodes and dependencies.

### Dependency Graph

The computation graph structure we want to build:

```
    A (start node)
   / \
  B   C (parallel execution)
   \ /
    D (convergence node)
```

This graph represents:
- Node A executes first (no dependencies)
- Nodes B and C wait for A to complete, then execute in parallel
- Node D waits for both B and C to complete before executing

### Node Implementation Example

First, let's look at the implementation of basic nodes:

```python
# MyNode1.py
from datetime import datetime
import time
from PyCGraph import GNode, CStatus

class MyNode1(GNode):
    def run(self):
        print(f"[{datetime.now()}] {self.getName()}, entering MyNode1 execution. Sleeping 1 second...")
        time.sleep(1)  # Simulate computation time
        return CStatus()

# MyNode2.py
class MyNode2(GNode):
    def run(self):
        print(f"[{datetime.now()}] {self.getName()}, entering MyNode2 execution. Sleeping 1 second...")
        time.sleep(1)  # Simulate computation time
        return CStatus()
```

### Building the Pipeline

```python
from PyCGraph import GPipeline, CStatus
from MyGNode.MyNode1 import MyNode1
from MyGNode.MyNode2 import MyNode2

def tutorial_simple():
    # 1. Create pipeline and node instances
    pipeline = GPipeline()
    a, b, c, d = MyNode1(), MyNode2(), MyNode1(), MyNode2()

    # 2. Register nodes and their dependencies
    # registerGElement(node, dependency_set, node_name, loop_count)
    pipeline.registerGElement(a, set(), "nodeA")        # a has no dependencies, executes first
    pipeline.registerGElement(b, {a}, "nodeB")          # b depends on a, executes after a completes
    pipeline.registerGElement(c, {a}, "nodeC")          # c depends on a, executes in parallel with b
    pipeline.registerGElement(d, {b, c}, "nodeD")       # d depends on b and c, waits for both to complete

    # 3. Manual control of execution flow (recommended for production environments)
    status = pipeline.init()
    if status.isErr():
        print(f'Pipeline initialization failed: error_code={status.getCode()}, error_message={status.getInfo()}')
        return

    # 4. Run the same pipeline multiple times
    for i in range(3):
        print(f"\n=== Execution {i+1} starts ===")
        status = pipeline.run()
        print(f"=== Execution {i+1} completed, status code: {status.getCode()} ===")

    # 5. Clean up resources
    pipeline.destroy()
```

### Key Concept Explanations

#### Dependency Sets
- `set()`: Empty set, indicates no dependencies
- `{a}`: Depends on node a
- `{b, c}`: Depends on both nodes b and c (both must complete)

#### Execution Order
CGraph automatically determines execution order based on dependency relationships:
1. First execute nodes without dependencies (A)
2. Then execute nodes whose dependencies are satisfied (B, C in parallel)
3. Finally execute nodes whose all dependencies are met (D)

#### Error Handling
- Always check `status.isErr()` to determine if there's an error
- Use `status.getCode()` and `status.getInfo()` to get detailed error information
- Error code 0 indicates success, negative numbers indicate errors

#### Multiple Runs
- `init()` only needs to be called once to initialize all nodes
- `run()` can be called multiple times, each time executing the complete pipeline
- `destroy()` is called at the end to clean up all resources

### Expected Output

```text
=== Execution 1 starts ===
[2025-01-01 10:00:00] nodeA, entering MyNode1 execution. Sleeping 1 second...
[2025-01-01 10:00:01] nodeB, entering MyNode2 execution. Sleeping 1 second...
[2025-01-01 10:00:01] nodeC, entering MyNode1 execution. Sleeping 1 second...
[2025-01-01 10:00:02] nodeD, entering MyNode2 execution. Sleeping 1 second...
=== Execution 1 completed, status code: 0 ===
```

Note that B and C start executing almost simultaneously, demonstrating parallel characteristics.

## Cluster Nodes

When we need to combine multiple related nodes into a logical unit, we can use `GCluster` (cluster). Clusters can simplify the management of complex graphs and provide unified control interfaces.

**Note**: GCluster is a weakened version of GRegion, where nodes are executed sequentially in the order they are added. If you need more complex node combination logic, it's recommended to use GRegion first.

### What is a Cluster?

A cluster is a mechanism to package multiple nodes into a logical unit:
- **Logical Grouping**: Logical combination of related nodes
- **Unified Management**: Can set parallelism for the entire cluster, add aspects, etc.
- **Simplified Dependencies**: Other nodes can depend on the entire cluster without needing to depend on each node within the cluster

### Cluster Execution Mode

How nodes within a cluster execute:
- **Sequential Execution**: Nodes within the cluster execute sequentially in the order they are added
- **No Dependencies**: No dependency relationships between nodes within the cluster
- **Overall Completion**: The cluster is only considered complete when all nodes within it are completed

### Example: Building a Cluster Pipeline

The computation graph structure we want to build:

```text
    A (single node)
   / \
  B   C (B is a cluster containing 3 nodes; C is a single node)
   \ /
    D (single node)
```

```python
from PyCGraph import GPipeline, GCluster
from MyGNode.MyNode1 import MyNode1
from MyGNode.MyNode2 import MyNode2

def tutorial_cluster():
    # 1. Create nodes within the cluster
    b1 = MyNode1("nodeB1")           # Cluster node 1
    b2 = MyNode1("nodeB2")           # Cluster node 2
    b3 = MyNode2("nodeB3")           # Cluster node 3

    # 2. Create cluster (combine multiple nodes)
    b_cluster = GCluster([b1, b2, b3])  # Note: pass a list of nodes

    # 3. Create other independent nodes
    pipeline = GPipeline()
    a = MyNode1()                    # Preceding node
    c = MyNode2()                    # Parallel node
    d = MyNode1()                    # Post node

    # 4. Register nodes and cluster to pipeline
    pipeline.registerGElement(a, set(), "nodeA")
    pipeline.registerGElement(b_cluster, {a}, "clusterB", 2)  # Cluster executes 2 times
    pipeline.registerGElement(c, {a}, "nodeC")
    pipeline.registerGElement(d, {b_cluster, c}, "nodeD", 2)

    # 5. Execute
    pipeline.process()
```

### Key Parameter Explanations

#### Cluster Creation
```python
b_cluster = GCluster([b1, b2, b3])
```
- Pass a list of nodes, not dependency relationships
- Nodes in the list are combined into a logical unit

#### Loop Count Control
```python
pipeline.registerGElement(b_cluster, {a}, "clusterB", 2)
```
- The 4th parameter `2` indicates the cluster's loop execution count
- This means the entire cluster will execute 2 times (each time sequentially executing all nodes within the cluster)
- Default value is 1, meaning execute only once

### Execution Flow Analysis

1. **Node A executes**: Executes first, no dependencies
2. **Cluster B and Node C execute in parallel**:
   - Nodes b1, b2, b3 within cluster B execute sequentially (execute 2 times)
   - Node C starts executing simultaneously
3. **Node D executes**: Waits for both cluster B and node C to complete

### Advantages of Clusters

#### 1. Simplified Dependency Management
```python
# Without cluster: need to depend on each node in the cluster
pipeline.registerGElement(d, {b1, b2, b3, c}, "nodeD")

# With cluster: only need to depend on the cluster itself
pipeline.registerGElement(d, {b_cluster, c}, "nodeD")
```

#### 2. Unified Control
```python
# Can set properties for the entire cluster
b_cluster.addGAspect(MyTimerAspect())  # Add aspect to entire cluster
```

#### 3. Clear Logic
- Combine nodes with related functionality
- Improve code readability and maintainability

### Expected Output

```text
[timestamp] nodeA, entering MyNode1 execution...
[timestamp] nodeB1, entering MyNode1 execution...  # Cluster nodes execute sequentially
[timestamp] nodeB2, entering MyNode1 execution...  # Sequential execution
[timestamp] nodeB3, entering MyNode2 execution...  # Sequential execution
[timestamp] nodeB1, entering MyNode1 execution...  # 2nd loop starts
[timestamp] nodeB2, entering MyNode1 execution...  # 2nd loop
[timestamp] nodeB3, entering MyNode2 execution...  # 2nd loop
[timestamp] nodeC, entering MyNode2 execution...   # Parallel with cluster
[timestamp] nodeD, entering MyNode1 execution...   # Wait for cluster and C to complete
```

## Parameter Passing

In complex computation graphs, nodes often need to share data. CGraph provides the `GParam` mechanism to implement thread-safe parameter passing and sharing.

### Core Concepts of Parameter Passing

#### Why Do We Need Parameter Passing?
- **Data Sharing**: Multiple nodes need to access the same data
- **State Persistence**: Maintain state between multiple runs
- **Thread Safety**: Safely share data in parallel execution environments
- **Lifecycle Management**: Automatically manage parameter creation and destruction

#### Parameter Lifecycle
1. **Creation**: Created in a node's `init()` method
2. **Sharing**: Other nodes access through parameter name
3. **Usage**: Read and write in node's `run()` method
4. **Destruction**: Automatically cleaned up when pipeline is destroyed

### Step 1: Define Parameter Class

All parameter classes must inherit from `GParam`:

```python
from PyCGraph import GParam, CStatus

class MyParam(GParam):
    """
    Custom parameter class for sharing data between nodes
    """
    def __init__(self):
        super().__init__()
        self.value = 0      # Business data 1
        self.count = 0      # Business data 2
        self.data = []      # Can contain any complex data structure

    def reset(self, curStatus: CStatus):
        """
        Reset method: called before each pipeline run
        Used to clean up or reset parameter state
        """
        self.value = 0
        self.data.clear()
        # Note: Usually don't reset count as it may need to persist across runs
```

**Key Points:**
- Inherit from `GParam` class
- Can contain data members of any type
- `reset()` method is used to reset state before each run

### Step 2: Create Parameter Node (Write Node)

```python
from PyCGraph import GNode, CStatus
from MyParams.MyParam import MyParam

class MyWriteParamNode(GNode):
    """
    Node responsible for creating and writing parameters
    """
    def init(self):
        """
        Create parameter in initialization phase
        createGParam(parameter_instance, parameter_name)
        """
        return self.createGParam(MyParam(), "param1")

    def run(self):
        """
        Modify parameter in execution phase
        """
        # 1. Get parameter object
        param: MyParam = self.getGParam("param1")

        # 2. Lock (ensure thread safety)
        param.lock()

        try:
            # 3. Modify parameter data
            param.count += 1
            param.value += 10
            param.data.append(f"data_{param.count}")

            print(f'[{self.getName()}] Write data - value: {param.value}, count: {param.count}')
            print(f'[{self.getName()}] Data list: {param.data}')

        finally:
            # 4. Unlock (ensure unlock even if error occurs)
            param.unlock()

        return CStatus()
```

### Step 3: Read Parameter Node (Read Node)

```python
class MyReadParamNode(GNode):
    """
    Node responsible for reading parameters
    """
    def run(self):
        """
        Read data from shared parameters
        """
        # 1. Get parameter object (by parameter name)
        param: MyParam = self.getGParam("param1")

        # 2. Lock (read operations also need locking)
        param.lock()

        try:
            # 3. Read parameter data
            current_value = param.value
            current_count = param.count
            data_copy = param.data.copy()  # Copy data to avoid external modification

            print(f'[{self.getName()}] Read data - value: {current_value}, count: {current_count}')
            print(f'[{self.getName()}] Data list length: {len(data_copy)}')

        finally:
            # 4. Unlock
            param.unlock()

        return CStatus()
```

### Step 4: Build Parameter Passing Pipeline

```python
from PyCGraph import GPipeline
from MyGNode.MyWriteParamNode import MyWriteParamNode
from MyGNode.MyReadParamNode import MyReadParamNode

def tutorial_param():
    """
    Demonstrate complete parameter passing pipeline
    """
    pipeline = GPipeline()

    # Create node instances
    write_node = MyWriteParamNode()  # Node that creates parameters
    read_node1 = MyReadParamNode()   # Node that reads parameters 1
    read_node2 = MyReadParamNode()   # Node that reads parameters 2
    write_node2 = MyWriteParamNode() # Node that writes again

    # Register nodes and dependencies
    pipeline.registerGElement(write_node, set(), "writeNodeA")      # First create parameter
    pipeline.registerGElement(read_node1, {write_node}, "readNodeB") # Read parameter
    pipeline.registerGElement(read_node2, {write_node}, "readNodeC") # Read in parallel
    pipeline.registerGElement(write_node2, {read_node1, read_node2}, "writeNodeD") # Modify again

    # Execute pipeline
    pipeline.init()

    for i in range(3):
        print(f"\n=== Run {i+1} ===")
        status = pipeline.run()
        print(f"Run status: {status.getCode()}")

    pipeline.destroy()
```

### Importance of Thread Safety

#### Why Do We Need Locking?
```python
# Wrong example: concurrent access without locking
param.value += 1  # May cause data race in parallel environment

# Correct example: lock protection
param.lock()
param.value += 1  # Thread-safe operation
param.unlock()
```

#### Best Practices
```python
# Recommended: use try-finally to ensure unlock
param.lock()
try:
    # All operations on parameter
    param.value += 1
    param.data.append("new_item")
finally:
    param.unlock()  # Ensure unlock even if error occurs
```

## Conditional Execution

In practical applications, we often need to decide which branch to execute based on runtime conditions. CGraph provides the `GCondition` mechanism to implement conditional control similar to if-else logic.

### Core Concepts of Conditional Execution

#### What is Conditional Execution?
Conditional execution allows selecting different node branches based on specific conditions at runtime:
- **Dynamic Selection**: Decide execution path based on runtime state
- **Branch Control**: Implement logic similar to if-else, switch-case
- **Resource Optimization**: Only execute necessary branches, saving computational resources

#### How Conditional Execution Works
1. **Candidate Nodes**: Define multiple nodes that may be executed
2. **Selection Logic**: Implement `choose()` method to return the index of the node to execute
3. **Dynamic Execution**: Only execute the corresponding node based on selection result

### Step 1: Simple Condition Implementation

```python
from PyCGraph import GCondition

class MyCondition(GCondition):
    """
    Simple condition selector
    """
    def choose(self):
        """
        Select the node to execute
        Return value: index of the node in the candidate list (starting from 0)
        """
        # Can implement any selection logic here
        import random
        choice = random.randint(0, 2)  # Randomly select one of 0, 1, 2
        print(f"[MyCondition] Randomly select execution node index: {choice}")
        return choice
```

### Step 2: Parameter-Based Condition

```python
class MyParamCondition(GCondition):
    """
    Parameter-based condition selector
    """
    def choose(self):
        """
        Select execution branch based on shared parameter value
        """
        # Get shared parameter
        param: MyParam = self.getGParam("param1")

        param.lock()
        try:
            current_value = param.value

            # Select different execution branches based on parameter value
            if current_value < 10:
                choice = 0  # Execute first node
                reason = f"value({current_value}) < 10, execute branch 0"
            elif current_value < 20:
                choice = 1  # Execute second node
                reason = f"10 <= value({current_value}) < 20, execute branch 1"
            else:
                choice = 2  # Execute third node
                reason = f"value({current_value}) >= 20, execute branch 2"

            print(f"[MyParamCondition] {reason}")
            return choice

        finally:
            param.unlock()
```

### Step 3: Create Condition Nodes

```python
# Create different nodes for different branches
class ConditionNodeA(GNode):
    def run(self):
        print(f"[{self.getName()}] Execute branch A - handle small value situation")
        return CStatus()

class ConditionNodeB(GNode):
    def run(self):
        print(f"[{self.getName()}] Execute branch B - handle medium value situation")
        return CStatus()

class ConditionNodeC(GNode):
    def run(self):
        print(f"[{self.getName()}] Execute branch C - handle large value situation")
        return CStatus()
```

### Step 4: Build Conditional Execution Pipeline

```python
from PyCGraph import GPipeline
from MyGNode.MyWriteParamNode import MyWriteParamNode
from MyGNode.MyReadParamNode import MyReadParamNode
from MyGCondition.MyCondition import MyCondition
from MyGCondition.MyParamCondition import MyParamCondition

def tutorial_condition():
    """
    Demonstrate complete conditional execution pipeline
    """
    # 1. Create candidate nodes for condition branches
    b0 = ConditionNodeA("conditionNodeB0")  # Branch 0
    b1 = ConditionNodeB("conditionNodeB1")  # Branch 1
    b2 = ConditionNodeC("conditionNodeB2")  # Branch 2

    # 2. Create condition selector (pass candidate node list)
    b_condition = MyCondition([b0, b1, b2])

    # 3. Create another group of condition branches
    d0 = MyNode1("conditionNodeD0")
    d1 = MyNode1("conditionNodeD1")
    d2 = MyNode1("conditionNodeD2")

    # 4. Create parameter-based condition selector
    d_condition = MyParamCondition([d0, d1, d2])

    # 5. Build complete pipeline
    pipeline = GPipeline()
    write_node = MyWriteParamNode()  # Create and modify parameters
    read_node = MyReadParamNode()    # Read parameters

    # 6. Register nodes and dependencies
    pipeline.registerGElement(write_node, set(), "writeNodeA")           # First write parameter
    pipeline.registerGElement(b_condition, {write_node}, "conditionB")   # Select branch based on parameter
    pipeline.registerGElement(read_node, {b_condition}, "readNodeC")     # Read parameter state
    pipeline.registerGElement(d_condition, {read_node}, "conditionD")    # Condition selection again

    # 7. Execute multiple times to observe different selection results
    pipeline.init()
    for i in range(5):
        print(f"\n=== Run {i+1} ===")
        status = pipeline.run()
        print(f"Run status: {status.getCode()}")

    pipeline.destroy()
```

### Advanced Features of Conditional Execution

#### 1. Nested Conditions
```python
# Can nest conditions within condition branches
inner_condition = MyCondition([node1, node2])
outer_condition = MyCondition([inner_condition, node3, node4])
```

#### 2. Dynamic Conditions
```python
class DynamicCondition(GCondition):
    def choose(self):
        # Can make selections based on external state, time, random numbers, etc.
        import datetime
        hour = datetime.datetime.now().hour
        if hour < 12:
            return 0  # Execute branch 0 in the morning
        else:
            return 1  # Execute branch 1 in the afternoon
```

#### 3. Conditions Combined with Clusters
```python
# Condition candidates can also be clusters
cluster1 = GCluster([node1, node2])
cluster2 = GCluster([node3, node4])
condition = MyCondition([cluster1, cluster2])
```

### Usage Scenario Examples

#### 1. Data Processing Branches
```python
class DataProcessCondition(GCondition):
    def choose(self):
        data_size = self.getGParam("data_info").size
        if data_size < 1000:
            return 0  # Small dataset processing
        elif data_size < 10000:
            return 1  # Medium dataset processing
        else:
            return 2  # Large dataset processing
```

#### 2. Error Handling Branches
```python
class ErrorHandlingCondition(GCondition):
    def choose(self):
        error_count = self.getGParam("error_info").count
        if error_count == 0:
            return 0  # Normal processing flow
        elif error_count < 5:
            return 1  # Minor error handling
        else:
            return 2  # Severe error handling
```

### Important Notes

1. **Index Range**: The index returned by `choose()` must be within the candidate node list range
2. **Thread Safety**: When accessing shared parameters in the `choose()` method, lock is required
3. **Performance Consideration**: Condition selection logic should be as simple and fast as possible
4. **Debug-Friendly**: It's recommended to print logs during selection for debugging and monitoring

## Aspect-Oriented Programming

Aspect-Oriented Programming (AOP) is a programming paradigm that allows inserting additional logic at specific points in program execution without modifying the original code. CGraph's `GAspect` provides powerful aspect support.

### Application Scenarios of Aspect Programming

- **Performance Monitoring**: Statistics on node execution time
- **Logging**: Record node execution status
- **Resource Management**: Manage resources before and after execution
- **Error Handling**: Unified exception handling logic
- **Permission Checking**: Permission verification before execution

### Create Timer Aspect

```python
from PyCGraph import GAspect, CStatus
import time

class MyTimerAspect(GAspect):
    """
    Timer aspect: monitor node execution time
    """
    _start_time = None
    
    def beginRun(self):
        """
        Called before node execution
        """
        self._start_time = time.time()
        print(f"[TimerAspect] {self.getName()} starts execution...")
        return CStatus()

    def finishRun(self, curStatus: CStatus):
        """
        Called after node execution
        """
        span = time.time() - self._start_time
        print(f"[TimerAspect] {self.getName()} execution completed, time taken: {span:.3f} seconds")
        return
```

### Create Log Aspect

```python
class MyLogAspect(GAspect):
    """
    Log aspect: record detailed execution information
    """
    def beginRun(self):
        print(f"[LogAspect] === {self.getName()} preparing to execute ===")
        print(f"[LogAspect] Current time: {time.strftime('%Y-%m-%d %H:%M:%S')}")
        return CStatus()

    def finishRun(self, curStatus: CStatus):
        print(f"[LogAspect] === {self.getName()} execution ended ===")
        return
```

### Using Aspects

```python
from PyCGraph import GPipeline, GCluster
from MyGNode.MyNode1 import MyNode1
from MyGNode.MyNode2 import MyNode2
from MyGAspect.MyTimerAspect import MyTimerAspect
from MyGAspect.MyLogAspect import MyLogAspect

def tutorial_aspect():
    """
    Demonstrate the use of aspect programming
    """
    # 1. Create nodes and cluster
    a = MyNode1()
    b1, b2 = MyNode2('nodeB1'), MyNode1('nodeB2')
    b_cluster = GCluster([b1, b2])
    c = MyNode2()

    # 2. Add aspects to nodes
    a.addGAspect(MyTimerAspect())           # Add timer aspect to single node
    a.addGAspect(MyLogAspect())             # Same node can have multiple aspects

    b_cluster.addGAspect(MyTimerAspect())   # Add aspect to entire cluster

    c.addGAspect(MyTimerAspect())           # Add aspect to another node

    # 3. Build and execute pipeline
    pipeline = GPipeline()
    pipeline.registerGElement(a, set(), 'nodeA')
    pipeline.registerGElement(b_cluster, {a}, 'clusterB')
    pipeline.registerGElement(c, {b_cluster}, 'nodeC')

    pipeline.process()
```

### Aspect Execution Order

When a node has multiple aspects, the execution order is:
1. Execute all aspects' `beginRun()` in the order they were added
2. Execute the node's `run()` method
3. Execute all aspects' `finishRun()` in reverse order

```python
# Add order: Timer -> Log
node.addGAspect(MyTimerAspect())
node.addGAspect(MyLogAspect())

# Execution order:
# 1. MyTimerAspect.beginRun()
# 2. MyLogAspect.beginRun()
# 3. node.run()
# 4. MyLogAspect.finishRun()
# 5. MyTimerAspect.finishRun()
```

## Event Handling

CGraph provides a powerful event mechanism that allows nodes to trigger asynchronous events during execution, enabling loose-coupled component communication.

### Core Concepts of Event Mechanism

#### Purpose of Events
- **Decoupled Communication**: Loose-coupled communication between nodes
- **Asynchronous Processing**: Event processing doesn't block the main flow
- **Status Notification**: Notify system status changes
- **Monitoring Alerts**: Implement monitoring and alerting mechanisms

#### Event Workflow
1. **Define Events**: Create event classes inheriting from `GEvent`
2. **Register Events**: Register events to the pipeline
3. **Trigger Events**: Trigger events in nodes
4. **Handle Events**: Event handlers execute asynchronously

### Step 1: Define Event Handlers

```python
from PyCGraph import GEvent, CStatus

class MyPrintEvent(GEvent):
    """
    Simple print event handler
    """
    def trigger(self, _):
        """
        Event handling logic when triggered
        """
        print(f"[PrintEvent] Event received")
        return CStatus()

class MyLogEvent(GEvent):
    """
    Log recording event handler
    """
    def trigger(self, _):
        import datetime
        timestamp = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        print(f"[LogEvent] {timestamp} - Event recorded")
        return CStatus()
```

### Step 2: Create Event Triggering Node

```python
from PyCGraph import GNode, CStatus

class MyEventNode(GNode):
    """
    Node capable of triggering events
    """
    def run(self):
        node_name = self.getName()
        print(f"[{node_name}] Starting execution, preparing to trigger events...")

        # Trigger events - use correct API
        from PyCGraph import GEventType
        self.notify("print_event", GEventType.SYNC)
        self.notify("log_event", GEventType.SYNC)

        # Trigger different events based on conditions
        import random
        if random.random() > 0.5:
            self.notify("print_event", GEventType.SYNC)

        print(f"[{node_name}] Execution completed")
        return CStatus()
```

### Step 3: Build Event-Driven Pipeline

```python
from PyCGraph import GPipeline
from MyGNode.MyNode1 import MyNode1
from MyGNode.MyEventNode import MyEventNode
from MyGNode.MyWriteParamNode import MyWriteParamNode
from MyGEvent.MyPrintEvent import MyPrintEvent
from MyGEvent.MyLogEvent import MyLogEvent

def tutorial_event():
    """
    Demonstrate complete event handling pipeline
    """
    # 1. Create nodes
    pipeline = GPipeline()
    write_node = MyWriteParamNode()     # Parameter writing node
    event_node1 = MyEventNode()         # Event triggering node 1
    normal_node = MyNode1()             # Normal node
    event_node2 = MyEventNode()         # Event triggering node 2

    # 2. Register nodes and dependencies
    pipeline.registerGElement(write_node, set(), "writeNodeA")
    pipeline.registerGElement(event_node1, {write_node}, "eventNodeB")
    pipeline.registerGElement(normal_node, {event_node1}, "normalNodeC")
    pipeline.registerGElement(event_node2, {normal_node}, "eventNodeD")

    # 3. Register event handlers
    pipeline.addGEvent(MyPrintEvent(), "print_event")  # Register print event
    pipeline.addGEvent(MyLogEvent(), "log_event")      # Register log event

    # 4. Execute pipeline
    print("=== Starting event-driven pipeline execution ===")
    pipeline.process()
    print("=== Pipeline execution completed ===")
```

### Advanced Features of Events

#### 1. Event Parameter Passing
```python
# No parameters passed when triggering events, use notify method
from PyCGraph import GEventType
self.notify("my_event", GEventType.SYNC)
```

#### 2. Conditional Event Triggering
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

#### 3. Multiple Event Handlers
```python
# Multiple handlers can be registered for the same event name
pipeline.addGEvent(MyPrintEvent(), "status_event")
pipeline.addGEvent(MyLogEvent(), "status_event")
pipeline.addGEvent(MyEmailEvent(), "status_event")  # All will be triggered
```

### Events vs Parameters Selection

#### Use Events for:
- Status notifications and monitoring
- Asynchronous processing that doesn't affect main flow
- Loose-coupled component communication
- Logging and alerting

#### Use Parameters for:
- Data passing between nodes
- State sharing and synchronization
- Data persistence needed
- Thread-safe data access needed

### Important Notes

1. **Asynchronous Execution**: Event processing is asynchronous and doesn't block the main flow
2. **Error Isolation**: Errors in event processing don't affect the main flow
3. **Performance Consideration**: Frequent event triggering may affect performance
4. **Debugging Difficulty**: Asynchronous events may increase debugging complexity

## Advanced Features

### Multi-Pipeline Parallel Execution
```python
def tutorial_multi_pipeline():
    """
    Create multiple independent pipelines for parallel execution
    """
    pipeline1 = GPipeline()
    pipeline2 = GPipeline()

    # Configure different pipelines separately
    # pipeline1 processes data stream A
    # pipeline2 processes data stream B

    # Can execute in parallel
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

### Timeout Control
```python
def tutorial_timeout():
    """
    Set node execution timeout
    """
    pipeline = GPipeline()
    slow_node = SlowNode()
    pipeline.registerGElement(slow_node, set(), "slowNode")
    
    # Set node timeout (milliseconds)
    slow_node.setTimeout(5000)  # 5 second timeout

    # If node execution exceeds 5 seconds, it will be automatically terminated
    pipeline.process()
```

### Dynamic Node Creation
```python
class DynamicNodeFactory:
    """
    Factory class for dynamically creating nodes
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

## Best Practices

### 1. Error Handling and Status Checking

```python
def robust_pipeline_execution():
    """
    Robust pipeline execution pattern
    """
    pipeline = GPipeline()
    # ... register nodes

    # Initialization check
    status = pipeline.init()
    if status.isErr():
        print(f"Initialization failed: {status.getCode()} - {status.getInfo()}")
        return False

    try:
        # Execution check
        status = pipeline.run()
        if status.isErr():
            print(f"Execution failed: {status.getCode()} - {status.getInfo()}")
            return False

        print("Execution successful")
        return True

    finally:
        # Ensure resource cleanup
        pipeline.destroy()
```

### 2. Thread-Safe Parameter Usage

```python
class SafeParamNode(GNode):
    """
    Thread-safe parameter usage example
    """
    def run(self):
        param = self.getGParam("shared_param")

        # Recommended: use try-finally to ensure unlock
        param.lock()
        try:
            # All parameter operations under lock protection
            old_value = param.value
            param.value = old_value + 1
            param.data.append(f"update_{param.value}")

        finally:
            param.unlock()  # Ensure unlock

        return CStatus()
```

### 3. Performance Optimization Strategies

```python
def optimized_pipeline():
    """
    Performance-optimized pipeline configuration
    """
    pipeline = GPipeline()

    # 1. Reasonable parallelism settings
    cpu_intensive_node = MyNode1()
    io_intensive_cluster = GCluster([IONode1(), IONode2(), IONode3()])

    # CPU-intensive nodes: parallelism = CPU core count
    pipeline.registerGElement(cpu_intensive_node, set(), "cpu_node", 4)

    # IO-intensive clusters: can set higher parallelism
    pipeline.registerGElement(io_intensive_cluster, {cpu_intensive_node}, "io_cluster", 8)

    # 2. Use aspects to monitor performance
    cpu_intensive_node.addGAspect(MyTimerAspect())
    io_intensive_cluster.addGAspect(MyTimerAspect())

    pipeline.process()
```

### 4. Debugging and Monitoring

```python
class DebugAspect(GAspect):
    """
    Debug aspect: detailed execution information recording
    """
    def beforeRun(self):
        node_name = self.getBelongName()
        print(f"[DEBUG] {node_name} starts execution")
        print(f"[DEBUG] Current thread: {threading.current_thread().name}")
        return CStatus()

    def afterRun(self):
        node_name = self.getBelongName()
        print(f"[DEBUG] {node_name} execution completed")
        return CStatus()

def debug_pipeline():
    """
    Debug mode pipeline
    """
    pipeline = GPipeline()
    # ... create nodes

    # Add debug aspects to all nodes
    for node in [a, b, c, d]:
        node.addGAspect(DebugAspect())

    pipeline.process()
```

### 5. Configuration Management

```python
class PipelineConfig:
    """
    Pipeline configuration management
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
        # Load configuration from file
        pass

    def create_pipeline(self):
        pipeline = GPipeline()
        pipeline.setTimeout(self.config["timeout"])
        return pipeline
```

### 6. Common Pitfalls and Solutions

#### Pitfall 1: Forgetting to Unlock Parameters
```python
# Wrong: may cause deadlock
def bad_example():
    param.lock()
    if some_condition:
        return CStatus(-1, "error")  # Forgot to unlock!
    param.unlock()

# Correct: use try-finally
def good_example():
    param.lock()
    try:
        if some_condition:
            return CStatus(-1, "error")
    finally:
        param.unlock()
```

#### Pitfall 2: Circular Dependencies
```python
# Wrong: will cause deadlock
pipeline.registerGElement(a, {b}, "nodeA")
pipeline.registerGElement(b, {a}, "nodeB")  # Circular dependency!

# Correct: ensure dependency relationships form a directed acyclic graph
pipeline.registerGElement(a, set(), "nodeA")
pipeline.registerGElement(b, {a}, "nodeB")
```

#### Pitfall 3: Condition Selection Out of Bounds
```python
# Wrong: may return index out of range
class BadCondition(GCondition):
    def choose(self):
        return 5  # If there are only 3 candidate nodes, this will error!

# Correct: ensure return value is within valid range
class GoodCondition(GCondition):
    def choose(self):
        candidate_count = len(self.getGElementList())
        return random.randint(0, candidate_count - 1)
```

## Summary

CGraph Python is a powerful computation graph framework that provides:

### Core Advantages
- **High Performance**: Efficient execution engine based on C++
- **Ease of Use**: Concise and intuitive Python API
- **Flexibility**: Supports complex dependency relationships and execution patterns
- **Extensibility**: Rich extension mechanisms (aspects, events, custom nodes)
- **Thread Safety**: Built-in concurrency control and data protection

### Main Features
1. **Nodes and Pipelines**: Basic components for building computation graphs
2. **Dependency Management**: Automatic topological sorting and execution scheduling
3. **Parallel Execution**: Efficient multi-threaded parallel processing
4. **Parameter Passing**: Thread-safe data sharing mechanism
5. **Conditional Control**: Flexible branch execution logic
6. **Aspect Programming**: Unified handling of cross-cutting concerns
7. **Event Mechanism**: Loose-coupled asynchronous communication

### Applicable Scenarios
- **Data Processing Pipelines**: ETL, data cleaning, feature engineering
- **Machine Learning Workflows**: Model training, validation, inference flows
- **Batch Processing Systems**: Large-scale data batch processing tasks
- **Business Process Orchestration**: Automated execution of complex business logic
- **Microservice Coordination**: Dependency management and scheduling between services

### Learning Recommendations
1. **Start Simple**: First master basic node and pipeline concepts
2. **Gradual Deepening**: Learn advanced features like parameters, conditions, aspects, events in sequence
3. **Practice-Oriented**: Deepen understanding through actual projects
4. **Focus on Performance**: Learn to use monitoring and debugging tools
5. **Follow Best Practices**: Avoid common pitfalls, write robust code

By mastering these concepts and techniques, you can use CGraph Python to build efficient, reliable, and maintainable computation graph applications.
