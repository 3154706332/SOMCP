---
name: "android-reverse-toolkit"
description: "Android/Flutter 逆向工程全栈工具集。通过 mcp.maxss.top/mcp 网关提供 192 个工具，覆盖 APK 分析、SO 二进制逆向、Flutter/Dart 反编译、Unidbg 模拟执行、调用图导航、R2 反汇编、Frida Hook、Android 设备控制等。当用户需要进行 Android 逆向分析、APK 破解、SO 加密算法提取、Flutter 应用逆向、native 层调试、抓包脱壳、协议分析、或任何移动安全相关任务时调用此 Skill。"
---

# Android Reverse Engineering Toolkit

通过 `mcp.maxss.top/mcp` 网关服务器，提供 Android/Flutter 逆向工程全栈能力。该网关聚合 4 个后端服务，共 192 个工具，覆盖从 APK 静态分析到 SO 动态模拟执行的完整逆向链路。

---

## MCP 服务器信息

| 项目 | 值 |
|------|-----|
| 服务器名称 | `mcp-protocol-skeleton` |
| 协议版本 | MCP JSON-RPC 2.0 (2025-06-18) |
| HTTP 端点 | `https://mcp.maxss.top/mcp` |
| SSE 端点 | `https://mcp.maxss.top/sse` |
| 工具总数 | 192 |

### 调用方式

**方式一：JSON-RPC 直接调用（推荐）**

```bash
# 初始化
curl -s -X POST https://mcp.maxss.top/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"trae-client","version":"1.0"}},"id":1}'

# 调用工具
curl -s -X POST https://mcp.maxss.top/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"<工具名>","arguments":{<参数>}},"id":2}'
```

**方式二：通过 run_mcp 调用（如果已配置为本地 MCP server）**

```
run_mcp(server_name="<server_name>", tool_name="<工具名>", args={<参数>})
```

---

## 网关架构与工具命名规则

网关聚合 4 个后端服务，工具名以各后端 `prefix` 为前缀：

| 前缀 | 后端 | 端口 | 工具数 | 定位 |
|------|------|------|--------|------|
| `port8118_` | Android 设备控制 | 8118 | 21 | 真机/模拟器控制、UI 自动化 |
| `port8787_` | MT 管理器 | 8787 | 26 | APK 反编译、DEX 分析、资源编辑 |
| `port8000_` | SO 二进制分析 | 8000 | 38 | ELF 分析、Rizin、Unidbg、LIEF |
| `127001_5051_` | 全功能逆向套件 | 5051 | 106 | R2、Blutter、Unidbg、NavGraph、项目管理 |

**工具全名 = 前缀 + 工具名**，例如 `127001_5051_R2_Open`、`port8118_screenshot`。

---

## 一、port8118 — Android 设备控制 (21 工具)

用于真机或模拟器的远程控制，无需手动操作设备。

### 设备信息与环境

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8118_device_info` | 获取设备基本信息（型号、系统版本、架构） | 无 |
| `port8118_installed_apps` | 列出已安装应用 | packageName(可选过滤) |
| `port8118_shell_exec` | 执行 shell 命令 | command(必填) |
| `port8118_env_simulate` | 模拟环境变量/设备指纹 | 各类环境参数 |
| `port8118_permission_manage` | 管理应用权限 | action, packageName, permission |

### 应用管理

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8118_debug_app` | 以调试模式启动应用 | packageName(必填) |
| `port8118_install_apk` | 安装 APK | filePath(必填) |
| `port8118_app_manage` | 应用管理（启动/停止/卸载/清除数据） | action, packageName |
| `port8118_app_data_manage` | 应用数据管理（备份/恢复/清除） | action, packageName |
| `port8118_send_intent` | 发送 Intent | action, data, packageName |

### 文件操作

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8118_file_list` | 列出设备目录 | path(必填) |
| `port8118_file_read` | 读取设备文件 | path(必填) |
| `port8118_file_write` | 写入设备文件 | path, content |

### UI 与日志

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8118_screenshot` | 截屏 | 无 |
| `port8118_ui_dump` | 导出 UI 层级（JSON/XML） | format(默认json), compressed |
| `port8118_ui_action` | UI 自动化操作（点击/滑动/输入） | action, 坐标/文本 |
| `port8118_logcat_read` | 读取 logcat 日志 | packageName(可选过滤) |
| `port8118_filter_manage` | 管理日志过滤器 | action, packageName, tag, level |
| `port8118_recording_manage` | 管理日志录制 | action(start/stop/list/get) |
| `port8118_crash_log` | 查看崩溃日志 | action, packageName |
| `port8118_crash_manage` | 管理崩溃日志 | action(report/list/get) |

### 典型工作流

```
1. 安装 APK → port8118_install_apk
2. 启动应用 → port8118_debug_app
3. 抓取日志 → port8118_recording_manage(start) → port8118_logcat_read
4. UI 操作 → port8118_ui_dump → port8118_ui_action → port8118_screenshot
5. 提取文件 → port8118_file_list → port8118_file_read
```

---

## 二、port8787 — APK 逆向工程 (26 工具)

MT 管理器风格的 APK 全量分析工具集。

### APK 生命周期

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8787_mt_apk_open` | 打开 APK 文件 | filePath(必填) |
| `port8787_mt_apk_close` | 关闭 APK | session_id |
| `port8787_mt_apk_list_available_apks` | 列出可用 APK | 无 |
| `port8787_mt_apk_list` | 列出 APK 内文件 | session_id, path(可选) |
| `port8787_mt_apk_search` | 搜索 APK 内文件 | session_id, query |

### DEX/Smali 分析

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8787_mt_apk_dex_outline_class` | 列出 DEX 类结构 | session_id, dex_index |
| `port8787_mt_apk_dex_xref` | DEX 交叉引用 | session_id, class_name, method_name |
| `port8787_mt_apk_read_text` | 读取文本文件（Smali/XML） | session_id, path |
| `port8787_mt_apk_read_bytes` | 读取二进制文件 | session_id, path, offset, size |

### 资源分析

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8787_mt_apk_resource_read` | 读取资源（arsc） | session_id, resource_id |
| `port8787_mt_apk_resource_xref` | 资源交叉引用 | session_id, resource_id |

### Native 层分析

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8787_mt_apk_native_inspect` | 检查 native 库 | session_id, lib_name |
| `port8787_mt_apk_native_read_items` | 读取 native 项（导出/导入/字符串） | session_id, lib_name, type |
| `port8787_mt_apk_native_map_address` | 地址映射 | session_id, lib_name, address |
| `port8787_mt_apk_native_xref` | Native 交叉引用 | session_id, lib_name, address |
| `port8787_mt_apk_native_disassemble` | 反汇编 native 代码 | session_id, lib_name, address, count |
| `port8787_mt_apk_native_function_cfg` | 生成函数控制流图 | session_id, lib_name, function_name |

### 修改与重打包

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8787_mt_apk_edit_open` | 打开编辑模式 | session_id |
| `port8787_mt_apk_edit_text` | 编辑文本文件 | session_id, path, content |
| `port8787_mt_apk_edit_resource` | 编辑资源 | session_id, resource_id, value |
| `port8787_mt_apk_patch_bytes` | 字节级 Patch | session_id, path, offset, bytes |
| `port8787_mt_apk_native_patch_instructions` | 修改 native 指令 | session_id, lib_name, address, instructions |
| `port8787_mt_apk_native_patch_string` | 修改 native 字符串 | session_id, lib_name, old_str, new_str |
| `port8787_mt_apk_edit_check` | 检查编辑状态 | session_id |
| `port8787_mt_apk_build` | 构建/重打包 APK | session_id, output_path |
| `port8787_mt_apk_continue` | 继续操作 | session_id |

### 典型工作流

```
1. port8787_mt_apk_open → 打开 APK
2. port8787_mt_apk_list → 浏览文件结构
3. port8787_mt_apk_dex_outline_class → 分析 DEX 类
4. port8787_mt_apk_native_inspect → 检查 SO 库
5. port8787_mt_apk_edit_open → port8787_mt_apk_edit_text → port8787_mt_apk_build → 修改重打包
```

---

## 三、port8000 — SO/ELF 二进制分析 (38 工具)

以 Rizin/Unidbg/LIEF 为引擎的 SO 库深度分析。

### SO 文件管理

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8000_so_open` | 打开 SO/ELF 文件 | filePath(必填) |
| `port8000_so_close` | 关闭 SO 会话 | session_id |
| `port8000_apk_analyze` | 分析 APK 中的所有 SO | apk_path(必填) |

### 静态分析

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8000_analyze_elf` | 全面分析 ELF 结构 | session_id |
| `port8000_read_stats` | 读取分析统计 | session_id |
| `port8000_analysis_report` | 生成分析报告 | session_id, format |
| `port8000_analyze_functions` | 列出函数 | session_id, filter |
| `port8000_analyze_cfg` | 控制流图分析 | session_id, function_name |
| `port8000_analyze_crypto` | 检测加密常量/算法 | session_id |
| `port8000_analyze_xrefs` | 交叉引用分析 | session_id, address |
| `port8000_analyze_esil` | ESIL 模拟分析 | session_id, address |
| `port8000_search_bytes` | 搜索字节模式 | session_id, pattern(hex) |
| `port8000_search_strings` | 搜索字符串 | session_id, query |
| `port8000_read_disasm` | 反汇编 | session_id, address, count |
| `port8000_read_hexdump` | 十六进制查看 | session_id, address, size |
| `port8000_flutter_blutter` | Flutter 库分析 | session_id |

### 修改与 Patch

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8000_edit_hex` | 十六进制编辑 | session_id, address, bytes |
| `port8000_edit_asm` | 汇编级编辑 | session_id, address, asm_code |
| `port8000_edit_symbol` | 编辑符号 | session_id, name, new_name |
| `port8000_edit_fix_sections` | 修复节区 | session_id |

### 模拟执行 (Unidbg)

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8000_emulate_call` | 模拟调用函数 | session_id, function_name, args |
| `port8000_emulate_dump` | Dump 模拟内存 | session_id, address, size |
| `port8000_unidbg_session` | 创建 Unidbg 会话 | so_path(必填) |
| `port8000_unidbg_memory` | Unidbg 内存操作 | session_id, action, address |
| `port8000_unidbg_debug` | Unidbg 调试 | session_id, address |
| `port8000_unidbg_batch` | 批量 Unidbg 调用 | session_id, calls |

### 高级功能

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `port8000_diff_so` | 对比两个 SO 差异 | session_id, other_path |
| `port8000_rizin_api` | Rizin API 直接调用 | session_id, cmd |
| `port8000_lief_api` | LIEF API 调用 | session_id, command |
| `port8000_unidbg_api` | Unidbg API 调用 | session_id, command |
| `port8000_xanso_api` | XanSo API 调用 | session_id, command |
| `port8000_build_so` | 构建/编译 SO | session_id, source_path |
| `port8000_session_open` | 打开分析会话 | filePath |
| `port8000_session_history` | 会话历史 | session_id |
| `port8000_session_audit` | 会话审计 | session_id |
| `port8000_system_control` | 系统控制 | action |
| `port8000_app_config` | 应用配置 | key, value |
| `port8000_meta_info` | 元信息 | 无 |

---

## 四、127001_5051 — 全功能逆向套件 (106 工具)

这是最强大的工具集，整合了 R2、Blutter、Unidbg、NavGraph 等核心引擎。

### 4.1 R2 系列 (22 工具)

基于 Radare2 的二进制分析引擎。

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `127001_5051_R2_Version` | 查看 R2 版本 | 无 |
| `127001_5051_R2_Open` | 打开二进制文件创建 R2 会话 | file(必填), write_mode(可选) |
| `127001_5051_R2_Close` | 关闭 R2 会话 | session_id(必填) |
| `127001_5051_R2_Cmd` | 执行原生 R2 命令 | session_id, cmd(必填) |
| `127001_5051_R2_Analyze` | 自动分析二进制 | session_id, depth(可选) |
| `127001_5051_R2_Disassemble` | 反汇编指定地址 | session_id, address, count(可选) |
| `127001_5051_R2_Seek` | 跳转到指定地址 | session_id, address(必填) |
| `127001_5051_R2_Info` | 获取二进制信息 | session_id |
| `127001_5051_R2_Functions` | 列出函数 | session_id, filter(可选) |
| `127001_5051_R2_Search_Functions` | 搜索函数 | session_id, query(必填) |
| `127001_5051_R2_Search_String` | 搜索字符串 | session_id, query(必填) |
| `127001_5051_R2_Sections` | 列出节区 | session_id |
| `127001_5051_R2_Imports` | 列出导入 | session_id |
| `127001_5051_R2_Exports` | 列出导出 | session_id |
| `127001_5051_R2_Hexdump` | 十六进制 dump | session_id, address, size |
| `127001_5051_R2_Calculate` | 地址计算 | session_id, expression |
| `127001_5051_R2_Xrefs` | 交叉引用 | session_id, address |
| `127001_5051_R2_Hash` | 计算哈希 | session_id, algorithm |
| `127001_5051_R2_Search` | 搜索二进制模式 | session_id, pattern |
| `127001_5051_R2_Entries` | 列出入口点 | session_id |
| `127001_5051_R2_List_Sessions` | 列出所有 R2 会话 | 无 |

**PseudoC 反编译系列：**

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `127001_5051_R2_PseudoC_Status` | 查看 PseudoC 插件状态 | session_id |
| `127001_5051_R2_Decompile_Function` | 反编译函数到伪 C 代码 | session_id, address |
| `127001_5051_R2_Get_PseudoC` | 获取指定函数的伪 C 代码 | session_id, function_name |
| `127001_5051_R2_Search_PseudoC` | 在伪 C 代码中搜索 | session_id, query |
| `127001_5051_R2_Export_PseudoC_To_File` | 导出伪 C 代码到文件 | session_id, output_path |

**分析辅助：**

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `127001_5051_R2_Analyze_Target` | 分析目标函数（深度分析） | session_id, target |
| `127001_5051_R2_Analyze_File` | 分析文件 | session_id, file_path |
| `127001_5051_R2_Config_Manager` | 管理 R2 配置 | session_id, action, key, value |
| `127001_5051_R2_Analysis_Hints` | 分析提示 | session_id, address |
| `127001_5051_R2_Manage_Xrefs` | 管理交叉引用 | session_id, action, address |

### 4.2 Blutter 系列 (17 工具)

专门针对 Flutter/Dart 应用（libapp.so）的逆向分析。

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `127001_5051_Blutter_Analyze` | 分析 libapp.so，**所有 Blutter 操作的入口** | file(必填,libapp.so路径) |
| `127001_5051_Blutter_Classes` | 列出所有 Dart 类 | session_id, filter(可选) |
| `127001_5051_Blutter_Class_Detail` | 查看类详情（字段/方法） | session_id, class_name(必填) |
| `127001_5051_Blutter_Functions` | 列出/搜索 Dart 函数 | session_id, filter(可选) |
| `127001_5051_Blutter_Disassemble` | 反汇编 Dart 函数 | session_id, function_name |
| `127001_5051_Blutter_Strings` | 列出所有 Dart 字符串 | session_id, filter(可选) |
| `127001_5051_Blutter_PP_Table` | 查看 pp.txt 偏移表 | session_id, filter(可选) |
| `127001_5051_Blutter_Object_Layouts` | 查看对象内存布局 | session_id, class_name |
| `127001_5051_Blutter_Frida` | 生成 Frida Hook 脚本 | session_id, function_name |
| `127001_5051_Blutter_To_R2` | 将 Blutter 函数偏移跳转到 R2 分析 | session_id, function_name, r2_session_id |
| `127001_5051_Blutter_Patch` | 修改 Dart 函数字节码 | session_id, function_name, bytes |
| `127001_5051_Blutter_Modify_String` | 修改 Dart 字符串 | session_id, old_str, new_str |
| `127001_5051_Blutter_To_Frida_Hook` | 生成完整 Frida Hook 代码 | session_id, function_name, hook_type |
| `127001_5051_Blutter_Dart_Access` | Dart 对象访问（运行时） | session_id, expression |
| `127001_5051_Blutter_Find_Instances` | 查找类实例 | session_id, class_name |
| `127001_5051_Blutter_Info` | 查看 Blutter 会话信息 | session_id |
| `127001_5051_Blutter_Logs` | 查看 Blutter 日志 | session_id |
| `127001_5051_Blutter_Close` | 关闭 Blutter 会话 | session_id |
| `127001_5051_Blutter_List_Sessions` | 列出所有 Blutter 会话 | 无 |

**Blutter 典型工作流：**

```
1. 从 APK 提取 libapp.so → port8787_mt_apk_open → port8787_mt_apk_list → port8787_mt_apk_read_bytes
2. 分析 libapp.so → 127001_5051_Blutter_Analyze(file="libapp.so")
3. 搜索加密函数 → 127001_5051_Blutter_Functions(filter="encrypt")
4. 查看函数详情 → 127001_5051_Blutter_Disassemble(function_name="encryptData")
5. 生成 Frida Hook → 127001_5051_Blutter_To_Frida_Hook(function_name="encryptData")
```

### 4.3 NavGraph 调用图导航 (10 工具)

基于调用图的分析，快速定位关键函数和调用链。

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `127001_5051_Nav_BuildGraph` | 构建调用图 | session_id, blutter_session_id(必填) |
| `127001_5051_Nav_Stats` | 调用图统计信息 | session_id |
| `127001_5051_Nav_ExportGraph` | 导出调用图数据 | session_id, format |
| `127001_5051_Nav_Modules` | 列出模块 | session_id |
| `127001_5051_Nav_Function` | 查看函数信息 | session_id, function_name |
| `127001_5051_Nav_Callers` | 查看谁调用了此函数 | session_id, function_name |
| `127001_5051_Nav_Callees` | 查看此函数调用了谁 | session_id, function_name |
| `127001_5051_Nav_CallChain` | 查找两个函数间的调用链 | session_id, from(必填), to(必填), max_depth |
| `127001_5051_Nav_Search` | 搜索函数（按名称/字符串内容） | session_id, query(必填), limit |
| `127001_5051_Nav_PatchPoints` | 推荐 Patch 点候选 | session_id, limit |
| `127001_5051_Nav_Mermaid` | 生成 Mermaid 流程图 | session_id, type(modules/function), target |
| `127001_5051_Nav_StringXrefs` | 字符串交叉引用（r2扫描） | session_id, query(必填), limit |

**NavGraph 典型工作流：**

```
1. 构建调用图 → 127001_5051_Nav_BuildGraph(blutter_session_id=xxx)
2. 搜索关键函数 → 127001_5051_Nav_Search(query="verify")
3. 查看调用链 → 127001_5051_Nav_CallChain(from="main", to="verifySign")
4. 获取 Patch 点 → 127001_5051_Nav_PatchPoints
5. 可视化 → 127001_5051_Nav_Mermaid(type="function", target="verifySign")
```

### 4.4 Unidbg 模拟执行 (21 工具)

在无真机环境下模拟执行 SO 中的 ARM 函数，提取加密/解密结果。

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `127001_5051_Ub_Open` | 加载 SO 到 Unidbg 模拟器 | file(必填,so路径), call_jni(可选) |
| `127001_5051_Ub_Close` | 关闭模拟会话 | session_id(必填) |
| `127001_5051_Ub_List_Sessions` | 列出所有活跃会话 | 无 |
| `127001_5051_Ub_Call` | 按导出符号名调用函数 | session_id, symbol(必填), args(可选JSON数组) |
| `127001_5051_Ub_Call_Offset` | 按模块偏移调用函数 | session_id, offset(必填,hex), args(可选) |
| `127001_5051_Ub_Dump` | 读取模拟器内存 | session_id, address(必填), size(可选) |
| `127001_5051_Ub_Modules` | 列出已加载模块及基址 | session_id(必填) |
| `127001_5051_Ub_Write` | 写模拟器内存 | session_id, address(必填), hex_bytes(必填) |
| `127001_5051_Ub_Regs` | 读/写 CPU 寄存器 | session_id, set(可选,JSON对象) |
| `127001_5051_Ub_Alloc` | 在模拟器中 malloc 内存 | session_id, size(必填), init(可选) |
| `127001_5051_Ub_Free` | 释放分配的内存 | session_id, address(必填) |
| `127001_5051_Ub_Read_String` | 读 C 字符串(遇\\0截断) | session_id, address(必填), max_len(可选) |
| `127001_5051_Ub_Hook` | 安装执行 Hook | session_id, address(必填), max_hits(可选) |
| `127001_5051_Ub_Hook_Hits` | 读取 Hook 命中记录 | session_id, keep(可选) |
| `127001_5051_Ub_Search` | 搜索内存 hex 模式 | session_id, pattern(必填,hex), start(可选) |
| `127001_5051_Ub_Disasm` | 反汇编模拟器内存 | session_id, address(必填), count(可选) |
| `127001_5051_Ub_Patch` | 运行时汇编级 Patch | session_id, address(必填), asm(必填) |
| `127001_5051_Ub_Save_State` | 保存 CPU 寄存器快照 | session_id(必填) |
| `127001_5051_Ub_Restore_State` | 恢复寄存器快照 | session_id(必填) |
| `127001_5051_Ub_Dump_To_R2` | 从 Unidbg dump 内存→R2 反汇编 | session_id, address(必填), size(可选) |

**Unidbg 典型工作流：**

```
1. 加载 SO → 127001_5051_Ub_Open(file="libxxx.so")
2. 查看模块 → 127001_5051_Ub_Modules
3. 调用函数 → 127001_5051_Ub_Call(symbol="encrypt", args=["0x0","jstr:hello"])
4. 读取返回 → 127001_5051_Ub_Dump(address="0x...", size=256)
5. Hook 调试 → 127001_5051_Ub_Hook(address="0x...") → Ub_Call → Ub_Hook_Hits
```

### 4.5 交叉引擎桥接工具 (4 工具)

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `127001_5051_Blutter_Annotate_R2` | Blutter IR 注入 R2 注释 | session_id, function_filter(可选) |
| `127001_5051_Address_Lookup` | 地址双向翻译 | blutter_session_id, address |
| `127001_5051_R2_To_Ub_Call` | R2 函数→Unidbg 调用 | r2_session_id, function_name, ub_session_id |
| `127001_5051_Ub_Hook_To_R2_Xrefs` | Hook 返回地址→R2 交叉引用 | ub_session_id, r2_session_id, address |
| `127001_5051_Blutter_To_Unidbg_Call` | Blutter 函数→Unidbg 调用 | blutter_session_id, ub_session_id, function_name |

### 4.6 项目管理 (5 工具)

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `127001_5051_Project_Save` | 保存分析状态 | name(必填), session_id, file_path, notes |
| `127001_5051_Project_Load` | 加载项目 | project_id(必填) |
| `127001_5051_Project_List` | 列出所有项目 | 无 |
| `127001_5051_Project_Delete` | 删除项目 | project_id(必填) |
| `127001_5051_Project_Export` | 导出分析报告(markdown/html) | project_id(必填), format(可选) |

### 4.7 其他工具 (31 工具)

| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `127001_5051_McpUsage` | 查看 MCP 使用统计 | 无 |
| `127001_5051_Read_Logcat` | 读取 logcat | session_id, filter |
| `127001_5051_Os_List_Dir` | 列出服务器目录 | path(必填) |
| `127001_5051_Os_Read_File` | 读取服务器文件 | path(必填) |
| `127001_5051_Find_Jni_Methods` | 查找 JNI 方法 | session_id, filter |
| `127001_5051_Apply_Hex_Patch` | 应用十六进制补丁 | session_id, address, bytes |
| `127001_5051_Rename_Function` | 重命名函数 | session_id, old_name, new_name |
| `127001_5051_Scan_Crypto_Signatures` | 扫描加密特征 | session_id |
| `127001_5051_Shell_Command` | 执行 shell 命令 | command(必填) |
| `127001_5051_File_Download` | 下载文件 | url, output_path |
| `127001_5051_Sqlite_Query` | 查询 SQLite 数据库 | db_path, query |
| `127001_5051_Apk_Open` | 打开 APK（快速分析） | file(必填) |
| `127001_5051_Apk_Close` | 关闭 APK | session_id |
| `127001_5051_Apk_Pack` | 打包 APK | session_id, output_path |

---

## 全局典型工作流

### 工作流一：APK 静态分析 → 提取加密算法

```
1. [设备] port8118_install_apk → port8118_debug_app → port8118_recording_manage(start)
2. [APK] port8787_mt_apk_open → port8787_mt_apk_list → 提取 libapp.so / libxxx.so
3. [SO] 127001_5051_R2_Open → R2_Analyze → R2_Functions → 定位 encrypt/decrypt
4. [SO] 127001_5051_R2_Decompile_Function → 获取伪 C 代码，理解算法
5. [Unidbg] 127001_5051_Ub_Open → Ub_Call → Ub_Dump → 验证加密结果
6. [保存] 127001_5051_Project_Save → 保存分析状态
```

### 工作流二：Flutter 应用逆向

```
1. [APK] port8787_mt_apk_open → 提取 lib/arm64-v8a/libapp.so
2. [Blutter] 127001_5051_Blutter_Analyze(file="libapp.so")
3. [Blutter] 127001_5051_Blutter_Functions(filter="sign") → 找到签名函数
4. [NavGraph] 127001_5051_Nav_BuildGraph → Nav_CallChain(from="main", to="sign") → 追踪调用链
5. [Blutter] 127001_5051_Blutter_To_Frida_Hook → 生成 Frida 脚本
6. [Unidbg] 127001_5051_Blutter_To_Unidbg_Call → 模拟执行验证
```

### 工作流三：真机动态调试 + Frida Hook

```
1. [设备] port8118_device_info → 确认设备连接
2. [设备] port8118_install_apk → port8118_debug_app
3. [UI] port8118_ui_dump → port8118_ui_action → 触发目标功能
4. [日志] port8118_logcat_read → 抓取运行日志
5. [Frida] 127001_5051_Blutter_Frida → 生成 Hook 脚本
6. [设备] port8118_recording_manage → 录制 Hook 输出
7. [分析] 分析 Hook 结果 → 定位关键逻辑
```

### 工作流四：SO 修改重打包

```
1. [APK] port8787_mt_apk_open → 提取目标 SO
2. [SO] 127001_5051_R2_Open → R2_Analyze → 分析目标函数
3. [SO] 127001_5051_Ub_Open → Ub_Patch → 运行时验证 Patch 效果
4. [APK] port8787_mt_apk_edit_open → port8787_mt_apk_native_patch_instructions → 修改 SO
5. [APK] port8787_mt_apk_build → 重打包
6. [设备] port8118_install_apk → 安装测试
```

---

## 调用规则与注意事项

### 工具调用方式

所有工具通过 MCP 协议调用，工具名包含前缀（如 `127001_5051_R2_Open`），参数以 JSON 对象传递。

### 会话管理

- R2、Blutter、Unidbg、MT APK 均使用会话（session_id）机制
- 使用前必须先 `Open`/`Analyze` 创建会话
- 使用后应 `Close` 释放资源
- `List_Sessions` 可以查看当前活跃会话

### 跨引擎桥接

- `Blutter → R2`：`127001_5051_Blutter_To_R2`
- `Blutter → Unidbg`：`127001_5051_Blutter_To_Unidbg_Call`
- `R2 → Unidbg`：`127001_5051_R2_To_Ub_Call`
- `Unidbg → R2`：`127001_5051_Ub_Dump_To_R2`、`127001_5051_Ub_Hook_To_R2_Xrefs`

### 参数注意事项

- 地址参数通常为十六进制格式（如 `0x9a88`）
- Unidbg 调用时，字符串参数使用 `jstr:文本` 前缀，对象参数使用 `jobj:名`
- 路径参数使用服务器上的绝对路径
- Blutter 的 `Blutter_Analyze` 是几乎所有 Blutter 操作的前置条件

### 错误处理

- 如果工具返回错误，先检查会话是否有效（未过期、未关闭）
- 地址参数格式错误是常见问题，注意使用正确的 hex 格式
- Unidbg 调用时确保 SO 已正确加载且 JNI_OnLoad 已执行
- 跨引擎桥接需要两个会话同时有效