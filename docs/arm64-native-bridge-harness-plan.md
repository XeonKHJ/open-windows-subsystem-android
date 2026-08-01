# ARM64 Native Bridge 研发实现文档（Harness）

> 项目：Open Windows Subsystem Android（OWSA）  
> 目标平台：Windows x86_64 → Hyper-V → Android x86_64 Guest  
> 文档状态：研发基线  
> 范围：仅定义 ARM64 (`arm64-v8a`) Native Bridge / 动态二进制翻译器（DBT）的研发与验收计划。

---

## 1. 目标与非目标

### 1.1 最终研发目标

在 Android x86_64 Guest 中运行 Android APK：

- Java/Kotlin/DEX 由 Guest 内的 x86_64 ART 正常执行；
- 若 APK 带有 `lib/x86_64/*.so`，直接加载和执行；
- 若 APK **仅**带有 `lib/arm64-v8a/*.so`，通过自研 Android Native Bridge 与 ARM64→x86_64 动态二进制翻译器执行其 native code；
- 不把整个 Android Guest 模拟为 ARM64，也不翻译 Java/Kotlin/DEX。

```text
Windows x86_64
└── Hyper-V
    └── Android x86_64 Guest
        ├── ART：执行 DEX（x86_64）
        ├── x86_64 native .so：直接执行
        └── arm64-v8a native .so
            └── OWSA Native Bridge
                └── ARM64 decoder / interpreter / JIT
                    └── x86_64 machine code
```

### 1.2 首个可验收闭环（Research MVP）

一个仅携带 `arm64-v8a/libdemo.so` 的测试 APK 能在 x86_64 Android Guest 中：

1. 由 Java/Kotlin 调用 `System.loadLibrary("demo")`；
2. 由 AOSP Native Bridge 将库加载请求交给 `libowsa_nativebridge.so`；
3. 调用 ARM64 JNI 函数 `nativeAdd(40, 2)`；
4. 经解释器或 JIT 执行 ARM64 指令；
5. 正确返回 `42`；
6. 失败时导出可定位的诊断信息（APK、ABI、ELF、ARM PC、指令、寄存器和宿主 backtrace）。

### 1.3 明确非目标（第一阶段）

- `armeabi-v7a`、ARM32、Thumb；
- 承诺兼容所有 ARM APK；
- Unity/Unreal、大型游戏、DRM、反作弊、虚拟化规避；
- Vulkan、复杂 OpenGL ES 与完整媒体加速；
- ARM native JIT/self-modifying code；
- Google Play 兼容认证；
- 直接运行 ARM64 Android Guest。

---

## 2. 设计原则

1. **先正确，后性能**：先解释器，后 JIT；解释器是 JIT 的语义参考实现。
2. **仅 ARM64**：所有 guest native context、指针与 host ABI 保持 64 位。
3. **最小集成面**：使用 AOSP Native Bridge 作为 ART/Linker 集成边界，而非改造 ART 执行 DEX 的基本模型。
4. **可诊断优先**：每个 bridge、ELF loader、decoder 和 JIT 失败都必须携带结构化错误上下文。
5. **安全优先**：JIT code cache 遵循 W^X；禁止长期 RWX 页；不信任 APK native code。
6. **可重复实验**：固定 AOSP revision、Android build target、内核、Hyper-V 配置、测试 APK 和测试结果。
7. **范围门禁**：一个阶段的 Gate 未通过，不开始下一阶段的高风险能力。

---

## 3. 系统边界与组件

```text
+-------------------------------------------------------------+
| Android x86_64 process                                       |
|                                                             |
|  ART / JNI                                                  |
|     |                                                       |
|  AOSP libnativebridge                                        |
|     | NativeBridgeItf                                        |
|  libowsa_nativebridge.so                                    |
|     ├── Adapter / ABI policy                                 |
|     ├── Foreign ELF loader + linker                          |
|     ├── ARM64 interpreter                                    |
|     ├── ARM64 IR + x86_64 JIT                                |
|     ├── JNI trampolines                                      |
|     ├── Thread / TLS / signal runtime                        |
|     └── Diagnostics / compatibility database                 |
|                                                             |
|  Android system libraries / linker namespaces                |
+-------------------------------------------------------------+
```

### 3.1 主要模块

| 模块 | 责任 | 初始阶段 |
|---|---|---|
| `adapter/` | 实现 Native Bridge 回调、ABI 识别、加载入口和 JNI trampoline 入口 | R1 |
| `loader/` | ELF64/AArch64 映射、动态段、符号、relocation、依赖库加载 | R2 |
| `interpreter/` | ARM64 decode 与精确执行语义 | R2 |
| `jit/` | IR、x86_64 codegen、block cache、链接和回退 | R3 |
| `runtime/` | 线程、TLS、signals、futex、atomics、JNI 回调 | R2–R4 |
| `compatibility/` | 支持矩阵、应用 workaround、已知问题及回归案例 | R4–R5 |
| `tests/` | unit、native、APK、integration、stress、differential tests | R0 起持续 |

---

## 4. Harness 执行模型

每个阶段必须按以下 Harness 字段执行：

| 字段 | 含义 |
|---|---|
| **Input** | 本阶段开始前必须存在的条件或材料 |
| **Work** | 必须完成的可审查任务 |
| **Output** | 提交到仓库的代码、文档、测试或二进制产物 |
| **Evidence** | 可复现的日志、命令输出、测试报告或性能数据 |
| **Gate** | 进入下一阶段的硬性验收条件 |
| **Rollback / Stop rule** | 失败时的回退或暂停条件 |
| **Risks** | 需要持续追踪的技术风险 |

### 4.1 全局质量门禁

- 每个新功能必须带最小自动化测试；
- 每次 AOSP 版本或 ABI 接口变更必须重新跑完整 smoke suite；
- 任何 native crash 都必须生成可关联的 crash ID；
- 不允许将“未支持指令”伪装为普通 JNI 返回值；
- 不允许以关闭 SELinux、放宽内存保护或长期 RWX 作为可接受的正式方案；
- 支持状态必须分类为：`supported`、`experimental`、`blocked`、`unknown`。

---

## 5. 里程碑与阶段计划

## M0 / R0：实验平台与测试基线

### Objective

建立可重复的 Android x86_64 Native Bridge 实验环境，并提供 x86_64 对照库与 ARM64 目标库。

### Input

- 可访问的 Android x86_64 AOSP 源码与构建环境；
- 可重复启动的 Hyper-V Android x86_64 Guest；
- ADB、`logcat`、tombstone 收集链路；
- 固定的构建主机工具链版本。

### Work

1. 固定 AOSP revision、build target、kernel revision 与 Hyper-V VM 基线；
2. 建立 `nativebridge-smoke` APK：Java/Kotlin 调用层保持一致；
3. 构建三个 native 变体：
   - `lib/x86_64/libdemo.so`：对照组；
   - `lib/arm64-v8a/libdemo.so`：翻译目标；
   - 可选 `host` 单元测试库：脱离 Android 验证算法；
4. 实现测试命令与日志归档脚本；
5. 文档化 build、boot、install、run、collect 的完整流程。

### Output

```text
nativebridge/
  tests/apk/nativebridge-smoke/
  tests/native/demo/
  docs/environment-baseline.md
  docs/test-protocol.md
```

### Evidence

- Android Guest 的启动日志；
- `adb install` 成功记录；
- x86_64 对照库返回 `nativeAdd(40, 2) = 42` 的测试输出；
- `logcat`、tombstone、宿主日志的归档样本。

### Gate

- 在干净 VM 快照上连续 10 次完成：启动 Guest → 安装 APK → 运行 x86_64 对照 JNI 测试；
- 成功率至少 9/10；
- 所有失败可由日志定位到 boot、ADB、APK、ART 或 native crash 中的一个分类。

### Rollback / Stop rule

- Guest 启动或 APK 基线不稳定时，冻结 Native Bridge 开发；
- 仅修复平台复现性，直到 Gate 通过。

### Risks

- AOSP x86_64 / Hyper-V 设备适配不稳定；
- 构建环境漂移；
- 测试 APK 同时携带多个 ABI 时的 ABI 选择不可控。

---

## M1 / R1：Native Bridge Adapter Stub

### Objective

验证 AOSP `libnativebridge`、动态 linker、ART 和 OWSA bridge 的实际控制流已正确接通；本阶段不执行 ARM 指令。

### Input

- R0 Gate 已通过；
- 目标 AOSP 分支的 `NativeBridgeItf` 接口定义已冻结并记录；
- `nativebridge-smoke` 的 ARM64 库样本。

### Work

1. 创建 `libowsa_nativebridge.so`；
2. 实现与目标 AOSP 版本匹配的 Native Bridge adapter 回调；
3. 实现 `initialize`、ABI/路径支持判断、库加载入口、trampoline 查询入口及必要的 namespace 回调；
4. 对 AArch64 ELF 作最小识别：magic、class、endianness、machine、ABI；
5. 建立结构化日志：bridge version、process、package、library、requested symbol、ABI、错误码；
6. 对未实现执行路径返回**明确、可测试**的失败状态，禁止 ART 崩溃或静默加载错误库。

### Output

```text
nativebridge/
  Android.bp
  adapter/native_bridge_adapter.cc
  include/nb_adapter.h
  docs/nativebridge-integration.md
  tests/integration/adapter_stub_test.sh
```

### Evidence

- Bridge 初始化日志；
- `arm64-v8a/libdemo.so` 被识别为 foreign ABI 的记录；
- ART/Native Bridge 对 JNI symbol 请求到达 adapter 的日志；
- 受控 “not implemented” 失败的测试结果。

### Gate

- 对照 x86_64 库继续直接执行；
- ARM64 库的加载与符号请求能稳定进入 OWSA adapter；
- 连续 100 次执行不发生 ART、linker 或 zygote 崩溃；
- 错误码和日志字段由自动化测试验证。

### Rollback / Stop rule

- 若 Native Bridge 接口在 AOSP 分支上无法稳定触发，先完成集成调查；
- 不在未确认接口版本与调用顺序前开始 ELF loader/JIT 工作。

### Risks

- AOSP 分支间接口和系统属性行为差异；
- Package Manager 的 ABI 选择与 linker 行为不一致；
- namespace 语义被低估。

---

## M2 / R2：ARM64 ELF Loader 与解释器闭环

### Objective

以解释器实现第一个正确的 Java → JNI → ARM64 `.so` → Java 闭环。

### Input

- R1 Gate 已通过；
- ARM64 `libdemo.so` 已限制在受控指令与依赖范围；
- ARM64 ABI、ELF relocation、JNI 调用约定的测试用例。

### Work

1. 实现 ELF64/AArch64 loader：
   - `PT_LOAD` 映射；
   - dynamic section；
   - `.dynsym` / `.dynstr`；
   - 基础 symbol resolution；
   - 最小 relocation 子集；
   - 显式页权限管理。
2. 实现 ARM64 interpreter：
   - 通用寄存器和 `SP` / `PC` 状态；
   - 基础整数运算、比较、条件分支；
   - load/store；
   - call/return；
   - 常用 PC-relative addressing；
   - AAPCS64 参数与返回值规则。
3. 实现最小 JNI trampoline：
   - Java → ARM64 JNI；
   - ARM64 返回 primitive values；
   - 明确限制的对象引用策略。
4. 实现 trap/unsupported-instruction 报告与寄存器快照；
5. 为每条已支持指令提供 unit test，为测试库提供端到端 APK test。

### Output

```text
nativebridge/
  loader/elf64.cc
  loader/reloc_aarch64.cc
  interpreter/decode_aarch64.cc
  interpreter/execute_aarch64.cc
  runtime/thread_context.cc
  runtime/jni_trampoline_x86_64.S
  docs/supported-instructions.md
  docs/elf-loader-contract.md
```

### Evidence

- `nativeAdd(40, 2) == 42` 的 APK 测试报告；
- 每条目标 instruction 的 decode/execute 单测；
- 未支持指令的诊断样本；
- loader 映射、relocation 和 symbol resolution trace。

### Gate

- Research MVP 闭环通过；
- 最少 10 个 ARM64 interpreter 单测、5 个 ELF loader 单测、3 个 APK end-to-end 场景全部通过；
- 同一测试连续运行 1,000 次无结果偏差、无 native crash；
- 未支持指令只能以受控诊断失败，不能静默错误执行。

### Rollback / Stop rule

- 若 loader、JNI trampoline 和 interpreter 的缺陷无法区分，则暂停扩充指令集；
- 补齐单元测试和 trace 后才继续。

### Risks

- relocation 和 PLT/GOT 细节；
- Java/JNI/x86_64/ARM64 双 ABI 边界；
- foreign library 依赖宿主 Bionic 符号时的语义差异；
- 指针、对象引用和 GC 生命周期。

---

## M3 / R3：IR、x86_64 JIT 与 Code Cache

### Objective

在保留解释器 fallback 的前提下，为热点 ARM64 basic block 生成 x86_64 机器码。

### Input

- R2 Gate 已通过；
- interpreter 已作为语义参考实现；
- 基准程序能识别热点循环与调用模式。

### Work

1. 定义可测试的 ARM64 → IR 语义；
2. 实现 basic block 翻译：decode → IR → x86_64 codegen；
3. 实现 code cache、block lookup、block chaining 和间接分支缓存；
4. 实现 W^X：代码页写入后变为可执行；
5. 提供 runtime 开关：`interpreter`、`jit`、`jit-verify`；
6. 在 `jit-verify` 模式中，对比解释器和 JIT 的返回值、寄存器与可见内存副作用；
7. 建立 benchmark 基线并记录性能变化。

### Output

```text
nativebridge/
  jit/ir.cc
  jit/ir.h
  jit/x86_64_codegen.cc
  jit/code_cache.cc
  jit/block_linker.cc
  tests/differential/
  docs/jit-design.md
  docs/code-cache-security.md
```

### Evidence

- interpreter/JIT differential test 报告；
- W^X 检查结果；
- 热点基准的执行时间、翻译次数、cache hit rate；
- JIT fallback 发生原因统计。

### Gate

- R2 的所有测试在 interpreter 与 JIT 两种模式均通过；
- `jit-verify` 对目标测试集无语义差异；
- 热点循环较解释器获得可测量加速；
- 无长期 RWX 内存映射；
- 发生 JIT 失败时能安全回退 interpreter。

### Rollback / Stop rule

- 任何 semantic mismatch 都阻断 JIT 范围扩展；
- 回退到解释器执行，并为该 block 记录禁用原因。

### Risks

- x86_64 flags、memory ordering 和 ARM64 语义不匹配；
- 代码缓存生命周期；
- 间接跳转与异常路径；
- 性能优化掩盖正确性问题。

---

## M4 / R4：Android Runtime 兼容层

### Objective

从单一受控 `.so` 扩展到普通 JNI 库常见的依赖、线程和动态加载行为。

### Input

- R3 Gate 已通过；
- 一组经过分类的真实或准真实 ARM64 JNI 测试库；
- 已记录的 Android linker namespace 规则。

### Work

1. 支持 foreign `.so` 的 `DT_NEEDED` 依赖图；
2. 实现受控 `dlopen()`、`dlsym()`、`dlclose()`；
3. 实现 linker namespace 和系统库代理策略；
4. 补齐 TLS、pthread、futex、atomics 语义；
5. 实现 signal/crash translation；
6. 支持 JNI `RegisterNatives` 和 native → Java callback；
7. 实现 native thread attach/detach ART；
8. 建立兼容性状态数据库与回归测试样本。

### Output

```text
nativebridge/
  loader/foreign_linker.cc
  runtime/tls.cc
  runtime/signals.cc
  runtime/pthread_bridge.cc
  runtime/jni_callbacks.cc
  compatibility/packages.yaml
  docs/runtime-compatibility.md
```

### Evidence

- 主库 `dlopen` 次级 ARM64 库的测试；
- 多线程与 TLS stress test；
- JNI 动态注册与 callback 测试；
- crash translation report；
- 每个支持/阻断案例的 compatibility entry。

### Gate

- 多库加载、动态符号解析、线程和 JNI callback 场景连续 1,000 次通过；
- 对不支持的 C++ exceptions、self-modifying code 等行为有明确检测和拒绝路径；
- 每个已知崩溃具有最小复现、分类和回归状态。

### Rollback / Stop rule

- 未能保证 TLS/signals/atomics 的正确性时，不对外宣称多线程支持；
- 将相关 APK 状态标记为 `blocked`，避免无诊断 crash。

### Risks

- Android linker namespace 行为与目标 AOSP 版本耦合；
- signal 与 ART GC/线程协作；
- 原子内存序与高并发测试覆盖不足；
- 复杂 C++ runtime 依赖。

---

## M5 / R5：NEON、图形路径与真实应用扩展

### Objective

有选择地提高常用 ARM64 native 库的兼容性和性能，并建立长期维护机制。

### Input

- R4 Gate 已通过；
- 应用样本的合法测试授权与版本固定；
- 性能与稳定性监控能力。

### Work

1. 按真实样本统计优先实现 NEON 整数，再实现常用浮点指令；
2. 扩展图像、音频等计算型 JNI 库测试；
3. 评估 OpenGL ES、媒体和 Vulkan 需求，分别建立独立 RFC；
4. 建立 package-level compatibility rules；
5. 建立 crash telemetry（默认本地、可选择上报）与去敏策略；
6. 定期运行兼容性、性能和安全回归。

### Output

```text
nativebridge/
  interpreter/neon_*.cc
  jit/neon_lowering_*.cc
  compatibility/workarounds/
  tests/real-apk/
  docs/compatibility-policy.md
  docs/telemetry-privacy.md
```

### Evidence

- 每新增指令或 workaround 的回归测试；
- 应用样本成功、失败与性能矩阵；
- 内存、启动时间、翻译 cache、崩溃率趋势；
- 安全审查记录。

### Gate

- 不以“应用能启动”作为唯一成功标准；必须验证功能、稳定性、资源使用和退出/恢复；
- 每个 supported 应用有固定版本、测试步骤和自动化或半自动化回归；
- 发布说明明确列出 supported / experimental / blocked 范围。

### Rollback / Stop rule

- 新 NEON/JIT 优化引入语义回归时，按 instruction、package 或 feature flag 禁用；
- 图形/Vulkan 工作未获独立设计评审前不得与核心 translator 耦合。

### Risks

- 应用特定 workaround 无限增长；
- 图形、媒体、DRM 和反作弊带来超出 Native Bridge 范围的问题；
- 性能目标与正确性/安全目标冲突。

---

## 6. 工作项分解（初始 Backlog）

### P0：立即开始

- [ ] 记录 AOSP revision、build target、内核与 Hyper-V VM 基线；
- [ ] 创建 `nativebridge/` 目录和 Android.bp 骨架；
- [ ] 创建 `nativebridge-smoke` APK 与 x86_64/arm64-v8a `libdemo.so`；
- [ ] 增加一键 install/run/log collection 脚本；
- [ ] 定义结构化 diagnostics schema；
- [ ] 调研并记录目标 AOSP 的 Native Bridge 接口、属性和加载顺序；
- [ ] 实现 `libowsa_nativebridge.so` adapter stub；
- [ ] 添加 adapter stub integration test。

### P1：R2 前置

- [ ] 定义 ARM64 context 和 AAPCS64 调用约定测试；
- [ ] 实现 ELF header/program header/dynamic section parser；
- [ ] 实现最小 `PT_LOAD` memory mapping；
- [ ] 定义 relocation 支持清单；
- [ ] 实现 ARM64 decode table 的最小整数子集；
- [ ] 实现 interpreter register/memory model；
- [ ] 实现 Java → ARM64 JNI primitive trampoline；
- [ ] 添加 unsupported instruction trap 测试。

### P2：R3 前置

- [ ] 编写 IR 设计 RFC；
- [ ] 实现 basic block boundary analysis；
- [ ] 实现 x86_64 code emitter；
- [ ] 实现 W^X code cache；
- [ ] 实现 interpreter/JIT differential harness；
- [ ] 添加 benchmark 与性能预算。

---

## 7. 目录约定

```text
nativebridge/
├── README.md
├── Android.bp
├── include/
│   ├── arm64_context.h
│   ├── diagnostics.h
│   ├── elf_loader.h
│   ├── nb_adapter.h
│   └── translator.h
├── adapter/
├── loader/
├── interpreter/
├── jit/
├── runtime/
├── compatibility/
│   ├── packages.yaml
│   ├── blocked-packages.yaml
│   └── workarounds/
├── tests/
│   ├── apk/
│   ├── differential/
│   ├── integration/
│   ├── native/
│   ├── stress/
│   └── unit/
└── docs/
    ├── abi-contract.md
    ├── code-cache-security.md
    ├── debugging.md
    ├── elf-loader-contract.md
    ├── jit-design.md
    ├── nativebridge-integration.md
    ├── supported-instructions.md
    └── threat-model.md
```

---

## 8. 测试与验收策略

### 8.1 测试层级

| 层级 | 关注点 | 示例 |
|---|---|---|
| Unit | 单条指令、ELF 字段、relocation、IR lowering | `ADD`, `CBZ`, `LDR`, `RET` |
| Differential | interpreter 与 JIT 语义一致 | 比较寄存器、返回值、内存副作用 |
| Native integration | loader、trampoline、TLS、`dlopen` | ARM64 `.so` 依赖链 |
| APK E2E | ART → Native Bridge → JNI → ARM64 | `nativeAdd`, callbacks |
| Stress | 重复执行、并发、恢复、资源泄露 | 1,000 次、多个线程 |
| Regression | 已修复应用/指令/崩溃不可复发 | package/version keyed cases |

### 8.2 必收集的诊断字段

每次 bridge/translator 失败至少收集：

```text
run_id
build_revision
bridge_version
android_build_fingerprint
package_name
process_name
apk_version
library_path
requested_abi
elf_machine
native_symbol
arm64_pc
instruction_bytes
decoded_instruction
arm64_registers
translator_mode            # interpreter / jit / jit-verify
error_class
host_backtrace
logcat_reference
tombstone_reference
```

### 8.3 发布前最低标准

- 所有 P0/R1 自动化测试通过；
- 不出现未分类 native crash；
- 文档中的 supported 范围与实际测试矩阵一致；
- 明确标注实验性能力，不暗示全量 ARM APK 兼容；
- 安全评审确认没有已知的长期 RWX 或未受控 native loader 暴露。

---

## 9. 风险登记与决策规则

| 风险 | 级别 | 触发信号 | 应对 |
|---|---|---|---|
| Native Bridge 接口/加载流程与预期不符 | 高 | adapter 未被调用或 ART 崩溃 | 冻结后续开发，针对目标 AOSP 分支完成集成调查 |
| ELF/linker namespace 不兼容 | 高 | 次级依赖加载失败 | 将 loader 和 namespace 建模为独立工作流，不靠全局路径 workaround |
| JIT 语义错误 | 高 | differential mismatch | 立即回退 interpreter，缩小翻译 block，新增最小复现 |
| TLS/signals/atomics 并发错误 | 高 | 偶现 crash/deadlock | 不发布多线程 support，优先 stress/replay 工具 |
| 性能不足 | 中 | JIT 无明显收益 | 先 profile，再优化热点；不得跳过正确性 Gate |
| NEON 覆盖不足 | 中 | 多媒体/图像库 trap | 以真实样本统计驱动实施，不盲目扩指令集 |
| 兼容性 workaround 膨胀 | 中 | 规则不可维护 | 所有 workaround 必须有复现、测试、移除条件和 owner |
| 安全边界削弱 | 高 | RWX、无验证 loader 映射 | 强制 W^X、最小权限、输入验证和安全评审 |

---

## 10. 完成定义（Definition of Done）

任何研发任务只有同时满足以下条件才可标记完成：

1. 代码与对应接口文档已提交；
2. 至少一个自动化测试覆盖正常路径；
3. 失败路径具有确定错误类别和诊断字段；
4. 不降低既有测试通过率；
5. 涉及 ABI、loader、JIT、线程或安全边界时，已完成设计/代码评审；
6. 已更新 `supported-instructions.md`、兼容性数据库或风险登记（如适用）；
7. 验收证据链接到任务或 PR。

---

## 11. 下一次迭代的唯一目标

在开始 JIT、窗口化、通知或 ARMv7 工作之前，优先完成：

> **R0 + R1：让固定的 Android x86_64 Guest 稳定运行 `nativebridge-smoke`，并确认 `arm64-v8a/libdemo.so` 的加载和 JNI symbol 请求确实进入 `libowsa_nativebridge.so` adapter。**

只有该 Gate 通过，才进入 ARM64 ELF loader 与 interpreter 的 R2 闭环。
