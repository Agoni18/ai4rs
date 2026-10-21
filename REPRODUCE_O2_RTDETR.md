# O2-RTDETR-R18 本地复现

本记录对应 DOTA-v1.0 单尺度模型 `o2_rtdetr_r18vd_2xb4_72e_dota.py`。

## 已验证环境

- Windows 11
- Python 3.8.20
- PyTorch 2.0.1 + CUDA 11.8
- MMEngine 0.10.7
- MMCV 2.0.1
- MMDetection 3.3.0（放在项目 `.deps`，未覆盖全局环境）
- NVIDIA GeForce RTX 4060 Laptop GPU

## 输入与产物

- 官方权重：`o2_r18_dota_epoch72.pth`
- 原始测试图：`../DOTA/test/images`，937 张 PNG
- 本地切分配置：`reproduction/ss_test_local.json`
- 切分结果：`work_dirs/dota_test_patches/images`，10,833 张图块
- 样例图：`work_dirs/reproduction/P0006_o2_rtdetr.jpg`
- 官方评测提交包：`work_dirs/reproduction/Task1/Task1.zip`

权重 SHA256：

```text
E583C605C494A490E4576AEB754CE4A0F0E0315D1C17CEC76ACC8A42BCD3464E
```

提交包 SHA256：

```text
9FC69B2A2CC9FA10B9E26178505507BBEE1BB1E208A868B5D9F42473228B180D
```

## 运行步骤

在 PowerShell 中进入本目录，将项目隔离依赖和源码加入模块搜索路径：

```powershell
$env:MPLCONFIGDIR="$PWD\work_dirs\mplconfig"
$env:PYTHONPATH="$PWD\.deps;$PWD"
```

切分 DOTA 测试图：

```powershell
python tools/data/dota/split/img_split.py --base-json reproduction/ss_test_local.json
```

单图 GPU 推理：

```powershell
python demo/image_demo.py `
  ../DOTA/test/images/P0006.png `
  projects/rotated_rtdetr/configs/o2_rtdetr_r18vd_2xb4_72e_dota.py `
  o2_r18_dota_epoch72.pth `
  --device cuda:0 `
  --out-file work_dirs/reproduction/P0006_o2_rtdetr.jpg
```

完整测试集推理与图块合并：

```powershell
python tools/test.py `
  projects/rotated_rtdetr/configs/o2_rtdetr_r18vd_2xb4_72e_dota.py `
  o2_r18_dota_epoch72.pth `
  --work-dir work_dirs/reproduction/full_test `
  --cfg-options `
  test_dataloader.dataset.data_root=work_dirs/dota_test_patches/ `
  test_dataloader.dataset.data_prefix.img_path=images/ `
  test_evaluator.outfile_prefix=work_dirs/reproduction/Task1
```

## 结果解释

本地已经完成官方权重的 GPU 推理并生成有效 DOTA 提交包。该提交包已经通过 DOTA-v1.0 官方服务器评测：

| 指标 | 复现结果 | 论文结果 | 差值 |
|---|---:|---:|---:|
| VOC mAP / AP50 | 77.3699% | 77.31% | +0.0599 |
| COCO AP50 | 77.3699% | - | - |
| COCO AP75 | 51.1180% | - | - |
| COCO mAP (AP50:95) | 48.8183% | - | - |

复现值与论文值仅相差 0.06 个百分点，并略高于论文报告值，可以认定官方权重、数据切分、推理、图块合并和提交格式均已成功复现。

### DOTA-v1.0 各类别 AP50

| 类别 | AP50 | 类别 | AP50 | 类别 | AP50 |
|---|---:|---|---:|---|---:|
| plane | 88.94% | baseball-diamond | 84.28% | bridge | 52.23% |
| ground-track-field | 71.74% | small-vehicle | 80.14% | large-vehicle | 84.95% |
| ship | 88.43% | tennis-court | 90.87% | basketball-court | 87.90% |
| storage-tank | 85.53% | soccer-ball-field | 57.81% | roundabout | 66.50% |
| harbor | 77.14% | swimming-pool | 74.83% | helicopter | 69.26% |

最高类别是 tennis-court（90.87%），其次是 plane（88.94%）和 ship（88.43%）；较困难类别为 bridge（52.23%）和 soccer-ball-field（57.81%）。前者形态细长、尺度跨度大且背景干扰强，后者边界和方向信息相对模糊。

从头训练需使用 train+val 切分后的 `trainval` 数据，论文设置为 2 张 RTX 2080 Ti、总 batch size 8、72 epoch。单张 8 GB RTX 4060 可以调整为更小 batch 并使用梯度累积，但这会改变批归一化与优化轨迹，属于单卡近似复现，而非完全相同硬件条件复现。
