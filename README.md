# Unique-Code-Per-Device

一机一码授权系统：C / Win32 实现，支持机器码绑定、离线激活码、有效期控制与防篡改校验。

当前版本：1.0.0.0

 Unique-Code-Per-Device（Cross-platform）
一机一码授权系统（跨平台）：代码请见 [GitHub](https://github.com/skyjkwy/Unique-Code-Per-Device_Cross-platform)

## 功能概览

- 机器码：基于本机硬件指纹生成，可复制、可规范化、可混淆显示。
- 离线激活：作者用私钥对「机器码 + 用户信息 + 有效时长」签名生成激活码，客户机用内置公钥离线验签，全程不需要授权服务器。
- 有效期控制：激活码可设置年/月/日/时/分/秒，运行中实时计算剩余时间。
- 时间防护：NTP 授时（UDP 123）加性能计数器锚定，抑制改系统时间。
- 防篡改：license.dll 构建时计算 .text 段 SHA-256 并写回源码，加载时校验；同时带反调试检查。
- 授权数据：`%APPDATA%\SkySoftware\license-SkyBox.dat`，并辅以注册表与 NTFS 备用数据流（ADS）冗余存储。
- 除 NTP 授时外无任何网络通信。

## 目录结构

```
Unique-Code-Per-Device/
├─ build.bat                 一键构建脚本（自动探测 VS，可选代码签名）
├─ bin/                      构建产物
├─ include/                  头文件（license_dll.h、ecdsa.h、sha256.h 等）
├─ src/                      源码（20 个 .c、4 个 .rc、app.manifest）
├─ key/
│  ├─ sky.bin                ECDSA 私钥（保密，仅用于 generator.exe）
│  ├─ sky_share.bin          ECDSA 公钥（编入 license.dll）
│  └─ 公钥私钥区分.txt
├─ sign/                     可选的签名证书配置（sign.config）
├─ logo.ico                  程序图标
├─ LICENSE                   MIT 许可证
└─ README.md
```

## 构建

环境要求：Windows x64 + Visual Studio 2019/2022 及以上（含 C++ 桌面开发工具）。脚本会自动查找 `VsDevCmd.bat`。

直接运行：

```
build.bat
```

构建流程：

1. 用 `embed_key` 把 `key\*.bin` 转成 `include\embedded_pubkey.h`、`include\embedded_privkey.h`。
2. 编译 `hashgen.exe`。
3. 第一次编译 `license.dll`。
4. `hashgen` 计算 `license.dll` 的 .text 段与文件结构 SHA-256，写入 `include\dll_hash.h`。
5. 第二次编译 `license.dll`，使自校验哈希一致。
6. 依次构建 `dll_test.exe`、`license_system.exe`、`generator.exe`、`keygen.exe`、`dat_viewer.exe`。
7. 如配置了 PFX，对全部产物做 SHA1 + SHA256 双签名（含 RFC3161 时间戳）。

产物均在 `bin\`：

| 文件 | 说明 |
| --- | --- |
| `license.dll` | 授权核心 DLL，对外提供 `License_*` 接口 |
| `license_system.exe` | 图形化激活工具（显示机器码、输入激活码） |
| `generator.exe` | 激活码生成器，内置私钥，仅供作者使用 |
| `keygen.exe` | ECDSA 密钥对生成器 |
| `dll_test.exe` | license.dll 加载与接口测试 |
| `dat_viewer.exe` | `license-SkyBox.dat` 授权文件查看器 |

## 使用流程

1. 生成密钥对（首次或需要换密钥时）：
   运行 `bin\keygen.exe`，得到 `sky.bin`（私钥）与 `sky_share.bin`（公钥），放入 `key\` 后重新构建。
2. 集成到目标程序：
   加载 `license.dll`，调用 `License_Init` 初始化，用 `License_GetMachineCode` 取机器码，用 `License_ShowActivationDialog` 或自定义界面完成激活，用 `License_IsActivated` / `License_Verify` 判断授权状态。
3. 客户激活：
   客户运行 `license_system.exe`，复制机器码发给作者。
4. 作者发码：
   在 `generator.exe` 中填入机器码、用户名/邮箱和有效时长，生成激活码；客户粘贴后点击「验证激活」。

## 主要接口

`include\license_dll.h` 导出：

- `License_Init` / `License_Shutdown`
- `License_IsActivated` / `License_Verify`
- `License_GetMachineCode` / `License_GetObfuscatedMachineCode` / `License_NormalizeMachineCode`
- `License_GetInfo`
- `License_ShowActivationDialog` / `License_Activate` / `License_Clear`
- `License_GenerateCode`（发码端生成激活码）
- `License_SetNtpBase`（外部已有时钟基准时可注入）

## 注意事项

- `key\sky.bin` 与 `generator.exe` 属于发码侧，切勿随客户程序分发。
- 修改 `license.dll` 相关源码后必须完整执行 `build.bat`，让 `hashgen` 重新写入哈希，否则自校验会失败。
- 代码签名是可选的：`sign\sign.config` 存在时复用其中证书，删除后下次构建会重新弹出选择对话框；不选证书则跳过签名。
- 全项目源码不含注释，`embed_key` 与 `hashgen` 生成的头部文件同样不带注释。

## 许可证

MIT，见 [LICENSE](LICENSE)。
