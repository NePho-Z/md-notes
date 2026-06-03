# 1.资料查询阶段（目标，了解应用程序的结构，了解godot相关知识，了解塔二mod工作原理，反编译一下塔二学习

- https://www.bilibili.com/video/BV17bwFznEvj/?spm_id_from=333.337.search-card.all.click&vd_source=a852e00ed73fdc83fda7efcf46560ee4反编译
- https://www.bilibili.com/video/BV1YENgzUED3/?spm_id_from=333.337.search-card.all.click&vd_source=a852e00ed73fdc83fda7efcf46560ee4模板mod制作
  - https://github.com/Alchyr/ModTemplate-StS2?tab=readme-ov-file github版本

## 塔二的本地文件

- 软件的基本原理
- 你这个问题非常好，这张图基本就是一个 Windows 游戏/应用的“发布目录”典型结构。按你图里的文件名，大致可以这样理解：
- 1. `SlayTheSpire2`（应用程序）
- 这是主程序，通常是 `.exe`。
- 双击它时，Windows 会先加载这个可执行文件，然后再按需加载它依赖的动态库（DLL）和资源文件。
- 2. `*.dll`（动态链接库）
- 例如 `fmod.dll`、`fmodstudio.dll`、`libsentry...dll`。
- DLL 不是单独启动的软件，而是“功能模块库”。主程序运行时调用里面的函数。

### 你图里这些很像：

- `fmod*.dll`：音频引擎相关（播放音乐/音效）。
- `libsentry...dll`：崩溃上报/错误监控相关。
- `libspine...dll`：2D 骨骼动画（Spine）相关。
- 用 DLL 的好处：模块化、可复用、升级某模块时不必重编整个程序。
- 3. `crashpad_handler`、`crashpad_wer.dll`
- 崩溃收集组件。程序崩溃时，它会抓取崩溃信息并可能上报，方便开发者定位问题。
- 4. `launch_d3d12`、`launch_opengl`、`launch_vulkan`（批处理）
- 这是启动脚本（通常是 `.bat`）。
- 作用是指定图形后端启动（Direct3D 12 / OpenGL / Vulkan），常用于兼容性或调试。
- 5. `SlayTheSpire2.pck`（资源包）
- 游戏资源打包文件（贴图、音频、场景、脚本资源等）。
- 和可执行程序分离，便于更新资源、减小主程序复杂度。
- `.pck` 这个后缀在 Godot 生态里很常见。
- 6. `data_sts2_windows_x86_64`（目录）
- 通常放更多运行时资源、配置、附加数据、可能的引擎子文件。
- 7. `release_info`（JSON）

### 发布元数据，常见内容有版本号、构建号、渠道信息、时间戳等，供程序或启动器读取。

- 为什么一个软件会由这么多文件构成
- 1. 功能拆分：主程序 + 各模块库（音频、渲染、网络、日志、崩溃上报）。
- 2. 资源分离：代码和素材不是一回事，资源通常单独打包。
- 3. 依赖复用：第三方库以 DLL 形式提供，避免把所有代码都塞进一个 EXE。
- 4. 更新效率：改一个模块或资源包时，不必替换全部文件。

### 5. 平台运行机制：Windows 本身就支持 EXE + DLL 的动态加载模型。

- 后缀名你可以先这样记
- 1. `.exe`：可执行程序（入口）。
- 2. `.dll`：动态链接库（运行时被 EXE 加载）。
- 3. `.bat`：批处理脚本（命令集合）。
- 4. `.json`：结构化文本配置/元数据。
- 5. `.pck`：打包资源文件（常见于游戏引擎）。

### 6. 无后缀的可执行文件：有些工具也可能不显示 `.exe`，取决于打包方式和资源管理器设置。

- 代码是怎么变成这些文件的（简化流程）
- 1. 写源码
- C/C++/C#/脚本 + 资源（图片、音频、模型）。
- 2. 编译
- 编译器把源码变成目标文件（机器码中间产物）。
- 3. 链接

### 把目标文件和库链接成最终产物：

- 主程序（EXE）
- 动态库（DLL）或静态库（如果选择静态链接）
- 4. 拷贝运行时依赖
- 把第三方 DLL、引擎 DLL、运行时组件放到发布目录。
- 5. 打包资源
- 引擎工具把素材压缩/打包成 `.pck`、`.pak` 等。
- 6. 生成启动器与元信息
- 生成 `.bat`、`release_info.json` 等辅助文件。
- 7. 形成最终发布目录

### 你看到的就是“可分发版本”，不是源码工程目录。

- 如果你愿意，我可以基于你这张图再给你做一个“启动时序图”（双击 EXE 后，Windows 按什么顺序找 DLL、加载资源、进入主循环），看完你会更直观理解这些文件为什么缺一不可。

## 2.尝试反编译，学习goddot，了解塔二mod工作模式

- 塔二用的godot版本4.5.1monowin64

### godot不识别.net问题，无法识别通常是系统环境变量配置问题

- 按下面做就行，Windows 10/11 通用。
- 1. 打开环境变量窗口
- 按 `Win` 键，搜索“环境变量”
- 点“编辑系统环境变量”
- 在“系统属性”里点“环境变量(N)...”
- 2. 修改 `Path`

#### 你会看到两块：

- “用户变量”（只对你当前账号生效）
- “系统变量”（对所有用户生效，需要管理员权限）
- 选中 `Path` -> 点“编辑”

#### 点“新建”，添加（推荐 x64）：

- `C:\Program Files\dotnet\`

#### 如果你之前只装了 x86，也可能有：

- `C:\Program Files (x86)\dotnet\`
- 建议把 x64 路径放在更靠前位置（用“上移”按钮）。
- 3. 保存并重开终端
- 一路“确定”保存
- 关闭所有 PowerShell/CMD/VS Code，再重新打开
- 旧终端不会自动更新 PATH

#### 4. 验证是否生效

- 在新 PowerShell 执行：
- `where.exe dotnet`

#### `dotnet --info`

- 如果仍提示无法识别，先确认该目录里真的有 `dotnet.exe`。
- 资源管理器直接打开：

#### `C:\Program Files\dotnet\` 看是否存在 `dotnet.exe`

- 给你一个更快的方法（命令行，当前用户）：
- powershell
- setx PATH "$env:PATH;C:\Program Files\dotnet\"
- 执行后也要“新开终端”才生效。
- 不过我更推荐图形界面，直观不容易配错。
- 如果你愿意，我可以下一条直接给你“最保险的 x64-only 清理方案”（先删哪些 PATH，再加哪些），避免 x86/x64 混用引发后续问题。
- x86代表32位操作系统x64代表64位操作系统
- https://github.com/GDRETools/gdsdecomp godot反编译软件
- github里可以直接在右边的release里下载想用的软件
- 塔二mod原理https://www.bilibili.com/opus/1178964343800201223
- godot文字教程https://kidscancode.org/godot_recipes/4.x/

## 2.godot学习

- 3.塔二源码分析
