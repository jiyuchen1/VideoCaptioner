# Python 3.13 支持：本次更新与后续问题

更新日期：2026-09-28

本文记录 VideoCaptioner 从「仅支持 Python 3.10–3.12」放宽到「支持 3.10–3.13」的完整变更、验证证据，以及刻意没有做、留给后续的事项。

## 一、本次变更

| 文件 | 改动 | 原因 |
| --- | --- | --- |
| `pyproject.toml:8` | `requires-python` 由 `>=3.10,<3.13` 改为 `>=3.10,<3.14` | 声明层放行 3.13；不改这行时 `uv sync --python 3.13.7` 与 `pip install` 都会被直接拒绝 |
| `pyproject.toml:20` | 新增分类符 `Programming Language :: Python :: 3.13` | 与上面的区间保持一致 |
| `pyproject.toml:32` | 新增 `"audioop-lts>=0.2.1; python_version >= '3.13'"` | 本次唯一的功能性修复，详见第二节 |
| `videocaptioner/cli/commands/doctor.py:54-56` | 运行期上界改 `< (3, 14)`，提示文案同步为 `>=3.10,<3.14` | `doctor` 自己会拒绝 3.13，不改会出现「装得上、诊断报错」 |
| `.github/workflows/ci.yml` | 构建矩阵扩为 `["3.12", "3.13"]`，`fail-fast: false` | 让 3.13 进入持续集成，而不是只在本地验证过 |
| `uv.lock` | 重新生成 | 锁文件必须随 `requires-python` 变动 |
| `docs/guide/getting-started.md:19` | 「Python 3.10 或更高版本」→「Python 3.10 ~ 3.13」 | 与实际约束一致，避免读成无上限 |

### 关于锁文件

`uv.lock` 的 diff 有 495 行新增，但逐条核对后确认：**没有任何依赖被顶版本**。新增内容只有三类——

1. `audioop-lts 0.2.2` 一个包；
2. 既有包在 3.13 下的 wheel 哈希条目；
3. 解析标记按 `python_full_version >= '3.13'` / `< '3.13'` 拆分。

因此 3.10–3.12 用户解析到的依赖集合一字未变。`uv lock --check` 通过。

## 二、为什么必须加 audioop-lts

Python 3.13 按 PEP 594 把 `audioop` 移出了标准库，而 `pydub` 在模块顶层就导入它：

```
pydub/utils.py:16  import pyaudioop as audioop
ModuleNotFoundError: No module named 'pyaudioop'
```

结果是**只要 import 就崩**，不是运行到某条分支才崩。受影响的代码路径有三处，全是主流程：

- `videocaptioner/core/asr/base.py:81` — 读取音频时长
- `videocaptioner/core/asr/chunked_asr.py:119-139` — 长音频切块并导出 mp3 分片
- `videocaptioner/core/dubbing/audio.py:41-51` — 配音时间线组装（静音底轨 + 按时位叠加 + 导出）

`audioop-lts` 是被移出时配套维护的官方长期支持版本，装上后提供顶层 `audioop` 包，正好命中 pydub 导入链的第一支，因此 **pydub 自身一行都不用改**。两点已核实：

- 它的 wheel 是 `cp313-abi3`，向后兼容 3.14 的标准构建，不会刚补完又卡在下个版本；
- 其 `requires-python` 为 `>=3.13`，所以必须带版本标记，不能写成无条件依赖（否则 3.10–3.12 会装失败）。

`pydub` 在 PyPI 上的最新版仍是 **0.25.1（2021 年）**，没有 `requires-python` 元数据，上游不会再为 3.13/3.14 做任何事。因此这里只能靠 shim，「升级 pydub」不是选项。

## 三、验证记录

在 Python 3.13.7（`C:\Program Files\Python313`）上实测：

```bash
uv sync --frozen --python 3.13.7      # 之前会报 incompatible with the project's Python requirement
.venv/Scripts/python.exe -m videocaptioner doctor
.venv/Scripts/python.exe -m pytest -q
```

- `doctor` 输出 `OK    python: Python 3.13.7`（改动前该项为 error）
- `audioop` 由 `site-packages\audioop\__init__.py` 提供，`from pydub import AudioSegment` 正常
- 调用项目真函数验证：`create_timeline_audio()` 产出 4000 ms / 384 KB wav；`AudioSegment` 切块导出 mp3 得到 8585 字节
- 全量 `pytest` 的失败集合为 36 条 `FAILED`/`ERROR`，与**改动前 3.13 环境**以及 **3.12 基线**逐条 diff 完全一致 → 无新增回归；`tests/test_dubbing` 14 passed
- `ruff check videocaptioner/` 与 `pyright videocaptioner/cli/`（即 CI 的两步）均干净，pyright 为 0 errors
- CI 的 3.12 那条腿用 `uv sync --frozen --python 3.12 --dry-run` 验证：exit 0、装 61 个包、**不会**装 `audioop-lts`（标记生效）、`pyqt5-sip` 仍是 3.12 对应的 12.18.0

## 四、三条构建线的结论

先给结论：**都不需要切 3.13**。放宽支持区间不等于放弃 3.12，所以留在 3.12 仍是合法且已验证的组合。

- **PyPI 发布（`publish-pypi.yml`）**：不切。项目自身没有 C 扩展，在 3.13 环境里 `uv build` 出来的产物名就是 `videocaptioner-...-py3-none-any.whl`，与构建机解释器无关。真正决定用户能否安装的是 METADATA，已核对带出 `Requires-Python: <3.14,>=3.10` 和 `Requires-Dist: audioop-lts>=0.2.1; python_version >= '3.13'`。3.13 用户 `pip install videocaptioner` 时由 pip 自己解析该标记。
- **桌面产物（`build-desktop.yml`）**：不切，但它是三条线里唯一**有真实解释器语义**的一条。PyInstaller 会把构建机的 CPython（现为 `python312.dll`）连同 site-packages 冻进产物，最终用户没有 Python，跑的就是打进来的那个解释器。切换与否属于「要不要升级随包运行时」的独立决定，不是本次适配的必要项。触发条件和注意事项见下一节 T1。
- **pyright `pythonVersion = "3.12"`（`pyproject.toml:118`）**：不切，也不该往 3.13 切。它是开发期的检查语义版本，不进 wheel、不进 METADATA，pip 用户永远看不到。相关的不一致问题见 T2。

## 五、后续问题清单

### T1 桌面产物若切 3.13，先补两处

1. `VideoCaptioner.spec` 的 `hiddenimports` 列了 `"pydub"`（第 50 行）但**没有 `"audioop"`**。3.13 上 `audioop` 来自 audioop-lts，是含 `_audioop.pyd` 独立扩展的包，静态分析通常能顺着 pydub 的 `try: import audioop` 找到，但冻结包里值得显式补一行作为保险。
2. 补完需重跑 `scripts/smoke_desktop.py` 并走完发布打包流程，不能只看构建成功。

### T2 两个检查工具的目标版本不一致

`tool.pyright.pythonVersion = "3.12"` 对上 `tool.ruff.target-version = "py310"`（`pyproject.toml:166`）。支持下限是 3.10，pyright 按 3.12 检查会**放行只在 3.11/3.12 才存在的 API**（`tomllib` 这类就需要手写版本分支，见 `videocaptioner/cli/config.py:18-24`）。要收敛的话，方向是把 pyright 对齐到下限 3.10，而不是跟着升到 3.13。这是本次改动之前就存在的口子。

### T3 pydub 是停滞依赖，去依赖是根治方案

当前只靠 shim 续命。若要彻底摆脱，pydub 的真实调用面只有三处，都能用项目已经硬要求的 ffmpeg/ffprobe 等价实现：

- 音频时长 → `ffprobe -show_entries format=duration`
- 切块导出 mp3 → `ffmpeg -ss/-t -f mp3`
- 配音时间线 → `anullsrc` 静音底轨 + `adelay`/`amix`

同文件的 `change_tempo()`、`mux_dubbed_audio()` 本来就是 `subprocess` 调 ffmpeg，风格一致。工作量约 60–80 行重写，主要风险点在叠加混音：需比对音量与削顶是否与现状逐位一致（现在是 48 kHz 底轨、`clip += gain_db` 的算法）。

### T4 `httpx` 未声明为直接依赖（已确认可复现）

`videocaptioner/core/llm/request_logger.py:7` 直接 `import httpx`，但它只靠 `openai` 传递引入（锁文件中为 openai 2.15.0 → httpx 0.28.1）。

**绕过锁文件的安装方式会立刻踩到**：`pip install .`（或将来从 PyPI 装）按 `openai>=1.97.1` 自由解析，实测浮到 **openai 3.19.2**，而该版本上游改用 `httpx2`，解析结果里不再有 `httpx` → 导入链在 `request_logger.py:7` 直接抛 `ModuleNotFoundError: No module named 'httpx'`。

两种处置，任选其一但都需要单独验：把上界写进声明（如 `openai>=1.97.1,<3`），或让 `request_logger` 适配 openai 3.x 的传输层。只补一条 `httpx` 声明**不一定够**，因为传给 openai 客户端的自定义 httpx 实例可能与 3.x 期望的 httpx2 不兼容。

### T5 既有测试基线（勿误判为回归）

`pytest` 稳定存在 36 条 `FAILED`/`ERROR`，与 Python 版本无关，成因是：

- `tests/test_asr/test_chunking.py` 仍向 `ChunkedASR` 传已改名的参数（应为 `audio_path`，测试里还是 `audio_input`）
- Bing 翻译的 `edge.microsoft.com/translate/auth` 返回 404
- 需要 LLM API key 的用例在无 key 环境下报 ERROR

**注意**：这条基线同时意味着 pydub 相关路径的测试覆盖不足（`test_chunking.py` 那几条一直在失败），所以该链路的正确性目前依赖手工验证，修这批测试本身就是 T3 的前置保障。

### T6 升级到 3.14 时

`audioop-lts` 的 `cp313-abi3` wheel 覆盖 3.14 标准构建，`pyqt5-sip` 也已发布 cp314 轮子。届时主要风险不在 audioop，而在 Qt5 运行时与 `PyQt5-Frameless-Window`，并且需要再次放宽 `requires-python` 上界、同步 `doctor.py` 与 CI 矩阵。

## 六、回滚方式

一次 `git revert` 即可，涉及 5 个文件：`pyproject.toml`、`uv.lock`、`videocaptioner/cli/commands/doctor.py`、`.github/workflows/ci.yml`、`docs/guide/getting-started.md`。回滚后 3.13 用户重新被 `requires-python` 挡在门外，3.10–3.12 用户不受影响。
