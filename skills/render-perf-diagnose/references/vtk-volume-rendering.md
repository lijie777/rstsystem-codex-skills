# VTK 体绘制 (Volume Rendering) 性能深化参考

当主诊断定位到「体绘制慢 / 体数据交互卡 / 体绘制占满帧时间」时读本文件。按顺序排查。

## 目录
1. Mapper 选型（最常见真因：用了 CPU mapper）
2. 采样步长与交互降质
3. 传输函数 / shading
4. 显存与大体数据
5. Blend 模式
6. 快速诊断顺序

## 1. Mapper 选型 —— 先确认是不是 GPU 在画
体绘制"巨卡"最常见的原因是**实际跑在 CPU ray cast 上**。
- `vtkGPUVolumeRayCastMapper`：GPU ray casting，绝大多数现代场景的正确选择。
- `vtkFixedPointVolumeRayCastMapper`：**CPU**（SMP 多线程），无合适 GPU 时的回退，慢一个量级——别在生产交互里用它。
- `vtkSmartVolumeMapper`：**推荐默认**。自动在 GPU / CPU(FixedPoint) / RayCast 间选；用 `SetRequestedRenderMode(vtkSmartVolumeMapper::GPURenderMode)` 可强制 GPU（强制后若不支持会报错，正好暴露问题）。
- `vtkMultiBlockVolumeMapper` / `vtkOpenGLGPUVolumeRayCastMapper`：多块/底层实现。
- **排查动作**：打印实际使用的 mapper 类型；用 SmartVolumeMapper 时查 `GetLastUsedRenderMode()` 确认真的走了 GPU。

## 2. 采样步长与交互降质 —— 质量/速度的主旋钮
- `SetSampleDistance(d)`：沿光线的采样间隔。太小→极慢但精细；过大→有条带(banding)。通常设为体素尺寸量级。
- `SetAutoAdjustSampleDistances(true)`：让 mapper 在交互时自动放大步长换流畅，停手后恢复精细——**交互卡时优先开这个**。
- `SetImageSampleDistance(n)`：屏幕方向降采样（每 n 像素发一条光线），>1 在交互时显著提速。
- `vtkSmartVolumeMapper::SetInteractiveUpdateRate()` / 交互时的 `InteractiveAdjustSampleDistances`：配合交互器目标帧率。
- 交互器：`interactor->SetDesiredUpdateRate(高值)`（如 15）+ `SetStillUpdateRate(低值)`（如 0.5），让"动时降质、停时精细"。

## 3. 传输函数 / shading
- opacity / color transfer function（`vtkPiecewiseFunction` / `vtkColorTransferFunction`）**频繁更新会触发内部重建/重传**——拖动传输函数编辑器时考虑节流。
- `SetShade(1)` 开光照需要算**梯度**，更贵；不需要立体感时关掉。
- `SetScalarOpacityUnitDistance()` 影响不透明度积分，设不当会让体看起来过实/过虚并间接逼你调小步长。
- `SetUseJittering(1)`（GPU mapper）可用较大步长同时抑制条带，是性价比高的设置。

## 4. 显存与大体数据
- 估算：`dim_x*dim_y*dim_z * 每体素字节`（short=2B、float=4B）。再加梯度/掩膜可能翻倍。超显存→驱动换页或失败、帧时间剧烈抖动。
- 对策：
  - **降位深**：float→short/uchar（若动态范围允许）。
  - **裁剪**：`vtkVolumeMapper::SetCropping(1)` + cropping region，或 clipping planes，只画 ROI。
  - **降采样 / 多分辨率**：交互用低分辨率体，静止换高分辨率。
  - mask volume / `vtkGPUVolumeRayCastMapper::SetMaskInput` 限制计算区域。

## 5. Blend 模式
- `SetBlendModeToComposite()`（标准）/ `MaximumIntensity`(MIP) / `MinimumIntensity` / `AdditiveBlendMode`。
- MIP/MinIP 通常比 composite 便宜（无需积分不透明度）；若需求允许，切换 blend 模式本身也是优化手段。

## 6. 快速诊断顺序
1. 确认 mapper 真的是 GPU（第 1 节）——这一步常常直接解决问题。
2. 开 `AutoAdjustSampleDistances` + 交互器 `DesiredUpdateRate`，看交互是否流畅（第 2 节）。
3. 关 shading / 减少传输函数实时更新（第 3 节）。
4. 估显存，超了就降位深/裁剪/降采样（第 4 节）。
5. 仍慢→用 `vtkTimerLog` 确认体绘制 vs 其它 actor 的耗时占比，必要时 RenderDoc 抓 GL。
