---
name: render-perf-diagnose
description: Use when diagnosing ITK, VTK, MITK, ITK-SNAP, volume rendering, reslice, mesh rendering, RenderingManager, low FPS, laggy interaction, or medical visualization performance.
---

# 渲染性能诊断 · 医学影像栈 (render-perf-diagnose)

这套栈（ITK 处理 → VTK 渲染 → MITK 框架，ITK-SNAP 同源）和游戏引擎的渲染优化逻辑**很不一样**：瓶颈往往不在 GPU shader，而在**管线被无谓地反复重算**、**大体数据/网格规模**、**每次切片/窗宽窗位都重采样**、**MITK 过度刷新所有窗口**。所以铁律仍是 **measure first**：先用计时器把时间归到「ITK 处理 / VTK 渲染 / MITK 刷新 / 数据规模」哪一层，再针对性优化。任何"我觉得是 XX 慢"都要先有 `vtkTimerLog` / `itk::TimeProbe` 的数据支撑。

## 诊断流程（务必按顺序）

### 第 0 步：先分清是"哪种慢"
- **首次加载慢**（打开数据/重建面/首帧）→ 多半在 ITK 处理或 VTK 首次 pipeline 执行 / 上传。
- **交互卡**（旋转/缩放/滚切片/拖窗宽窗位时掉帧）→ 多半在 VTK 每帧重算 / 体绘制采样 / reslice / MITK 重复刷新。
- **稳态低帧**→ GPU 渲染本身（大网格、体绘制步长、depth peeling）。
区分清楚再往下，否则方向全错。

### 第 1 步：用计时器把时间归层（关键，别跳过）
- **VTK 渲染耗时**：`vtkTimerLog`（`MarkStartEvent/MarkEndEvent`、`GetElapsedTime`），或 `renderWindow->Render()` 前后计时；renderer 的 `GetLastRenderTimeInSeconds()`。
- **ITK 处理耗时**：`itk::TimeProbe` 包住每个 filter 的 `Update()`，看哪个 filter / 哪次重算最贵。
- **MITK 刷新**：统计 `RenderingManager` 的 update 次数与每个 `QmitkRenderWindow` 的渲染耗时——常见问题是**一次操作触发了过多次重渲**。
- 判定：把总耗时拆成「ITK Update + VTK Render + 框架开销」三块，看大头在哪。

### 第 2 步：按对应层逐项排查，每次只改一项并重测。

## Checklist

### A. ITK 处理管线（首次加载/重建慢，CPU）
- [ ] **无谓重算 / Modified 级联**：上游 filter 一被 `Modified()`，整条下游在下次 `Update()` 全部重算。是否因为某个参数/输入被频繁标脏，导致每次交互都重跑整条处理链？把不变的部分缓存、断开标脏传播。
- [ ] **多线程 filter 线程数**：`itk::MultiThreaderBase::SetGlobalDefaultNumberOfThreads()` / filter 的 `SetNumberOfWorkUnits()`（旧版 `SetNumberOfThreads`）是否被限制成 1，或被设得过大导致争抢？默认通常合理，但被人为改过要查。
- [ ] **是否可流式 (streaming)**：超大图像用 `itk::StreamingImageFilter` 分块处理，降内存峰值、避免一次性巨分配。
- [ ] **In-place filter**：可原地处理的 filter（很多一元 filter）是否启用 in-place，省一份大图拷贝？
- [ ] **多余的深拷贝 / 类型转换**：`CastImageFilter`、`DeepCopy`、ITK↔VTK 互转(`itkImageToVTKImageFilter`/反向)是否在每帧/每次交互发生？应只在数据真正变化时做一次。
- [ ] **RequestedRegion**：是否每次都处理整图而本可只处理当前切片/ROI？

### B. VTK 渲染管线
- [ ] **pipeline 每帧重执行**：mapper 上游被 `Modified()` 会触发 `Update()` 重新跑。交互时（仅相机变化）不应重跑数据处理。对静态几何 mapper 设 `SetStatic(true)`，避免每帧检查/重建。
- [ ] **mapper 类型与上传**：VTK OpenGL2 后端用 VBO 缓存几何；确认没有用废弃的 immediate-mode。几何不变时数据应留在 GPU，不每帧重传。
- [ ] **大 polydata（marching cubes 出来的面）**：三角形是否百万级而未抽稀？用 `vtkDecimatePro` / `vtkQuadricClustering` / `vtkQuadricDecimation` 抽稀；交互时用 **`vtkLODActor` / `vtkQuadricLODActor`** 自动降细节。
- [ ] **体绘制 (volume rendering)**：
  - 用 **`vtkGPUVolumeRayCastMapper`**（GPU）还是 `vtkFixedPointVolumeRayCastMapper`（CPU，慢得多）？或 `vtkSmartVolumeMapper` 让它自动选。CPU ray cast 是常见的"体绘制巨卡"元凶。
  - **采样步长**：`SampleDistance` / `ImageSampleDistance` 太小→极慢但更精细。开 `AutoAdjustSampleDistances` + 交互时用 `InteractiveAdjustSampleDistances`/降采样换流畅。
  - **显存**：volume 尺寸 × 类型是否超显存→换页/失败。考虑降位深、裁剪 ROI、多分辨率。
  - 传输函数 / 梯度 shading：开 shading 要算梯度，更贵；传输函数频繁更新会触发重建。
- [ ] **半透明 / depth peeling**：叠加半透明（如 overlay 分割）开了 `SetUseDepthPeeling`？peel 层数(`SetMaximumNumberOfPeels`)和 `OcclusionRatio` 直接决定成本。确认是否真需要，或用更便宜的排序/OIT 近似。
- [ ] **2D 切片 reslice**：`vtkImageReslice` 的插值模式——`Cubic` 比 `Linear`/`NearestNeighbor` 贵很多。滚切片/拖窗宽窗位时是否每次都全幅重采样？交互时可先用低质插值。
- [ ] **标量/着色**：不需要按标量着色时 `SetScalarVisibility(false)`；lookup table / color mapping 是否每帧重建？
- [ ] **渲染窗口设置**：`SetMultiSamples`(MSAA) 开太高？设 `SetDesiredUpdateRate()`（交互时给高目标帧率→触发 LOD 降质，停手后再精细渲染）。
- [ ] **相机/重置**：是否每帧误调 `ResetCamera()` / 重建 mapper？

### C. MITK 框架层
- [ ] **过度刷新**：用 `RenderingManager::GetInstance()->RequestUpdate(renderWindow)` 只刷需要的窗口，避免动不动 `RequestUpdateAll()` 或 `ForceImmediateUpdate()`（强制同步，绕过合并，交互中很伤）。统计一次操作触发了几次重渲——常见是一个属性变更连锁刷新全部 2D+3D 窗口。
- [ ] **DataNode 属性变更**：改一个 node 属性（颜色/可见性/透明度）是否触发了比必要更大范围的重渲？确认 `Modified()` 传播范围。
- [ ] **切片 level-window 重采样**：2D mapper 在窗宽窗位/切片变化时会经 reslice 重采样当前切片（见 B 的 reslice 项）。拖动时考虑节流/低质预览。
- [ ] **多渲染窗口同步**：多个 `QmitkRenderWindow`（轴/冠/矢 + 3D）是否被不必要地一起刷新？只更新当前交互的视图。
- [ ] **纹理插值属性**：DataNode 的 texture interpolation、2D 显示插值设置对大图有成本。

### D. 数据规模 / 内存 / 传输（贯穿全栈）
- [ ] 体数据是否超显存 → 降位深 / 裁剪 ROI / 多分辨率金字塔。
- [ ] 面网格三角形数是否远超需要 → 抽稀 + LOD。
- [ ] **ITK↔VTK 互转 / GPU 上传**是否只在数据变化时发生一次，而非每次交互？这是医学影像里最常见的隐形浪费。
- [ ] 大图像的内存峰值是否触发系统换页（之后帧时间剧烈抖动）。

### E. 底层兜底（当上面都排除后，确认是 GPU 本身）
- [ ] **CPU-bound vs GPU-bound**：降低渲染窗口分辨率——明显变快＝GPU 像素侧（体绘制采样/填充/depth peeling）；几乎不变＝CPU（ITK 处理 / VTK pipeline 重算 / MITK 刷新）。
- [ ] 仍需细查 GPU 时，可用 RenderDoc 抓 VTK 的 OpenGL 调用看 draw/状态切换、用 `vtkOpenGLRenderWindow::ReportCapabilities()` 看上下文与显存。

## 深入参考（按需加载）

以下两份 reference 平时**不**加载；当诊断进入对应深水区、主 checklist 的某项需要更细的 API 级清单时，**再读取对应文件**：

- **体绘制相关**（mapper 是不是真 GPU、采样步长/交互降质、传输函数与 shading、显存/大体数据、blend 模式）
  → 读 `references/vtk-volume-rendering.md`
- **MITK 刷新相关**（RenderingManager 过度刷新、切片/窗宽窗位 reslice、多渲染窗口、DataNode 属性触发范围、与后台线程交互）
  → 读 `references/mitk-rendering.md`

主 checklist 够用时不必读它们——只在需要把某一类问题挖到底时展开，避免无谓占用上下文。

## 输出格式（诊断报告）

```
# 渲染/处理性能诊断报告（ITK/VTK/MITK）

## 现象归类
- 哪种慢：首次加载 / 交互卡（滚切片/旋转/拖窗位）/ 稳态低帧
- 目标帧预算 vs 当前耗时

## 计时归层（证据）
- ITK 处理(itk::TimeProbe)：X ms（哪个 filter）
- VTK 渲染(vtkTimerLog / GetLastRenderTimeInSeconds)：Y ms
- MITK 刷新：一次操作触发 N 次重渲，每次 Z ms
- 结论：瓶颈在 <ITK处理 / VTK渲染 / MITK刷新 / 数据规模>

## 🔴 主瓶颈（按预计收益排序）
1. [层/类] 现象 + 计时证据
   优化方向：（如 CPU→GPU volume mapper / 抽稀+LODActor / 只 RequestUpdate 当前窗口 / 交互降采样）
   预计收益：高/中/低

## 🟠 次要热点
...

## 测量与验证建议
- 还需补的计时：用 vtkTimerLog / itk::TimeProbe 包住 <具体环节>
- 每次只改一项 + 重测，确认收益再继续
```

## 重要提醒
- **绝不在没有计时数据时下"应该是 XX 慢"的结论**。没数据就先给"需要在 ITK/VTK/MITK 哪几处插 `TimeProbe`/`vtkTimerLog`"的测量方案。
- 这套栈最常见的真因是 **"无谓重算 + 过度刷新 + 数据没抽稀/体绘制用了 CPU mapper"**，而不是 shader——别一上来就钻 GPU。
- 涉及多线程 filter / 后台处理线程与渲染线程交互时，配合 [[cpp-concurrency-review]]；MITK/Qt 的渲染窗口跨线程刷新问题配合 [[qt-review]]。
- 不确定某 VTK/ITK/MITK 类的确切行为（默认 mapper、是否 GPU、默认线程数）时，明确标注"需查该类文档/在目标硬件实测"，不要替它担保。
