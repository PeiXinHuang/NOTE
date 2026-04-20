## Unity 热更新技术方案全解析## 一、 主流热更新方案对比
目前 Unity 行业内主要存在四种技术路线，具体特性如下表：

| 方案 | 技术原理 | 开发语言 | 性能 | 优缺点 |
|---|---|---|---|---|
| HybridCLR | 底层修改 IL2CPP 运行时 | 原生 C# | 极高 (接近原生) | 优点： 零学习成本，支持全特性。 缺点： 内存占用相对较大。 |
| Lua (XLua/ToLua) | 集成 Lua 虚拟机 | Lua | 中 | 优点： 极其成熟，稳定性高。 缺点： 需要维护两套语言，跨域调用开销大。 |
| ILRuntime | 应用层 C# 解释器 | C# | 低 | 优点： 无需修改底层，纯 C# 编写。 缺点： 性能较差，已逐渐被 HybridCLR 取代。 |
| Puerts | 集成 V8/QuickJS 引擎 | TS/JS | 高 | 优点： 适合 Web 背景团队，生态丰富。 缺点： 跨域开销与 Lua 类似。 |

------------------------------
## 二、 HybridCLR 核心接入流程
HybridCLR 通过扩展 IL2CPP，使其支持动态加载程序集（Assembly）。
## 1. 环境安装

* 通过 Unity Package Manager 导入 HybridCLR 包。
* 执行 HybridCLR -> Installer，自动下载并魔改 libil2cpp 源码。

## 2. 程序集划分

* AOT 程序集：主程序、SDK、HybridCLR 初始化代码。
* 热更程序集：游戏业务逻辑（需创建为独立的 .asmdef）。

## 3. 编码与加载

   1. 补充元数据：调用 RuntimeApi.LoadMetadataForAOTAssembly 加载 AOT 程序的元数据，解决泛型实例化问题。
   2. 加载 DLL：使用 System.Reflection.Assembly.Load 加载热更 DLL 的字节流。
   3. 启动逻辑：通过反射获取热更入口方法并执行。

## 4. 打包发布

* 执行 CompileDll 生成热更 DLL。
* 将 DLL 作为资源（如 Addressables）上传至服务器。

------------------------------
## 三、 Assembly（程序集）概念解析

* 定义：C# 代码编译后的逻辑单元（通常为 .dll），包含 IL 代码、元数据 (Metadata) 和 清单 (Manifest)。
* 热更中的角色：在 HybridCLR 中，热更代码被封装在一个独立的 Assembly 中，运行时由底层解释器动态解析。
* 内存痛点：Assembly 中的元数据描述了类、方法和继承关系。在加载时，HybridCLR 需要解析并在内存中重建这些索引，这在内存敏感平台（如微信小游戏）会产生较大的内存压力。

------------------------------
## 四、 针对内存压力的魔改与优化思路
针对社区版 HybridCLR 元数据内存占用高的问题，可尝试以下优化手段：
## 1. 元数据裁剪 (Stripping)

* 手段：利用 Mono.Cecil 在编译后扫描热更 DLL。
* 操作：剔除运行时不需要的 CustomAttribute、参数名称（ParameterName）等反射信息。
* 效果：有效缩减 DLL 体积及解析后的内存占用。

## 2. 精准 AOT 元数据补充 (Differential Metadata)

* 问题：全量加载 AOT 元数据（如 mscorlib.dll）会占用大量内存。
* 优化：静态分析热更代码中实际用到的 AOT 泛型实例，仅提取并加载必要的元数据片段，生成“瘦身版”元数据包。

## 3. 延迟解析 (Lazy Loading)

* 魔改点：修改 HybridCLR 底层 C++ 代码中的 Image 解析类。
* 做法：将原本加载时一次性解析的类成员信息，改为在代码第一次运行（Resolve）时按需解析。

## 4. 字符串常量池优化

* 手段：在 C++ 层实现全局字符串去重（Interning）。
* 效果：多个程序集中重复的类名、方法名字符串指向同一块内存，减少冗余。

## 5. 数据结构压缩

* 操作：优化底层 Metadata 索引的数据结构，将通用型容器（如 std::unordered_map）替换为高密度存储的专用 FlatMap。



