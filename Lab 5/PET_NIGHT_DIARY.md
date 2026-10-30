# Pet Night Diary — Raspberry Pi 使用指南

这个原型回答：**昨晚什么时候在指定休息区域检测到了狗？** 它不是睡眠分期或睡眠质量检测仪。狗没有被检测到，可能是遮挡、光线不足或检测遗漏，不等于离开了房间。

## 文件

- `pet_night_diary.py`：启动/停止、进程超时、JSONL 日志、早晨报告。
- `pet_detector.py`：USB 摄像头、YOLO 狗类别、ROI 和连续确认。
- `pet_display.py`：Mini PiTFT、背光和按钮；缺少库时回退到终端。
- `pet_morning.py`：统计历史日志和本地 Qwen 报告。
- `test_pet_night.py`：纯软件模拟测试，不需要相机或模型。
- `pet_requirements_display.txt`：屏幕的可选依赖。
- `pet_night_data/`：运行时创建，默认不提交到 Git。

原有 `yolo_to_qwen.py`、`overnight.py`、`morning.py` 和 Part E 不作修改。新程序沿用原有 `model.predict(..., imgsz=320)` 和本地 Ollama `/api/generate` 方式；不直接 import 原脚本，避免触发其中的顶层模型加载或采集循环。

## 1. 准备 Pi

以下命令在 **Pi 的 SSH 终端**运行，不是在 Mac 上运行。先把新增的 `pet_*.py`、`test_pet_night.py` 和本指南复制到 Pi 的 `~/Interactive-Lab-Hub/Lab 5/`。不要复制 Mac 的虚拟环境或覆盖 Pi 的原实验文件。

如果 Pi 的 SSH 地址是 `pi@raspberrypi.local`，可先在 Mac 终端复制新增文件（地址不同请替换）：

```bash
cd /Users/jindichai/Documents/ChatGPT/IDD/Interactive-Lab-Hub
scp 'Lab 5/pet_night_diary.py' 'Lab 5/pet_detector.py' 'Lab 5/pet_display.py' 'Lab 5/pet_morning.py' 'Lab 5/test_pet_night.py' 'Lab 5/pet_requirements_display.txt' 'Lab 5/PET_NIGHT_DIARY.md' 'pi@raspberrypi.local:Interactive-Lab-Hub/Lab 5/'
```

开发时从 `origin/Fall2026` 提取 Lab 5 作为基线。发布使用基于最新远端的独立检出目录，只提交新增程序、指南及 README/.gitignore 补充；其它 Lab 和原有实验脚本保持不变。

```bash
cd ~/Interactive-Lab-Hub/'Lab 5'
source .venv/bin/activate
python --version
ls yolo26n_ncnn_model
python -c 'import cv2; from ultralytics import YOLO; print("Detector imports OK")'
ollama list
```

使用已有的 Python 3.11、OpenCV、Ultralytics 和 NCNN 环境即可。默认不会下载模型。如果目录缺失，先按照 Lab 5 Part A 完成导出。

屏幕库缺失时才安装：

```bash
python -m pip install -r pet_requirements_display.txt
```

用 `sudo raspi-config` 确认 SPI 已启用。接线来自本仓库 `Lab 2/screen_test.py` 和 `dog_clock_v3.py`：

| 信号 | BCM GPIO | 物理针脚 |
| --- | --- | --- |
| CS | 5 | 29 |
| DC | 25 | 22 |
| 背光 | 22 | 15 |
| 按钮 A | 23 | 16 |
| 按钮 B | 24 | 18 |

**这里沿用 Lab 2 已改线的 GPIO5 CS，不是默认 CE0。** ST7789 初始化为 135×240、偏移 53/40，绘图以 90° 旋转为 240×135。按钮上拉、低电平按下。请关闭其它占用同一 GPIO/屏幕的实验程序。

## 2. 无硬件模拟和自动测试

```bash
python -m unittest discover -s . -p 'test_pet_night.py' -v
python pet_night_diary.py --simulate --terminal --start --seconds 30 --interval 1 --no-qwen
```

模拟序列包含检测、一次漏检、相机故障和连续未检测。所有模拟日志和报告明确标记 `SIMULATED`，不得当成真实宠物记录。模拟模式也可以去掉 `--terminal` 来测试屏幕，仍然不会使用摄像头。

## 3. 先做 60 秒真实相机测试

```bash
python pet_night_diary.py --terminal --start --seconds 60 --interval 2 --no-qwen
```

程序打印日志路径；60 秒后关闭日志并自动生成同名 `.report.json`。带 `--seconds` 的真实运行标记为 `short-session`。没有图形窗口，不需要 VNC。

让狗进入视野，保持至少三个采样，再移出视野；检查 `raw` 与 `confirmed` 的变化。可先用狗照片检查识别，但照片不证明真实狗在现场。确认无误后去掉 `--no-qwen` 测试 Ollama。第一次模型加载可能较慢；必要时增加 `--timeout 120`。

默认模型路径相对于脚本，不受启动目录影响。摄像头默认索引 0，可用 `--camera 2` 切换。相机每次采样重新打开并丢弃四帧，避免夜间暂停后读取旧缓冲帧；这也会反复触发曝光调整，需用真实相机检查低光效果。

## 4. 配置休息区域 ROI

`--roi X1 Y1 X2 Y2` 使用 0–1 的归一化坐标，原点在画面左上角。例如：

```bash
python pet_night_diary.py --terminal --start --seconds 60 --interval 2 --roi 0.2 0.3 0.9 0.95
```

规则：**检测框与 ROI 的交集面积 / 狗检测框面积 ≥ 0.5** 才计为区域内检测。用 `--overlap 0.7` 调高要求。默认 ROI 为整幅图像，只有摄像头确实只拍摄休息区域时才适合直接使用。

校准：用原有 `python snap.py --delay 3` 拍摄一张临时测试图，确定床和附近地板的矩形边界；横坐标除以图片宽、纵坐标除以图片高，填入上述参数。检查日志中归一化 `dogs[].box` 和 `roi_overlap`。校准照片由原 `snap.py` 管理，展示后自行删除；新监测程序不保存任何照片或视频。

`--conf 0.4` 控制 YOLO 置信度阈值。只保留 COCO class 16（dog），不重新训练模型。类别过滤不意味着更换了模型网络，推理仍使用原 NCNN 模型。

## 5. 按钮和真实过夜流程

先完成相机短测，再启动屏幕版本：

```bash
python pet_night_diary.py --seconds 60 --interval 2 --no-qwen
```

1. 屏幕显示 `Pet Night Diary / Press A to start`。
2. 按 A 建立新日志；第一帧推理成功后显示 `Camera + YOLO ready`，三秒后关闭背光。监测期间再次按 A 不会建立第二个会话。
3. 长按 B 至少 1.5 秒，停止采样并关闭日志，再生成报告。
4. 早晨屏幕先显示最后确认检测时间；按 A 翻页查看时长、数据质量和完整摘要，长按 B 退出程序。下一晚重新运行程序。

没有屏幕库时自动回退终端，也可用 `--terminal` 主动跳过 GPIO。终端输入 `a` 后回车启动，输入 `b` 后回车停止。`--start` 跳过启动按钮。`Ctrl-C` / SIGTERM 同样会安全结束当前会话。报告阶段 Qwen 最多等待 45 秒，期间不再采集；Ctrl-C 停止会话时跳过 Qwen。

安装 tmux（若已有则跳过）：

```bash
sudo apt install tmux
```

开始一晚：

```bash
tmux new -s pet-night
cd ~/Interactive-Lab-Hub/'Lab 5'
source .venv/bin/activate
python pet_night_diary.py --interval 10 --confirm 3 --roi 0.2 0.3 0.9 0.95
```

上例 ROI 是示例，请换成你的校准值。按 A 启动，看到就绪提示后，按 **Ctrl-B，松开，再按 D** 退出 tmux 画面。此后可以断开 SSH、关闭 Mac；Pi 必须保持供电。

无屏幕启动可替换最后一行为：

```bash
python pet_night_diary.py --terminal --start --interval 10
```

早晨重新 SSH 后：

```bash
tmux attach -t pet-night
```

长按硬件 B，或终端输入 `b` 回车，或按 Ctrl-C。不要用 `kill -9`，也不要直接断电。

## 6. 查看日志和重新生成报告

```bash
ls -lt pet_night_data/
python pet_morning.py --log pet_night_data/night_YYYYMMDD_HHMMSS_ID.jsonl
python pet_morning.py --log pet_night_data/night_YYYYMMDD_HHMMSS_ID.jsonl --no-qwen
```

把示例文件名替换为真实路径。独立报告命令只读取日志并打印结果，不修改原日志。每个会话使用时间戳和随机 ID，并以排他创建模式打开，避免覆盖历史记录。

日志含 `start`、`sample`、`transition`、`heartbeat`、`gap`、错误、恢复和 `end`。每条写入立即 flush，每 10 分钟心跳以及开始/结束时 fsync。突然断电仍可能损失尚未同步到存储介质的尾部；报告会标记损坏记录与缺少结束记录。

每条 sample 包含带时区时间、单调时钟 elapsed、原始检测、置信度、框和 ROI 交集比例、确认状态与连续次数。正常帧另记 `captured_at`；相机错误不会伪造采集时间。所有时间取 Pi 的系统时钟，请事先核对 `date`。

## 7. 过滤和统计的含义

- 默认连续 3 个有效同类观测才确认初始状态或切换状态，可用 `--confirm` 修改。
- 单次相反观测保留旧的确认状态，但不把该样本当成支持旧状态的证据。
- 相机/推理失败立即变为 `unknown`，清空累计次数，恢复后重新确认。
- 两次样本间隔超过配置间隔的 1.5 倍也清空确认，避免跨长时间缺口确认状态。
- 时长只累计相邻样本的 raw 与 confirmed 都相同、且间隔不超过 1.5 倍采样间隔的区间。采用单调时钟计算，支持跨午夜；不回填最初两次观测，也不向停止时间外推。
- 这些是采样支持的**近似区间时长**，不是持续录像所证明的时长。其余时间列为 `unclassified_seconds`，包括未确认、缺口及首尾空白。
- 最后确认检测时间来自 raw 和 confirmed 都为检测到狗的最后一条观测，单次漏检不会更新该时间。
- `.report.json` 包含所有支持区间的起止时间；空日志、全部错误和未完成会话会明确显示数据不足或警告。

Qwen 沿用本地 `qwen3.5:0.8b`，关闭 thinking。Python 先计算统计并构造三句事实；Qwen 只能按顺序使用这些原句。这是刻意保守的模板约束，不是自由发挥的睡眠分析。输出结构或文字不符、服务未启动、模型不存在、超时，都自动使用 Python 原句。照片和日志不会发送到云端。

## 8. Pi 上仍需完成的验收

| 测试 | 操作 | 预期 |
| --- | --- | --- |
| 短测 | 真实摄像头运行 60 秒 | 有有效帧、JSONL、end 和报告 |
| 变化 | 狗/照片出现和移开，各持续多次采样 | 连续阈值生效；单次漏检不立即切换 |
| ROI | 狗在框外出现 | 不算区域内检测 |
| 按钮 | A 启动、重复 A、长按 B | 一个会话、就绪后熄屏、安全停止 |
| 相机 | 拔掉 USB、等待、再插回 | unknown、重试、recovery、重新确认 |
| Ollama | 服务不可用时生成报告 | Python 回退，不丢日志 |
| 中断 | Ctrl-C 后重新读取日志 | end 存在，报告可复现 |
| 真过夜 | tmux 中运行数小时 | 心跳、样本、真实照片/视频演示证据 |

本次开发在 Mac 上运行自动化及 30 秒模拟验证。未验证真实 Pi、摄像头、GPIO、NCNN 模型执行或真实夜间数据。请把实际测试结果、图片/短视频和一晚真实日志补入 README。尤其注意：普通 USB 摄像头在黑暗环境可能完全无法识别；不能据此推断狗离开、醒来或睡眠不好。

原生相机调用超时会终止并重建工作进程（默认 60 秒）。极端内核/USB 故障可能仍需重新插拔设备。帧读取和推理耗时会使实际采样周期长于设定值；日志记录真实时间，统计排除过大缺口。

## 贡献说明

设计基于 Jindi Chai 的 Pet Night Diary storyboard、原 Lab 5 pipeline 和 Lab 2 Mini PiTFT 接线代码。使用 OpenAI ChatGPT/Codex 辅助实现代码、测试与文档。硬件校准、实地测试、测量和最终课程报告由学生完成，模拟结果不是实际实验数据。

API 参考：[Ultralytics predict](https://docs.ultralytics.com/modes/predict/)、[Ollama generate](https://docs.ollama.com/api/generate)。
