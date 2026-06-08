# MITK 渲染刷新性能深化参考

当主诊断定位到「MITK 交互卡 / 一次操作触发过多重渲 / 切片滚动或窗宽窗位拖动卡 / 多窗口刷新」时读本文件。

## 目录
1. RenderingManager 刷新策略（核心）
2. DataNode 属性变更的刷新范围
3. 2D 切片 reslice 与窗宽窗位
4. 多渲染窗口
5. 大数据节点
6. 与后台线程的交互
7. 快速诊断顺序

## 1. RenderingManager 刷新策略 —— 最常见真因
- `RequestUpdate(vtkRenderWindow*)`：只请求刷**指定**窗口，且会**合并**短时间内的多次请求（推荐）。
- `RequestUpdateAll(RenderingManager::REQUEST_UPDATE_ALL / _2DWINDOWS / _3DWINDOWS)`：刷一类/全部窗口——按需收窄到只刷必要的那类。
- `ForceImmediateUpdate(window)` / `ForceImmediateUpdateAll()`：**同步、立即、绕过合并**——在交互/循环里调用是性能杀手，只在确需即时结果（如导出截图）时用。
- **排查动作**：在一次用户操作路径上统计 `RequestUpdate*` / `ForceImmediate*` 的调用次数与目标窗口。常见 bug：改一个属性 → 连锁刷新全部 4 个窗口 N 次。

## 2. DataNode 属性变更的刷新范围
- 改 node 的 `visible` / `color` / `opacity` / `lookup table` / `texture interpolation` 等属性通常会触发重渲。确认：
  - 是否**批量改属性后只刷一次**，而不是每改一个属性刷一次。
  - 属性变更是否波及了不该重渲的窗口/节点。
- 大图像节点的 `"texture interpolation"`（线性插值）对大切片有成本，必要时设最近邻。

## 3. 2D 切片 reslice 与窗宽窗位
- 2D 显示经 reslice（`mitkExtractSliceFilter` → `vtkImageReslice`）把 3D 体重采样成当前切片平面。
- **滚切片 / 拖窗宽窗位**会反复触发重采样：
  - reslice 插值模式（`SetInterpolationMode`）：`Cubic` 远贵于 `Linear`/`NearestNeighbor`，交互时可临时降级。
  - 拖动时**节流**（throttle）或先出低质预览、停手再精细。
  - 窗宽窗位仅改的是 level-window 颜色映射时，理想情况只需重映射颜色而非整层重采样——确认没有触发不必要的几何重采样。
- 多时间步数据（TimeGeometry）：确认只处理当前时间步。

## 4. 多渲染窗口
- `QmitkStdMultiWidget` 含轴/冠/矢 3 个 2D + 1 个 3D，每个有独立 `BaseRenderer`。
- 只刷**当前交互**的视图：滚某一个 2D 窗口的切片，不必同时重渲另外两个 2D 和 3D（除非有联动需求）。
- 检查 crosshair / 联动机制是否过度触发全窗刷新。

## 5. 大数据节点
- Surface 节点三角形过多 → 在 node 的 VTK mapper/actor 上挂抽稀（`vtkDecimatePro`/`vtkQuadricClustering`）或用 LOD actor；交互降细节。
- Image 节点过大 → 见 vtk-volume-rendering.md（体绘制）或 2D reslice 降采样。

## 6. 与后台线程的交互
- MITK/VTK 渲染在**主(GUI)线程**。耗时的 ITK 处理应放 worker 线程，**完成后再回主线程** `RequestUpdate`。
- 切勿在非主线程直接操作渲染对象或 DataStorage（跨线程）——相关问题配合 [[qt-review]]（QmitkRenderWindow 跨线程/信号槽）与 [[cpp-concurrency-review]]（数据竞争）。

## 7. 快速诊断顺序
1. 统计一次操作触发了几次 `RequestUpdate*`/`ForceImmediate*`、刷了哪些窗口（第 1 节）——常常直接见真因。
2. 把 `ForceImmediateUpdate*` 换成 `RequestUpdate(当前窗口)`（第 1、4 节）。
3. 拖窗宽窗位/滚切片卡 → 降 reslice 插值 + 节流（第 3 节）。
4. 用 `vtkTimerLog` 包住 `QmitkRenderWindow` 的 render，量每窗口耗时；用 `itk::TimeProbe` 排除是不是后台处理在阻塞主线程。
