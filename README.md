# Snipaste 2.11.3 离线激活补丁 + 注册机

对 Snipaste 2.11.3 (x64) 离线授权体系的完整逆向实现：一个 3 字节特征码补丁 + 一个自包含注册机，
在本机真实 UI 上完成离线激活并重启持久化为专业版。

> 本项目为授权环境下的逆向研究记录，样本与产物仅用于本地兼容性分析。Snipaste 是优秀的产品，
> 商用请[购买正版授权](https://zh.snipaste.com/download.html)。

## 效果

```
Snipaste 2 专业版已激活
授权给: YG
个人版 (3 设备)
感谢您的支持！现在重启即可享受专业版~
```

重启后托盘显示 `Snipaste 2.11.3 专业版 (桌面版)`。

## 原理

### 授权码结构

```
明文 JSON(license info)
  │  Ed25519 私钥签名 → sig(64B)
  │  body = gzip(JSON) ^ sig        ← 生成端只做这一次 XOR（gzip 之后）
  │  data = sig(64B) + body
  │  Base64(标准表)
  │  前缀: '0' + 校验字符 + '-'
  ▼
授权码字符串 (形如 07-SHSpe6Cw...)
```

- 校验字符 = 载荷相邻字符 ASCII 差绝对值之和 `% len` 取该位置字符（防手误，无密码学强度）
- 客户端验签 message = `gunzip(body ^ sig)`（客户端内部三次 XOR，后两次抵消）
- Ed25519 公钥硬编码于客户端（AES-128-CTR 懒解密），私钥在官方手里 → 无法伪造签名 → 必须补丁验签

### 补丁点（本仓库实测，官方 2.11.3 x64）

```
VA 0x14024B880  文件偏移 0x24AC80
原: 0F B6 D8          movzx ebx, al        ; call sub_140250020 (Ed25519 verify) 的返回值
    4C 89 6C 24 48    mov [rsp+48], r13
    48 8D 05          lea rax, ...
改: B3 01 90          mov bl, 1 ; nop      ; 验签恒真
```

特征码：`0F B6 D8 4C 89 6C 24 48 48 8D 05`（在 .text 中唯一命中）。

### 特征码注意事项

同版本号不同 build 可能有微编译差异（寄存器选择不同，如 rbp/r13）。
本项目针对官方现行 2.11.3 x64 下载包，特征码唯一命中；若你的样本 0 命中，
按 `docs/analysis.md` §2 的流程（SHA-512 常量锚点 → 验签 call 上下文）重定位即可。

### 授权 JSON

```json
{
  "nam": "YG", "eml": "yg@local.test", "lic": "SNIPASTE-KG-0001",
  "dev": 3, "pln": "Personal", "dom": "WORKGROUP", "api": 0,
  "iat": "2026-10-25T15:19:24.161", "exp": "2036-10-22T15:24:24.161",
  "exx": "...", "ref": "...",
  "hwi": {"machineid": "29D1-B6EA3E40-556F151241", "platform": 1, "domain": ""}
}
```

validate()（`sub_140245880`）逐项校验：

| 字段 | 约束 | 注册机取值 |
|---|---|---|
| eml | 非空 | 任意邮箱 |
| hwi.platform | 必须 == 1 | 1 |
| dom | 空或 == 本机工作组明文/其 blake2s hex | NetWkstaGetInfo level 100 实取 |
| hwi.machineid | == 本机 genId(L) | blake2s 见下 |
| iat | ≤ now+20min | now-5min |
| exp | ≥ now | now+10 年 |
| nam/lic/dev/pln/api | **不检查**（仅展示） | 任意 |

machineid（比较用明文）= `genId(10)`：

```python
h = blake2s(digest_size=16)
h.update(b'Snipaste 2'); h.update(b'1'); h.update(MachineGuid)
hx = h.digest()[-10:].hex().upper()          # 20 hex 字符
s  = hx[:4] + '-' + hx[4:12] + '-' + hx[12:] # 4-8-8 分段
s += adjacent_check(hx)                       # 追加校验字符（对无横线 hex 计算）
s += '1'                                      # 追加版本位
# 自洽约束: (len(s) - 横线数 - 2) // 2 == 10
```

MachineUniqueId 来源：`HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid`（缺省回退 SQMClient\MachineId）。

设备码展示形式（UI 上那个 `3-6FFHY-...`）与 machineid 是**两种形态**，展示码机器段
`blake2s('Snipaste 2'+'1'+MachineGuid)[-4:]` 可用于人工核对算法对齐（本项目实测一致）。

## 依赖说明

补丁后的 `Snipaste.exe` 只是二进制改动，运行时**不需要** Python 或任何第三方包；下面的依赖都只装在
生成端（跑脚本的机器）。按脚本对照：

| 脚本 | 用途 | 需要安装 | 安装命令 |
|---|---|---|---|
| `keygen/kg_standalone.py` | 注册机（零依赖版，推荐） | 无，纯标准库 | — |
| `keygen/kg.py` | 注册机（PyNaCl 版，与上等价） | `pynacl` | `pip install pynacl` |
| `patch/patch_2113.py` | 特征码打补丁 | `pefile` | `pip install pefile` |
| `e2e/activate_auto.py` | UI 自动化全流程（可选） | `pywinauto` `pywin32` `pillow` | `pip install pywinauto pywin32 pillow` |

- **两个注册机二选一**：命令行参数、授权码结构、`[模拟客户端 validate]` 自检逻辑一致（实测两者都输出
  `validate = 0 (OK)`）。`kg_standalone.py` 内嵌 RFC 8032 纯 Python Ed25519 实现，干净机器上直接
  `python kg_standalone.py` 就能跑；用 PyInstaller 打包单文件时也不需要额外依赖或 hook。
- `pynacl` 只被 `kg.py` 用到（`from nacl.signing import SigningKey`），`kg_standalone.py` 不 import
  任何第三方模块；`pefile` 只用来解析 PE 头/节表把文件偏移换算成 VA，补丁动作本身是纯字节替换。
- `pywinauto` 会由 pip 自动带上 `comtypes`、`pywin32`、`six`；`pywin32` 提供 `win32clipboard` /
  `win32con`（把授权码写进剪贴板）；`pillow` 只在最后的 `ImageGrab` 截图取证用，不想装可删掉
  `e2e/activate_auto.py` 里生成 `activated.png` 的那两行。
- 标准库部分（无需安装）：`winreg` 读 MachineGuid、`ctypes`/`wintypes` 调 `netapi32!NetWkstaGetInfo`
  取工作组、`hashlib.blake2s`、`zlib`（gzip）、`base64`/`json`/`argparse`。
- Python：3.10+（本项目在 3.13.5 x64 上实测）；代码没用到 3.8 之后的新语法/API，理论 3.8/3.9 也可，
  未逐一验证。
- 实测通过的依赖版本：pynacl 1.6.2 / pefile 2024.8.26 / pywinauto 0.6.9 / pywin32 312 / pillow 12.1.0。

一次装齐（含可选的 UI 自动化）：

```powershell
pip install -r requirements.txt      # 只要注册机+补丁：pip install pefile（或再装 pynacl）
```

## 复现步骤

依赖见上节「依赖说明」；最省事是 `pip install -r requirements.txt`。

```powershell
# 1. 下载官方包（约 19MB）
curl.exe -L -o Snipaste-2.11.3-x64.zip https://download.snipaste.com/archives/Snipaste-2.11.3-x64.zip
Expand-Archive Snipaste-2.11.3-x64.zip -DestinationPath Snipaste-2.11.3-x64

# 2. 打补丁（自动备份 Snipaste.exe.orig）
python patch/patch_2113.py [Snipaste.exe 路径]   # 省略路径则用脚本内默认值；自动备份 .orig

# 3. 生成本机授权码（自动读 MachineGuid + 工作组，含客户端模拟自检）
python keygen/kg.py --name YG --email yg@local.test --devices 3
#    零依赖替代（不用 pynacl，输出等价）：
#    python keygen/kg_standalone.py --name YG --email yg@local.test --devices 3
#    输出 JSON + 授权码，存 license_code.txt，末行 [模拟客户端 validate] = 0 (OK) 才算数

# 4. 启动补丁版，托盘菜单 → 解锁专业版 → 离线激活 → 粘贴授权码 → 激活
# 5. 重启 Snipaste，托盘显示「专业版」即持久化成功

# （可选）全自动：UI 自动化一条龙（需 pip install pywinauto pywin32 pillow）
python e2e/activate_auto.py [Snipaste.exe 所在目录] [license_code.txt 路径]
```

> e2e 已知差异：Win11 25H2 的托盘菜单是右键弹出的 `TrayMenu` 窗口（本仓库脚本按左键点击图标 +
> 溢出区按钮查找，可能报「解锁对话框未弹出」）。手动等价路径：右键托盘 Snipaste 图标 →
> 「解锁专业版...」→「输入授权码」→「离线激活」→ 粘贴授权码 →「激活」。

## 激活方案对比（"还有啥更好的办法"）

| 方案 | 改动 | 优点 | 缺点 |
|---|---|---|---|
| **A. patch 验签返回值（本项目）** | 3 字节 | 特征码定位稳、唯一命中、注册机照常工作 | 每个版本特征码可能微调 |
| B. 替换硬编码公钥密文 | 64 字节 | 逻辑上更"正统"（用自己密钥对） | 需已知明文推 AES-CTR keystream 再加密自己的公钥；且启动有公钥自检（x_check_pubkey*），替换后还要处理自检，工作量大 |
| C. patch validate() 恒真 | ~6 字节 | 连授权码都不用对字段 | 返回值写入点带栈布局上下文，跨版本特征更脆；且授权码格式层(长度/校验符/gzip)仍要合法，省不了注册机 |
| D. Frida 动态 hook | 0 字节 | exe 不动 | 每次要起 frida；对便携分发不友好 |

结论：A 是当前最稳的最小改动。若未来版本特征码漂移，用 `docs/analysis.md` 的定位方法
（SHA-512 常量表锚点 + 验签 call 上下文）重新定位即可。

## 二开方向

- **新版本适配**：跑 `docs/analysis.md` 的定位流程 → 更新 `patch/PATTERN`；JSON 层近几版稳定
- **注册机参数**：`kg.py` 已参数化（姓名/邮箱/设备数/方案/年限）；pln 支持 Personal/Business/Trial
- **工作组段**：UI 展示码中工作组段实测为 5 字节（非帖子的 4 字节），只影响展示码复刻，不影响激活
- **跨机授权码**：换机器重跑 `kg.py` 即可（machineid 绑定 MachineGuid）；也可以做"收集目标机
  设备码 → 异地生成 → 回贴"的离线分发流
- **Mac/Linux**：授权体系同源（platform 字段区分），补丁点需按平台重新定位

## 项目结构

```
├── README.md
├── requirements.txt        # 依赖清单（按脚本分组注释，含可选 e2e）
├── keygen/kg.py            # 注册机（PyNaCl 签名版，依赖 pynacl）
├── keygen/kg_standalone.py # 注册机（零依赖版，内嵌纯 Python Ed25519）
├── patch/patch_2113.py     # 特征码补丁器（依赖 pefile）
├── e2e/activate_auto.py    # UI 自动化全流程（可选，依赖 pywinauto/pywin32/pillow）
├── docs/analysis.md        # 逆向分析记录（定位方法 + 差异点）
```

## License

MIT

## 直接下载(免动手)

到 [Releases](../../releases) 页下载任一:

| 资产 | 说明 |
|---|---|
| Snipaste-2.11.3-Pro-Portable.zip | 全套便携包: 已打补丁的 Snipaste 2.11.3 + SnipasteKeygen.exe + 快速使用说明.txt, 解压即用 |
| SnipasteKeygen.exe | 免 Python 的注册机单文件(零依赖, 在你的机器上运行生成本机授权码) |
| Snipaste-2.11.3-x64-patched.exe | 已打补丁的 Snipaste.exe 单文件, 替换官方包同名文件即可 |

三步走: 解压 → 跑 \SnipasteKeygen.exe\（确认末行 \[模拟客户端 validate] = 0 (OK)\）→ Snipaste 托盘菜单
alidate = 0 (OK)) → Snipaste 托盘菜单
「解锁专业版 → 离线激活」粘贴授权码 → 重启。授权码绑定本机, 换机重跑 keygen。
