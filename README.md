# Skyward for the Xiaomi 17 Pro Max (popsicle)

小米 17 Pro Max（popsicle）的 recovery 成品镜像与配置留档。
镜像 `v40` 由外部 fork 在 `OS4.0.6.0.XPBCNXM` 基础上构建，`device.toml` 与解密链从镜像内取出，
完整分析见 [docs/解密方式.md](docs/解密方式.md)。

## 解密

这一版显式配置 **weaver 优先**：

```toml
[data]
weaver_first = true
```

- **首选**：`<id>.weaver` 记录用户在 eSE 里的槽号 → `android.hardware.weaver-service.nxp-qti`
  （NXP weaver）经 **OMAPI** 打开 eSE1 的 applet 取槽 → keymint 解 `<id>.spblob`；
- **兜底**：Android 为同一用户另留的一份 blob 绑定在 `<id>.secdis` 字节上，
  由 keymint + gatekeeper 用同一口令打开，结果等价。

17 Pro Max 的 weaver 是 **NXP**（17 Ultra 是 Goodix），所以这条链必须凑齐：

| 服务 | 注册名 |
|---|---|
| `qseecomd` | — |
| `hlosminkdaemon` | — |
| `android.hardware.security.onekeymint-service-qti` | keymint / RKP / secureclock / sharedsecret |
| `android.hardware.gatekeeper-rust-service-qti` | gatekeeper |
| `android.hardware.weaver-service.nxp-qti` | `IWeaver/default` |
| `android.hardware.secure_element-service.qti` | `ISecureElement/SIM1`、`/SIM2`、**`/eSE1`** |
| `android.hardware.boot-service.qti` | `IBootControl/default` |
| `se_omapi` | `android.se.omapi.ISecureElementService/default` |

外加：`modules.load` 含 `nxp-nci`；`/system/etc/vintf/manifest/se_omapi.xml` 必须存在
（vendor 自带的 `secure_element-service.xml` 把 name 与 instance 分写在两个标签，
`secure::describes()` 匹配不到，weaver 会报 `Failed to start omapiSeService null`，`/data` 锁死）。

## 镜像

| 项 | 值 |
|---|---|
| 文件 | `v40-skyward-popsicle-OS4.0.6.0.17PM.img` |
| 大小 | 104,857,600 B |
| md5 | `806d692dc2ab81d94054d1ccb2c70429` |
| sha256 | `37edcf697d566d66a1f05b832731d68f094f7bbd38060c00d7ab058035a3030f` |
| stock 基线 | OS4.0.6.0.XPBCNXM（recovery sha256 `cfd845ae…`) |

## 刷入

    fastboot flash recovery_b <镜像>

只动 recovery 分区；刷前先备份原厂 recovery。

## 结构

4 KiB 头（`ANDROID!`）+ `0x1000` 起的 LZ4 legacy 流（15 块）→ 123,636,072 B 的 newc cpio（1,279 条目）。
主程序在 `/system/bin/recovery`，早期 init 在 `/skyward/system-init`，配置在 `/skyward/device.toml`。
