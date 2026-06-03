# 运行 external_compressible_flow.py 说明

## 案例简介

这个案例模拟**跨声速机翼的外部可压缩流动**（External Compressible Flow），使用：

- **几何类型**：ONERA M6 机翼（经典跨声速验证算例）
- **来流条件**：马赫数 M=0.8395，攻角 α=3.06°
- **湍流模型**：k-ω SST
- **网格**：Watertight Geometry 工作流生成的多面体-六面体核心网格（poly-hexcore）
- **求解器**：压力基耦合求解器（Pressure-Based Coupled Solver），全局时间步长伪瞬态法
- **气体模型**：理想气体 + Sutherland 粘性定律

详细文档见：[PyFluent 官方文档 - External Compressible Flow](https://fluent.docs.pyansys.com/stable/examples/00-fluent/external_compressible_flow.html)

---

## 前置条件

### 1. 安装 Ansys Fluent

此脚本需要**本地已安装 Ansys Fluent**（2022 R2 或更高版本），并持有有效 License。

验证 Fluent 是否可用：

```powershell
# Windows PowerShell
& "C:\Program Files\ANSYS Inc\vXXX\fluent\ntbin\win64\fluent.exe" -help
```

### 2. 安装 Python 依赖

```bash
# 在项目根目录下，以可编辑模式安装 ansys-fluent-core
pip install -e .

# 或者直接安装
pip install ansys-fluent-core
```

验证安装：

```bash
python -c "import ansys.fluent.core; print(ansys.fluent.core.__version__)"
```

### 3. 检查 License

确保 Fluent 的 License Manager 正在运行，且你有可用的求解器席位。

#### 方式一：检查 ANSYS 相关进程（推荐，无需知道安装路径）

在终端中运行：

```bash
# 检查 ANSYS/Fluent 许可证相关进程
tasklist | grep -i -E "ansys|fluent|lmgr|lmgrd"
```

若输出中包含 **`ansyslmd.exe`** 和 **`lmgrd.exe`**，说明 License Manager 正在运行。

也可用 PowerShell 查看进程详情（含启动时间）：

```powershell
Get-Process -Name '*ansys*','*fluent*','*lmgr*','*lmgrd*','*flexlm*' -ErrorAction SilentlyContinue | Select-Object ProcessName, Id, StartTime | Format-Table -AutoSize
```

以及检查相关 Windows 服务：

```powershell
Get-Service -Name '*ansys*','*flexlm*' -ErrorAction SilentlyContinue | Format-Table -AutoSize
```

#### 方式二：使用 lmutil（需指定 ANSYS 安装路径）

```powershell
& "C:\Program Files\ANSYS Inc\Shared Files\Licensing\winx64\lmutil.exe" lmstat -a
```

该命令可查看详细的许可证席位使用情况。

---

## 运行方法

### 方法一：直接运行 Python 脚本（推荐）

```bash
cd e:\Github\pyfluentybb\examples\00-fluent
python external_compressible_flow.py
```

### 方法二：在 VS Code 中运行

1. 在 VS Code 中打开 `external_compressible_flow.py`
2. 确保已选择正确的 Python 解释器（`Ctrl+Shift+P` → `Python: Select Interpreter`）
3. 点击右上角的 **运行按钮**（▶️），或按 `Ctrl+F5`

### 方法三：逐段运行（适合学习和调试）

在 VS Code 中使用 **Python Interactive Window**（`Shift+Enter`），逐 cell 运行：

```python
# Cell 1: 导入和下载几何文件
import shutil
import tempfile
import ansys.fluent.core as pyfluent
from ansys.fluent.core import examples

wing_spaceclaim_file, wing_intermediary_file = [
    examples.download_file(CAD_file, "pyfluent/external_compressible")
    for CAD_file in ["wing.scdoc", "wing.pmdb"]
]

# Cell 2: 启动 Fluent（网格模式）
meshing_session = pyfluent.launch_fluent(
    precision="double",
    processor_count=4,
    mode="meshing",
)
# ... 依此类推
```

---

## 运行过程说明

脚本执行分为两大阶段，预计总耗时 **10-30 分钟**（取决于硬件配置）：

### 阶段一：网格划分（Meshing）

| 步骤 | 说明 | 耗时 |
|------|------|------|
| 1. 下载几何文件 | 自动下载 `wing.scdoc` 和 `wing.pmdb` | ~10s |
| 2. 启动 Fluent | 以网格模式启动，双精度，4 核 | ~15s |
| 3. 导入几何 | 将机翼 CAD 导入 Fluent | ~5s |
| 4. 添加局部尺寸控制 | 对机翼表面、边缘、BOI 区域设置网格尺寸 | ~10s |
| 5. 生成面网格 | MaxSize=1000mm, MinSize=2mm | ~2-5min |
| 6. 描述几何 | 定义流体区域 | ~10s |
| 7. 更新边界/区域 | 自动识别边界类型和流体区域 | ~10s |
| 8. 添加边界层 | 12 层边界层网格 | ~30s |
| 9. 生成体网格 | Poly-hexcore 格式，HexMaxCellLength=512mm | ~5-15min |
| 10. 保存网格 | 写入 `wing.msh.h5` | ~10s |

### 阶段二：求解计算（Solving）

| 步骤 | 说明 | 耗时 |
|------|------|------|
| 11. 切换至求解模式 | 从网格模式切换到求解器模式 | ~10s |
| 12. 设置物理模型 | k-ω SST 湍流，理想气体，Sutherland 粘性 | ~5s |
| 13. 设置边界条件 | 压力远场，M=0.8395，攻角=3.06° | ~5s |
| 14. 设置运行条件 | 操作压力 80600 Pa | ~5s |
| 15. 混合初始化 | Hybrid Initialization | ~10s |
| 16. 迭代计算 | 25 次迭代 | ~2-5min |
| 17. 保存结果 | 写入 `.cas.h5` 文件 | ~10s |

---

## 常见问题

### Q1：`ModuleNotFoundError: No module named 'ansys'`

**原因**：未安装 `ansys-fluent-core`。

**解决**：
```bash
pip install ansys-fluent-core
```

### Q2：`Fluent launch failed` / `Connection refused`

**原因**：Fluent 未正确安装或 License 不可用。

**解决**：
1. 确认 Fluent 已安装并在 PATH 中
2. 检查 License Manager 状态
3. 尝试在命令行手动启动 Fluent 验证：
   ```bash
   fluent 2ddp -meshing  # 2D 双精度网格模式
   ```

### Q3：下载几何文件失败（网络问题）

**原因**：脚本自动从 Ansys 服务器下载几何文件，需要网络连接。

**解决**：
1. 检查网络连接
2. 如果在内网环境，可能需要配置代理：
   ```python
   import os
   os.environ["HTTPS_PROXY"] = "http://your-proxy:port"
   ```

### Q4：网格生成过程中内存不足

**原因**：体网格生成（poly-hexcore）对内存需求较高。

**解决**：
- 减少 `processor_count` 参数（如设为 2）
- 增大 `HexMaxCellLength`（如 1024）以降低网格密度
- 确保主机至少有 16 GB RAM

### Q5：`could not find a matching Fluent installation`

**原因**：Fluent 安装路径不在默认位置。

**解决**：设置环境变量或指定 Fluent 路径：
```python
meshing_session = pyfluent.launch_fluent(
    precision="double",
    processor_count=4,
    mode="meshing",
    product_version="2024R2",  # 指定版本
)
```

---

## 代码逐行详解（Python 初学者友好版）

> 本节面向 **Python 初学者**，用通俗语言解释脚本中每一段代码的作用。
> 即使你不懂 Python，读完本节也能理解脚本在做什么、每行代码的意义。

---

### 0. 先理解几个 Python 基础概念

在阅读代码之前，先花 3 分钟了解以下概念，会让后续理解顺畅很多：

| 概念 | 通俗解释 | 代码中的例子 |
|------|----------|-------------|
| **`import`** | "拿来用"——把别人写好的工具箱导入到当前脚本中 | `import ansys.fluent.core` = "把 Ansys Fluent 的 Python 工具箱拿来" |
| **变量** | 给数据贴一个标签，方便后面引用 | `meshing_session = ...` = "把启动的 Fluent 会话贴个标签叫 `meshing_session`" |
| **对象** | 一个"东西"，它有属性（数据）和方法（能做的事） | `meshing_session` 是一个 Fluent 会话对象 |
| **`.xxx`** | 访问对象的属性或调用其方法 | `meshing_session.workflow` = "访问这个会话里的 workflow 模块" |
| **方法（函数）** | 让对象执行一个动作 | `.Execute()` = "执行！"；`.set_state(...)` = "把参数设置成..." |
| **字典 `{}`** | 用 `{键: 值}` 格式存放一组配置参数 | `{"FileName": "wing.pmdb"}` = "文件名这个参数，值是 wing.pmdb" |
| **列表 `[]`** | 方括号里放一串东西，有顺序 | `["wing.scdoc", "wing.pmdb"]` = 两个文件名组成的列表 |
| **注释 `#`** | `#` 开头的是给人看的备注，Python 会忽略它 | `# 这是注释，解释代码在做什么` |
| **字符串 `""`** | 双引号（或单引号）包裹的是文本数据 | `"meshing"` 是文本，不是变量 |

---

### 1. 脚本头部：元数据与版权声明（第 1-57 行）

```python
# /// script
# dependencies = [
#   "ansys-fluent-core",
# ]
# ///
```

**作用**：声明这个脚本依赖 `ansys-fluent-core` 这个包。某些工具（如 `uv`、`pipx`）读取这段注释后，会自动帮你安装依赖。
`# /// script` 和 `# ///` 是特殊的标记符，告诉工具"中间这几行是脚本元数据"。

```python
# Copyright (C) 2021 - 2026 ANSYS, Inc. and/or its affiliates.
# SPDX-License-Identifier: MIT
```

**作用**：版权声明和开源许可证标识。`MIT` 是最宽松的开源许可证之一，意味着你可以自由使用、修改、分发此代码。

```python
""".. _ref_external_compressible_flow_settings_api:

Modeling External Compressible Flow
-----------------------------------
...
"""
```

**作用**：**文档字符串（docstring）**。三个双引号 `"""..."""` 之间的文字是多行字符串，Python 不会执行它，而是把它当作文档。
这里描述了案例的目的：计算跨声速机翼（ONERA M6）在攻角 3.06°、马赫数 0.8395 下的湍流流动。

---

### 2. 导入模块（第 76-81 行）

```python
import shutil
import tempfile
```

| 模块 | 用途 |
|------|------|
| `shutil` | **sh**ell **util**ities——文件和目录操作工具（复制、删除文件夹等） |
| `tempfile` | 创建临时文件和临时目录（用完就删的那种） |

```python
import ansys.fluent.core as pyfluent
```

- `import ansys.fluent.core`：导入 Ansys 官方的 PyFluent 库
- `as pyfluent`：起一个**别名**。后续代码中写 `pyfluent.launch_fluent(...)` 比写全称 `ansys.fluent.core.launch_fluent(...)` 更简短

```python
from ansys.fluent.core import examples
```

- `from ... import ...`：只从模块中取**某一部分**来用
- 这里只取了 `examples` 这个子模块，它提供了下载官方示例文件的功能

```python
wing_spaceclaim_file, wing_intermediary_file = [
    examples.download_file(CAD_file, "pyfluent/external_compressible")
    for CAD_file in ["wing.scdoc", "wing.pmdb"]
]
```

这是整个脚本中**最复杂的一行**，拆解开来看：

**外层**：`变量A, 变量B = [列表]`
- 等号左边两个变量，右边是一个列表 → Python 自动把列表的第 0 个元素赋给 `wing_spaceclaim_file`，第 1 个赋给 `wing_intermediary_file`（这叫"解包" / unpacking）

**内层**：`[ ... for CAD_file in ["wing.scdoc", "wing.pmdb"]]`
- 这是**列表推导式（list comprehension）**，等价于：
  ```python
  结果列表 = []
  for CAD_file in ["wing.scdoc", "wing.pmdb"]:  # 依次取 "wing.scdoc" 和 "wing.pmdb"
      结果列表.append(examples.download_file(CAD_file, "pyfluent/external_compressible"))
  ```

**核心调用**：`examples.download_file(CAD_file, "pyfluent/external_compressible")`
- 从 Ansys 官方服务器下载几何文件
- 第一个参数是文件名（`"wing.scdoc"` 或 `"wing.pmdb"`）
- 第二个参数是服务器上的目录路径
- **返回值**：下载后文件在本地的**完整路径**（字符串）

**最终效果**：下载两个几何文件，把它们的本地路径分别存入两个变量。`wing_intermediary_file`（即 `wing.pmdb`）是后续实际用于网格划分的文件。

---

### 3. 启动 Fluent 网格模式（第 93-101 行）

```python
meshing_session = pyfluent.launch_fluent(
    precision="double",
    processor_count=4,
    mode="meshing",
)
```

调用 `pyfluent.launch_fluent(...)` 启动 Fluent 软件。参数含义：

| 参数 | 值 | 含义 |
|------|-----|------|
| `precision` | `"double"` | 双精度计算（64 位浮点数），比单精度更精确 |
| `processor_count` | `4` | 使用 4 个 CPU 核心并行计算 |
| `mode` | `"meshing"` | 以**网格模式**启动（Fluent 有两种模式：meshing 画网格、solver 求解） |

返回值 `meshing_session` 是一个 **Fluent 会话对象**，你可以把它理解成"远程遥控器"——后续所有操作都通过这个对象向 Fluent 发指令。

```python
print(meshing_session.get_fluent_version())
```

`print()` 是 Python 的内置函数，把内容输出到屏幕上。
`.get_fluent_version()` 是会话对象的方法，获取当前 Fluent 的版本号。
所以这行代码的作用是：**在屏幕上打印 Fluent 版本**，确认启动成功。

```python
tmpdir = tempfile.mkdtemp()
meshing_session.preferences.MeshingWorkflow.TempFolder = tmpdir
```

- `tempfile.mkdtemp()`：创建一个临时目录（位置由系统决定），返回其路径
- 把 Fluent 网格工作流的**临时文件夹**设置为这个目录，这样中间文件不会散落在项目目录中

---

### 4. 初始化水密几何网格工作流（第 108 行）

```python
meshing_session.workflow.InitializeWorkflow(WorkflowType="Watertight Geometry")
```

- `meshing_session.workflow`：访问 Fluent 的"工作流"功能模块
- `.InitializeWorkflow(...)`：初始化一个工作流
- `WorkflowType="Watertight Geometry"`：指定工作流类型为**"水密几何"**——这是 Fluent 中最常用的网格工作流，适用于没有缝隙/孔洞的封闭几何体

---

### 5. 导入 CAD 几何（第 119-126 行）

```python
geo_import = meshing_session.workflow.TaskObject["Import Geometry"]
```

- `TaskObject["Import Geometry"]`：从工作流中获取名为 `"Import Geometry"` 的**任务对象**
- 工作流由一系列任务组成（导入几何 → 局部尺寸 → 面网格 → ...），每个任务都有名字

```python
geo_import.Arguments.set_state(
    {
        "FileName": wing_intermediary_file,
    }
)
```

- `.Arguments.set_state(...)`：设置这个任务的参数
- `{"FileName": wing_intermediary_file}`：字典，键 `"FileName"` 对应值 `wing_intermediary_file`（即之前下载的 `wing.pmdb` 的路径）
- **通俗理解**：告诉 Fluent "把这个文件导入进来"

```python
geo_import.Execute()
```

执行这个任务——真正开始导入几何。

---

### 6. 添加局部尺寸控制（第 132-165 行）

三组 `Add Local Sizing` 任务，分别在机翼的不同位置设置不同的网格密度。

**第一组**：机翼表面尺寸（wing-facesize）

```python
local_sizing = meshing_session.workflow.TaskObject["Add Local Sizing"]
local_sizing.Arguments.set_state({
    "AddChild": "yes",
    "BOIControlName": "wing-facesize",
    "BOIFaceLabelList": ["wing_bottom", "wing_top"],
    "BOISize": 10,
})
local_sizing.AddChildAndUpdate()
```

| 参数 | 值 | 含义 |
|------|-----|------|
| `AddChild` | `"yes"` | "我要新增一个子尺寸控制" |
| `BOIControlName` | `"wing-facesize"` | 给这个控制起个名字 |
| `BOIFaceLabelList` | `["wing_bottom", "wing_top"]` | 应用在哪些面上：机翼下表面和上表面 |
| `BOISize` | `10` | 目标网格尺寸 = 10 mm |

`AddChildAndUpdate()`：把新的尺寸控制添加到工作流中并更新。

**第二组**：机翼边缘尺寸（wing-edge-facesize）

```python
local_sizing.Arguments.set_state({
    "AddChild": "yes",
    "BOIControlName": "wing-ege-facesize",
    "BOIFaceLabelList": ["wing_edge"],
    "BOISize": 2,
})
local_sizing.AddChildAndUpdate()
```

- 机翼边缘处网格更密（2mm vs 10mm），因为边缘处的流动变化更剧烈

**第三组**：Body of Influence（boi_1）

```python
local_sizing.Arguments.set_state({
    "AddChild": "yes",
    "BOIControlName": "boi_1",
    "BOIExecution": "Body Of Influence",
    "BOIFaceLabelList": ["wing-boi"],
    "BOISize": 5,
})
local_sizing.AddChildAndUpdate()
```

- **BOI（Body Of Influence）**：一种影响体——在某个区域内逐渐细化网格，使附近的网格平滑过渡
- 这里 BOI 区域的网格尺寸为 5mm，介于翼面（2mm）和远场（更大）之间

---

### 7. 生成面网格（第 171-176 行）

```python
surface_mesh_gen = meshing_session.workflow.TaskObject["Generate the Surface Mesh"]
surface_mesh_gen.Arguments.set_state(
    {"CFDSurfaceMeshControls": {"MaxSize": 1000, "MinSize": 2}}
)
surface_mesh_gen.Execute()
```

- 获取"生成面网格"任务
- 设置参数：**最大网格尺寸 1000mm**，**最小网格尺寸 2mm**（机翼边缘处会更细）
- `Execute()`：开始生成面网格（这一步可能需要 2-5 分钟）

> **什么是面网格？** 在几何体的**表面**上划分网格单元（三角形/四边形），这是体网格的基础。只有先生成面网格，才能在内部填充体网格。

---

### 8. 描述几何并定义流体区域（第 182-191 行）

```python
describe_geo = meshing_session.workflow.TaskObject["Describe Geometry"]
describe_geo.UpdateChildTasks(SetupTypeChanged=False)
```

- 获取"描述几何"任务，先更新子任务列表

```python
describe_geo.Arguments.set_state(
    {"SetupType": "The geometry consists of only fluid regions with no voids"}
)
describe_geo.UpdateChildTasks(SetupTypeChanged=True)
```

- 告诉 Fluent："这个几何体**只包含流体区域，没有空腔**"
- 机翼外部是空气（流体），没有内部空腔
- `SetupTypeChanged=True`：通知 Fluent 设置类型已改变，需要重新更新子任务

```python
describe_geo.Execute()
```

---

### 9. 更新边界和区域（第 198-205 行）

```python
meshing_session.workflow.TaskObject["Update Boundaries"].Execute()
meshing_session.workflow.TaskObject["Update Regions"].Execute()
```

- **Update Boundaries**：自动识别边界类型（哪些面是入口、出口、壁面等）
- **Update Regions**：自动识别和命名流体区域

这两步都是自动化的——Fluent 根据几何特征智能分类。

---

### 10. 添加边界层网格（第 212-215 行）

```python
add_boundary_layer = meshing_session.workflow.TaskObject["Add Boundary Layers"]
add_boundary_layer.Arguments.set_state({"NumberOfLayers": 12})
add_boundary_layer.AddChildAndUpdate()
```

- **边界层（Boundary Layer）**：在固体壁面附近，流体速度从 0 急剧变化到来流速度，需要非常细密的网格来准确捕捉
- `NumberOfLayers: 12`：在壁面上堆叠 12 层逐渐增厚的网格
- 这 12 层的总厚度需要保证 `y+` 值适合 k-ω SST 湍流模型

---

### 11. 生成体网格（第 222-234 行）

```python
volume_mesh_gen = meshing_session.workflow.TaskObject["Generate the Volume Mesh"]
volume_mesh_gen.Arguments.set_state({
    "VolumeFill": "poly-hexcore",
    "VolumeFillControls": {"HexMaxCellLength": 512},
    "VolumeMeshPreferences": {
        "CheckSelfProximity": "yes",
        "ShowVolumeMeshPreferences": True,
    },
})
volume_mesh_gen.Execute()
```

| 参数 | 值 | 含义 |
|------|-----|------|
| `VolumeFill` | `"poly-hexcore"` | 体网格类型：多面体-六面体核心（边界层用多面体，内部用六面体，兼顾精度和效率） |
| `HexMaxCellLength` | `512` | 六面体核心区域最大网格尺寸 = 512 mm |
| `CheckSelfProximity` | `"yes"` | 检查网格自相交等质量问题 |

> 这一步是整个脚本中**最耗时**的部分（5-15 分钟），因为要在三维空间中生成数百万个网格单元。

---

### 12. 检查网格并保存（第 241-248 行）

```python
meshing_session.tui.mesh.check_mesh()
```

- `.tui` 是 Fluent 的**文本用户界面（Text User Interface）**——相当于在 Fluent 控制台中输入命令
- `mesh.check_mesh()` 等效于在 Fluent TUI 中输入 `/mesh/check-mesh`，检查网格质量（是否有负体积、扭曲度是否过大等）

```python
meshing_session.meshing.File.WriteMesh(FileName="wing.msh.h5")
```

- 将网格保存为 `wing.msh.h5`（HDF5 格式，Fluent 的高效网格存储格式）

---

### 13. 切换到求解模式（第 262 行）

```python
solver_session = meshing_session.switch_to_solver()
```

- 从**网格模式**切换到**求解模式**
- 返回一个新的会话对象 `solver_session`，后续用这个对象操作求解器
- 原来的 `meshing_session` 不再使用

---

### 14. 在求解模式下再次检查网格（第 272 行）

```python
solver_session.settings.mesh.check()
```

- 在求解模式下再次检查网格
- 这次检查会报告网格在 SI 单位（米）下的 x、y、z 范围，以及更多网格质量指标

---

### 15. 设置湍流模型（第 282-285 行）

```python
viscous = solver_session.settings.setup.models.viscous
viscous.model = "k-omega"
viscous.k_omega_model = "sst"
```

- 通过 `.settings.setup.models.viscous` 路径找到"粘性模型"设置
- 把模型设为 **k-ω SST**（Shear Stress Transport）
- **为什么选 k-ω SST？** 这是航空航天外流计算的标准模型，结合了 k-ω 在近壁面的精度和 k-ε 在远场的稳定性

---

### 16. 设置气体材料属性（第 299-311 行）

```python
air = solver_session.settings.setup.materials.fluid["air"]
```

- 从材料库中获取 `"air"`（空气）这个流体材料对象
- `["air"]` 是字典索引——流体材料是一个字典，键是材料名，值是材料对象

```python
air.density.option = "ideal-gas"
```

- 密度模型设为**理想气体**——这是可压缩流动的关键设置
- 理想气体意味着密度随压力和温度变化（ρ = p/RT），而不是常数
- 如果设为 `"constant"`，就是不可压缩流动

```python
air.viscosity.option = "sutherland"
air.viscosity.sutherland.option = "three-coefficient-method"
air.viscosity.sutherland.reference_viscosity = 1.716e-05
air.viscosity.sutherland.reference_temperature = 273.11
air.viscosity.sutherland.effective_temperature = 110.56
```

- 粘性模型设为 **Sutherland 定律**——粘度随温度变化
- **三系数法**：用三个参数定义 Sutherland 公式 μ = μ₀ × (T/T₀)^(3/2) × (T₀+S)/(T+S)

| 参数 | 值 | 物理意义 |
|------|-----|----------|
| `reference_viscosity` | `1.716e-05` kg/(m·s) | 参考温度下的动力粘度 μ₀ |
| `reference_temperature` | `273.11` K | 参考温度 T₀（≈ 0°C） |
| `effective_temperature` | `110.56` K | Sutherland 常数 S（有效温度） |

---

### 17. 设置边界条件（第 326-344 行）

```python
pressure_farfield = (
    solver_session.settings.setup.boundary_conditions.pressure_far_field[
        "pressure_farfield"
    ]
)
```

- 获取名为 `"pressure_farfield"` 的**压力远场边界条件**对象
- 括号 `(...)` 在这里只是换行作用，不影响逻辑
- 压力远场用于模拟"无限远处的来流"——是外流 CFD 的标准边界条件

```python
pressure_farfield.momentum.gauge_pressure = 0
```

- 表压 = 0（相对于操作压力）

```python
pressure_farfield.momentum.mach_number = 0.8395
```

- 来流马赫数 M = 0.8395（跨声速，约 288 m/s）

```python
pressure_farfield.thermal.temperature = 255.56
```

- 来流温度 255.56 K（约 -17.6°C，这是对应 11000m 巡航高度的标准大气温度）

```python
pressure_farfield.momentum.flow_direction[0] = 0.998574   # x 分量
pressure_farfield.momentum.flow_direction[2] = 0.053382   # z 分量
```

- 用方向向量定义攻角 α = 3.06°：
  - x 分量 = cos(3.06°) = 0.998574（几乎平行于机翼弦向）
  - z 分量 = sin(3.06°) = 0.053382（向上的分量）
  - y 分量保持默认 0（无侧滑角）

```python
pressure_farfield.turbulence.turbulent_intensity = 0.05
pressure_farfield.turbulence.turbulent_viscosity_ratio = 10
```

- 湍流强度 = 5%（外流场的典型值）
- 湍流粘度比 = 10（湍流粘度 / 层流粘度的比值）

---

### 18. 设置操作条件（第 353 行）

```python
solver_session.settings.setup.general.operating_conditions.operating_pressure = 80600
```

- 操作压力 = 80600 Pa（约 0.795 atm）
- 这对应高空巡航高度的大气压力
- Fluent 中的压力 = 操作压力 + 表压（gauge pressure）

---

### 19. 初始化流场（第 360 行）

```python
solver_session.settings.solution.initialization.hybrid_initialize()
```

- **Hybrid Initialization（混合初始化）**：Fluent 的智能初始化方法
- 它结合了多种初始化策略，自动为整个流场生成合理的初始值
- 好的初始值能让求解更快收敛

---

### 20. 保存初始 case 文件（第 367-369 行）

```python
solver_session.settings.file.write(
    file_name="external_compressible.cas.h5", file_type="case"
)
```

- 保存**计算前的 case 文件**（包含网格 + 所有设置，但不含计算结果）
- 这是好习惯——如果后续计算出问题，可以从这里重新开始

---

### 21. 迭代求解（第 376 行）

```python
solver_session.settings.solution.run_calculation.iterate(iter_count=25)
```

- 开始迭代计算，共 **25 次迭代**
- Fluent 会在每次迭代后输出残差（方程误差），残差越小说明收敛越好
- 官方建议 100 次迭代，但这个例子里 25 次足以看到收敛趋势

---

### 22. 保存最终结果（第 383-385 行）

```python
solver_session.settings.file.write(
    file_name="external_compressible1.cas.h5", file_type="case"
)
```

- 保存**计算后的 case 文件**（包含网格 + 设置 + 计算结果）

---

### 23. 清理与退出（第 392-395 行）

```python
solver_session.exit()
```

- 关闭 Fluent 求解器进程，释放计算资源

```python
shutil.rmtree(tmpdir, ignore_errors=True)
shutil.rmtree("wing_workflow_files", ignore_errors=True)
```

- `shutil.rmtree(path)`：递归删除整个目录（类似 Windows 的 Shift+Delete）
- `ignore_errors=True`：如果目录不存在或删除失败，不报错，继续执行
- 删除临时目录和工作流中间文件，保持项目目录整洁

---

### 总结：脚本整体流程

```
┌─────────────────────────────────────────────┐
│  1. 下载几何文件 (wing.scdoc, wing.pmdb)      │
│  2. 启动 Fluent（网格模式，4核，双精度）       │
│  3. 初始化"水密几何"工作流                     │
│  4. 导入 CAD → 局部尺寸 → 面网格               │
│  5. 描述几何 → 更新边界/区域 → 边界层 → 体网格  │
│  6. 保存网格 (wing.msh.h5)                    │
├─────────────────────────────────────────────┤
│  7. 切换到求解模式                             │
│  8. 设置 k-ω SST 湍流模型                      │
│  9. 设置理想气体 + Sutherland 粘性              │
│ 10. 设置压力远场边界条件 (M=0.8395, α=3.06°)   │
│ 11. 混合初始化 → 迭代 25 步                     │
│ 12. 保存结果 → 退出 → 清理临时文件              │
└─────────────────────────────────────────────┘
```

---

## 修改参数实验

如果你想修改工况做自己的研究，以下是关键参数的位置：

| 参数 | 代码位置 | 默认值 |
|------|----------|--------|
| 马赫数 | `pressure_farfield.momentum.mach_number` | 0.8395 |
| 攻角 | `pressure_farfield.momentum.flow_direction` (x, z 分量) | (0.998574, 0.053382) = 3.06° |
| 来流温度 | `pressure_farfield.thermal.temperature` | 255.56 K |
| 操作压力 | `operating_conditions.operating_pressure` | 80600 Pa |
| 迭代次数 | `iterate(iter_count=25)` | 25（官方推荐 100） |
| 边界层层数 | `NumberOfLayers` | 12 |
| 网格最小尺寸 | `MinSize` | 2 mm |
| 网格最大尺寸 | `MaxSize` | 1000 mm |

修改攻角的快速公式：
```python
import math
alpha = 3.06  # 攻角（度）
x_component = math.cos(math.radians(alpha))   # 0.998574
z_component = math.sin(math.radians(alpha))   # 0.053382
```

---

## 输出文件

运行完成后，在脚本所在目录会生成：

| 文件 | 说明 |
|------|------|
| `wing.msh.h5` | 网格文件（HDF5 格式） |
| `external_compressible.cas.h5` | 初始 case 文件 |
| `external_compressible1.cas.h5` | 最终 case 文件（含计算结果） |
| `wing_workflow_files/` | 工作流中间文件（脚本结束时自动删除） |

---

## 参考链接

- [PyFluent 官方文档](https://fluent.docs.pyansys.com/)
- [ONERA M6 机翼 - NASA 验证算例](https://www.grc.nasa.gov/WWW/wind/valid/m6wing/m6wing.html)
- [k-ω SST 湍流模型 - Menter 1994](https://turbmodels.larc.nasa.gov/sst.html)
