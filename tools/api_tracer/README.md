# API 调用追踪器 (API Call Tracer)

这是一个用于在运行时动态捕获框架 API 调用的工具。现已实现 PyTorch 原生底层的追踪功能。

其主要功能是抓取 API 的实际调用参数（配置），并生成配置集，可以为后续的模糊测试、随机测试或 API 重放提供数据支持。

## 主要功能

1. 能够追踪 `torch` 的原生函数和底层算子（即所有通过 `__torch_function__` 协议暴露的算子），而非 `python` 封装函数
2. 通过挂钩（Hook）机制，在代码实际执行时动态捕获每一次 API 调用及其参数
3. 将捕获到的 API 调用信息序列化为两种格式并保存：
    - `api_trace.yaml`: 结构化数据，便于序列化与反序列化
    - `api_trace.txt`: 人类可读格式，便于测试去重

## 技术设计与实现

### 设计原则

- **High Decoupling**:

    **`api_tracer.py`**:
    - `APITracer`: 顶层控制器，负责追踪任务的生命周期管理（启动、停止）

    **`framework_dialect.py`**:
    - `TracingHook`: 钩子实现类，定义了具体的API拦截策略
    - `FrameworkDialect`: 框架方言组，封装特定框架（如PyTorch）的独有逻辑，如特殊类型序列化

    **`config_serializer.py`**:
    - `ConfigSerializer`: 序列化类，负责将捕获的数据格式化并写入文件

- **High Extensibility**:

    - `TracingHook` : 允许开发者通过继承该基类，来实现全新的API挂钩策略
    - `FrameworkDialect` : 用户可以继承该基类，通过重写方法来快速支持新的深度学习框架

### 多功能钩子策略

- `TorchFunctionHook`:
    通过重写 PyTorch 官方的 `torch.overrides.TorchFunctionMode` 类实现。经过测试，这是追踪 PyTorch API 的首选方法，能够高效、准确地捕获所有进入 PyTorch C++ 后端的函数调用（Torch C API），覆盖范围广、对用户代码无侵入。

- `SetattrHook`:
    通过 Python 的 `setattr` 机制，在运行时动态遍历并替换模块中的函数对象。它可以挂钩任意 Python 库的 API。可用于扫描全库并产出 api_list，追踪纯 Python 函数的调用。
    
    但由于 PyTorch 的大量核心功能由 C++ 实现，`SetattrHook` 无法覆盖到底层算子，表现不尽人意。因此对于 PyTorch 不再启用，留作备用/辅助策略，当前默认仅使用 `TorchFunctionHook` 对 PyTorch 进行追踪。

### PyTorch 追踪相关实现

- `TorchFunctionHook` 启用全局的 `TorchFunctionMode`，所有支持该协议的 PyTorch API 调用都会被重载的 `__torch_function__` 方法捕获
- `FrameworkDialect` 抽象类实现 `PyTorchDialect`，其方法 `serialize_special_type`, `format_special_type` 实现了针对 `torch.Tensor`, `torch.dtype`, `torch.device` 等 PyTorch 特有类型的序列化逻辑

## 如何使用

使用 `APITracer` 非常简单，只需将其作为一个上下文管理器（Context Manager）包裹住需要追踪的 PyTorch 代码即可。

**示例代码:**

```python
import torch
from api_tracer import APITracer

# 初始化 Tracer，指定框架方言（当前为 'torch'）和输出路径
tracer = APITracer(dialect="torch", output_path="trace_output")

# 使用 with 语句来自动启动和停止追踪
with tracer:
    # 执行 PyTorch 模型或代码
    tensor1 = torch.randn(2, 3, device='cpu')
    tensor2 = torch.ones(2, 3)
    result = torch.add(tensor1, tensor2, alpha=10)
    final_sum = result.sum()

# 或者手动启动和停止追踪：
# tracer.start()
# 执行 PyTorch 模型或代码
# tracer.stop()
```

### 输出文件

执行上述代码后，你将在 `trace_output` 目录下找到两个文件：

1.  **`api_trace.yaml`**: 结构化的 API 调用记录

    ```yaml
    - api: torch.randn
      args:
      - 2
      - 3
      kwargs:
        device: cpu
    - api: torch.ones
      args:
      - 2
      - 3
      kwargs: {}
    - api: torch.add
      args:
      - type: torch.Tensor
        shape:
        - 2
        - 3
        dtype: torch.float32
        device: cpu
      - type: torch.Tensor
        shape:
        - 2
        - 3
        dtype: torch.float32
        device: cpu
      kwargs:
        alpha: 10
    - api: torch.Tensor.sum
      args:
      - type: torch.Tensor
        shape:
        - 2
        - 3
        dtype: torch.float32
        device: cpu
      kwargs: {}
    ```

2.  **`api_trace.txt`**: 更易读的格式

    ```text
    torch.randn(2, 3, device="cpu")
    torch.ones(2, 3)
    torch.add(Tensor([2, 3], "float32"), Tensor([2, 3], "float32"), alpha=10)
    torch.Tensor.sum(Tensor([2, 3], "float32"))
    ```
