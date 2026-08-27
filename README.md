# YOLOv5 识别快捷包使用说明

这是一个由教学提供文件继续修改而来的 YOLOv5 训练快捷包。它通过多个 Python 脚本串联图片、Labelme JSON、YOLO TXT、训练/验证集与训练权重，减少手工搬运文件的步骤。

> 本 README 已按 `team.rar` 内的实际源码、配置和训练记录核对；原始说明中个别文件名与实际包内名称不同，以下以包内文件为准。该包是 2024 年暑假的初步训练成果，并非经过系统调参的完整工程。

## 获取与完整性

完整使用包位于 Git LFS 管理的 `team.rar`。克隆仓库后执行：

```bash
git lfs pull
7z x team.rar
```

- 压缩包 SHA-256：`5eb3c69e3e6bcbd779fff9ce1b22fa78565e74f68a9004790ce5b52fbeecf3e2`
- 解压后的项目根目录为 `team/`；包含源码、数据集、训练记录、权重和树莓派导出脚本。
- 图片批量命名可用其他常规工具完成。

## 先备份：部分脚本会移动文件

`imageManager.py` 会把没有同名 TXT 标签的训练图片移动到 `dataset/train/unmatched_images/`；`latroYun.py` 会把无匹配图片的 TXT 移到 `txtrds/unmatched_txt/`。运行前请复制数据集或至少备份 `json/`、`dataset/` 与 `txtrds/`。

原始备份目录为 `zhunbei/`，其中包含 JSON 和图片备份材料。

## 包内文件名与原始说明的对应

| 原始说明 | 实际包内文件/目录 | 已核对的用途 |
| --- | --- | --- |
| `dataset.py` | `datasets.py` | 包内没有 `dataset.py`；实际为 YOLOv5 数据加载实现，不是本流程的单独启动脚本。 |
| `imageManger.py` | `imageManager.py` | 整理有同名 TXT 的 JPG，并按比例复制验证集图片。 |
| `txtManager` | `dataset/txtmanger/` | 实际目录名为 `txtmanger`；拼写须与脚本保持一致。 |
| `boat.py` | `boat.py` | 将 Labelme JSON 转为 YOLO TXT。 |
| `latroYun.py` | `latroYun.py` | 将匹配的 TXT 分发到训练/验证标签目录，并生成数据集 YAML。 |
| `fanlabel.py` | `fanlabel.py` | 生成类别名称文件 `labels.txt`，不是逐图片标签转换器。 |

## 数据集制作流程

### 1. 图片、JSON 与类别编号

1. 将待训练图片放到 `dataset/train/images/`。当前脚本只查找 `.jpg` 文件。
2. 用 Labelme 框选目标并保存 JSON；将 JSON 移到根目录 `json/`。原始标注备份可放入 `zhunbei/`。
3. 运行 `boat.py`。它会扫描 `json/` 中全部 JSON，将类别名称按**排序后的名称**分配类别编号，并输出 TXT 到 `txtrds/`。
4. `boat.py` 根据 JSON 同名的 `.jpg` 在 `dataset/train/images/` 中查找图片；找不到时会跳过转换，并写入 `conversion_log.txt`。

包内的 `conversion_log.txt` 显示，部分 JSON 曾因没有同名图片而被跳过；因此在下一步前必须核对图片、JSON 与 TXT 的同名关系。

### 2. 划分验证集：`imageManager.py`

`imageManager.py` 以 `txtrds/` 的 TXT 为准：

1. 对每个 TXT 在 `dataset/train/images/` 中查找同名 `.jpg`。
2. 将找不到同名 TXT 的训练图片移动到 `dataset/train/unmatched_images/`。
3. 对剩余匹配图片随机打乱，将 **20%** 复制到 `dataset/val/images/`；训练集图片仍保留在原位置。

该脚本没有固定随机种子，也不会同步复制标签；运行后仍需执行下一步分发 TXT。

### 3. 分发 TXT 并生成 YAML：`latroYun.py`

`latroYun.py` 的实际行为如下：

1. 扫描 `txtrds/`；若 TXT 找不到对应的训练或验证图片，就把它移动到 `txtrds/unmatched_txt/`。
2. 将仍匹配的 TXT 复制到 `dataset/train/labels/` 与/或 `dataset/val/labels/`。由于验证集图片是从训练集复制而来，进入验证集的图片通常会在两个标签目录各得到一份 TXT。
3. 扫描 `json/` 中的类别，生成 `data/screentrain.yaml`，其中包含 `train`、`val`、`nc` 与 `names`。

脚本中虽定义了 `dataset/txtmanger/` 和若干中转辅助函数，但主流程不把全部 TXT 复制到该目录；不要把它误认为必要的常规标签目录。所有硬编码路径都应先替换为当前解压目录的实际位置。

### 4. 类别名称文件：`fanlabel.py`

`fanlabel.py` 从 `json/` 按首次出现顺序提取唯一类别名，并分别写入：

```text
dataset/train/labels/labels.txt
dataset/val/labels/labels.txt
```

它不会生成逐图片的 YOLO 标签。逐图片标签由 `boat.py` 生成，实际训练仍以同名图片和 TXT 是否对应为准。

## 已归档的数据集配置

包内 `data/screentrain.yaml` 记录了 7 个类别：`key`、`keyboard`、`personface`、`screen`、`stool`、`人脸`、`勺子`。文件中的训练/验证路径是旧电脑的绝对路径；复现前必须替换为当前环境路径，并确认 `nc` 与 `names` 同步更新。

## 训练、检测与导出

`train.py` 的包内默认配置为：

| 配置 | 默认值 |
| --- | --- |
| 预训练权重 | `weights/v5lite-c.pt` |
| 模型配置 | `models/v5Lite-c.yaml` |
| 数据配置 | `data/screentrain.yaml` |
| 训练轮数 | `510` |
| 批量大小 | `16` |
| 图像尺寸 | `640 × 640` |
| GPU | `0` |
| 第 494 行小改 | `--learning-rate` 默认值 `0.06` |

训练时，权重、模型 YAML、数据 YAML 和 `nc` 必须配套。原始说明中“450 轮以上成功”与归档记录一致：`exp39`、`exp40` 的配置均为 480 epochs，`exp41` 为 510 epochs。

`tryDetect.py` 默认加载 `runs/train/exp41/weights/best01.pt`，默认输入为摄像头 `0`，结果保存到 `runs/detect/`。`pipi/piOnnx.py` 读取 `best.onnx`，用于树莓派侧 ONNX Runtime 推理；导出前应确认 ONNX 文件实际存在且类别顺序一致。

## 归档训练记录

`runs/train/` 保留 `exp39`、`exp40`、`exp41` 的曲线、配置、结果文本和权重。`exp41/results.txt` 的最后一行对应第 `509/509` 个 epoch，记录的验证集指标为：

| Precision | Recall | mAP@0.5 | mAP@0.5:0.95 |
| ---: | ---: | ---: |
| 0.9277 | 0.6010 | 0.7915 | 0.6244 |

这些数值来自该归档训练运行的最后一个 epoch，不等同于其他数据集、其他划分或实际部署条件下的通用识别精度。若重新训练，请保留新的 `opt.yaml`、`results.txt`、数据 YAML 和权重，再比较结果。
