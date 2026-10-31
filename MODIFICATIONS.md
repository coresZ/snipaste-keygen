# MODIFICATIONS

> 本项目唯一改动记录文件：每次修改后回填，按目录分节，格式「功能 → 修改内容 → 涉及文件与位置」。

## 根目录（README / 依赖说明）

### 2026-10-31 补全 Python 依赖说明

- **功能**：说明"跑这些脚本需要装哪些 Python 包"。原来 README 只有一行 `pip install pynacl pefile`
  且没区分是哪个脚本要哪个包，`kg_standalone.py` 完全没有提及。
- **修改内容**：
  - `README.md` 新增「依赖说明」小节（在原「复现步骤」之前）：脚本 → 用途 → 需要安装 → 安装命令
    对照表；说明两个注册机二选一（`kg_standalone.py` 零依赖）、`pynacl`/`pefile`/`pywinauto`/`pywin32`/
    `pillow` 各自用途、无需安装的标准库清单、Python 版本（3.10+，实测 3.13.5 x64）、实测依赖版本。
  - `README.md`「复现步骤」：开头依赖行改为指向该小节；第 3 步补充 `kg_standalone.py` 的零依赖替代命令；
    e2e 注释由"需上面的依赖"改为显式 `pip install pywinauto pywin32 pillow`。
  - `README.md`「项目结构」：补齐 `requirements.txt`、`keygen/kg_standalone.py`，并给每个脚本标注所需依赖。
  - 新增 `requirements.txt`：`pynacl` / `pefile` / `pywinauto` / `pywin32` / `pillow`，按脚本分组注释，
    文件头给出三种安装粒度（全装 / 注册机+补丁 / 零依赖注册机+补丁）。
- **涉及文件**：`README.md`（§依赖说明 新增；§复现步骤 L129-145；§项目结构 L~159）、`requirements.txt`（新增）。
- **验证**：`pip install --dry-run -r requirements.txt` 全部已满足（无解析错误）；`python keygen/kg_standalone.py`
  与 `python keygen/kg.py` 均退出码 0 且输出 `[模拟客户端 validate] = 0 (OK)`；`import pefile, win32clipboard,
  PIL, pywinauto` 均成功。
- **未验证**：Python 3.8/3.9 实际兼容性（仅按语法判断，未跑）；`e2e/activate_auto.py` 本次未运行（需真实
  Snipaste 授权对话框环境，且依赖已装齐但未触发 UI 流程）。
- **版本号**：纯文档/依赖清单改动，未改代码，`package.json`/`banner.txt` 不适用（本项目无这两文件）。

### 2026-10-31 补丁器支持命令行传路径

- **功能**：`patch/patch_2113.py` 原本把 `EXE` 硬编码为 `Snipaste-2.11.3-x64\Snipaste.exe`，
  对已安装目录只能改源码；改为可从 `sys.argv[1]` 接收路径（缺省值不变）。
- **修改内容**：
  - `patch/patch_2113.py` L4 用法注释、L19 `EXE = sys.argv[1] if len(sys.argv) > 1 else ...`。
  - `README.md` 第 3 步命令改为 `python patch/patch_2113.py [Snipaste.exe 路径]`；e2e 步骤补参数与
    「Win11 25H2 托盘 `TrayMenu` 右键菜单」已知差异说明。
- **涉及文件**：`patch/patch_2113.py`、`README.md`（复现步骤 2/4 段）。
- **验证**：官方 2.11.3 x64 样本上特征码唯一命中，文件偏移 `0x24AC80` 改写成功。
- **未验证**：`e2e/activate_auto.py` 全自动路径（Win11 25H2 托盘交互差异）。
- **版本号**：仅脚本参数兼容性改动，无 `package.json`/`banner.txt`。
