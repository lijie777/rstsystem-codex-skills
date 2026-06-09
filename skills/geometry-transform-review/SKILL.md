---
name: geometry-transform-review
description: Use when reviewing coordinate transforms, pose matrices, registration, Eigen/vtkMatrix conversions, DICOM LPS/RAS/IJK mapping, quaternions, angle units, or navigation geometry.
---

# 坐标系 / 几何变换正确性审查 (geometry-transform-review)

导航系统的全部价值是**把术者的工具/机械臂准确映射到病人解剖位置上**。这条映射是一串坐标系变换的连乘——任何一环方向搞反、单位搞错、矩阵序搞乱、四元数没归一化，都不会崩溃、不会报错，只会**悄悄打偏几毫米到几厘米**，而几毫米在脊柱手术里就是椎弓根穿破。这类 bug 编译器和功能测试都抓不到，只能靠戴上"坐标系/约定"这副眼镜逐项核对。

## 本项目的坐标系约定（审查锚点，先记住）

项目采用规整的 **`A2BMatrix` / `A2B` = "A 坐标系 → B 坐标系"右乘** 约定。核心坐标系：

| 坐标系 | 含义 | 典型命名 |
|---|---|---|
| **CT** | 影像/病人解剖坐标系（规划基准） | `getSpine2CtMatrix`、`screwDriver2Ct`、`toolTipCt` |
| **Spine/SpineBase** | 棘突示踪器（配准目标） | `getTool2SPineMatrix`、`getFlange2SPineMatrix` |
| **Camera** | NDI 光学相机坐标系 | `trackerToCameraTransform`、`tCameraProbe`(camera←probe) |
| **Tracker/Tool** | 单个示踪器/工具 | `XJTToolInfo`、`NDI2Ref_Trans` |
| **Flange** | 机械臂末端法兰 | `getScrewDriver2FlangeMatrix` |
| **Plate** | 参考板示踪器 | `tCameraPlate`、`m_lastConfirmedCtPlateMatrix` |
| **World/LPS/RAS/VTK** | VTK 渲染世界 = MITK physical = LPS；影像另有 RAS/VTK 约定 | `worldCoords`、`ras2lps` |

命名规则：`数字2`=to(`Tool2Ct`)、`A2B_Trans`、或 `tXY`(`tCameraProbe`=camera←probe)。**最该查的不变量：命名里的方向，必须与实现里的乘法顺序/`inverse()` 方向一致。** 几何核心库在 `src/Algorithm/GeometryAlgorithm/Transforms3d.{h,cpp}`。

## 审查前先确认三件事
1. **每个矩阵/位姿变量是"从哪到哪"**？按命名约定写出它的方向，再去代码里核对乘法与求逆是否真的实现了这个方向。
2. **这个 4x4 此刻是什么内存布局**？Eigen(列主序) / vtkMatrix4x4(行主序、(row,col)接口) / float[16](本项目行序、列序两种都有) / 序列化字节。跨布局传递时该转置吗？
3. **角度量此刻是度还是弧度**？函数期望的单位 vs 调用方传入的单位。

## Checklist

### 1. 变换链方向与命名一致性 — 最高优先级
- [ ] **`A2BMatrix` 命名方向必须与乘法/求逆方向一致**。正例(金标准) `getScrewDriver2CtMatrix = getSpine2CtMatrix() * getFlange2SPineMatrix() * getScrewDriver2FlangeMatrix()`——右乘链与注释 ScrewDriver→Flange→SpineBase→CT 严格对应。审查每条连乘：把每个因子的方向写出来，相邻因子的"接缝坐标系"必须接得上(`X2Y * W2X` 合法，`X2Y * Y2Z` 写成 `Y2Z * X2Y` 就反了)。
- [ ] **可疑的命名/用途反向**：`reg->getCtSpineMatrix(matrix)` 取出的矩阵却被当 `Spine2Ct` 用(`getSpine2CtMatrix` 内部调它)。命名 `CtSpine` 与用途 `Spine2Ct` 方向相反——这种地方必须读实现确认到底哪个方向，并提请改名。
- [ ] **`inverse()` 换基**：`tPlateCamera = tCameraPlate.inverse(); tPlateProbe = tPlateCamera * tCameraProbe;` 是正确范式。审查 `.inverse()` 出现处，确认求逆的是该求逆的那个、且方向需求确实是反过来。

### 2. Eigen 矩阵 / 四元数用法
- [ ] **硬件四元数转旋转矩阵前必须 `.normalize()`**。⚠️ 反模式：OTS/跟踪采集、安全空间或工具转换模块里重复实现 `ToolInfo -> Matrix4d`，直接 `quat=(qx,qy,qz,q0)` 后 `Quat2Mat(quat)`，**未归一化**，NDI/跟踪器量化误差直接进旋转矩阵、矩阵非正交、逐级链乘放大。正例：探针/工具预览姿态计算中先 `probeQuat.normalize()` 再 `toRotationMatrix()`。这是导航最底层函数，影响全系统精度。
- [ ] **欧拉角函数轴序/单位不一致**。⚠️ `Transforms3d.cpp`：`Mat2Euler` 用 `eulerAngles(0,1,2)`(XYZ)、`Mat2EulerAngle` 用 `eulerAngles(2,1,0)`(ZYX)——**同库两函数轴序相反**；`eulerAngles` 本身有 ±π/万向锁歧义。审查欧拉角往返转换是否同轴序、是否必要(能用矩阵/四元数就别转欧拉角)。
- [ ] **`inverse()` vs `transpose()`**：纯旋转正交矩阵两者等价但 `transpose()` 更快更稳；含平移的齐次矩阵**只能 `inverse()`**，用 `transpose()` 是错的。确认对 4x4 齐次矩阵没用 transpose 当逆。
- [ ] **矩阵未初始化**：手填部分元素前应 `setIdentity()`。`RAASettingSecuritySpace::toolinfo2Matrix4d` 的 `Matrix4d m_NDI2Ref_Trans` 未 setIdentity(此处恰好全覆盖，但属脆弱写法)。

### 3. 行序 / 列序 / 矩阵约定混用 — 最高危
- [ ] **float[16] 的行序与列序在同一文件并存，仅靠注释区分**。⚠️ `SurgicalNavigatWidget`：`copyEigenMatrixToColumnMajorFloat`/`matrix4x4ColumnMajorToEigen`(列序，规划螺钉/配准下发) 与 `SurgicalNavigatWidget.NavTools.cpp` 的 `matrix(row,col)=pose[row*4+col]`(行序，机械臂工具标定，注释明确"不能复用规划螺钉的列序转换")。复制粘贴串用 = 整个矩阵转置 = 姿态彻底错。审查每个 `float[16]`↔矩阵转换：`idx = r*4+c`(行序) 还是 `c*4+r`(列序)，与对端接口约定是否一致。
- [ ] **vtkMatrix4x4 ↔ Eigen**：`vtkMatrix4x4::SetElement(r,c,..)`/`GetElement(r,c)` 是 (行,列) 语义、内部行主序。**逐元素 `(r,c)` 拷贝安全；整块 `memcpy`/`GetData()` 直接拷会转置出错**。区分这两种写法。
- [ ] **裸 `reinterpret_cast`/`memcpy` 序列化矩阵**。⚠️ `commonData.h` 的 `eigenMatrix4dToQVariant` 直接按 Eigen 内存(默认列主序 double)`memcpy` 进 QByteArray 再 cast 回——跨端/跨 Eigen 版本/对齐无校验。审查这类序列化是否两端布局假设一致。

### 4. DICOM / 影像方向（LPS/RAS/VTK/NIfTI/IJK）
- [ ] **LPS/RAS/VTK/NIfTI 转换成套出现，不能漏环**。正例 `SegmentDataImpl.cpp`：`ConstructVTKtoNiftiTransform(...)` + `ras2lps`(翻转 X、Y) 组成 `vtk2mitkPhysical`。业务约定：**STL 网格存为 ITK-SNAP VTK 坐标、加载回 SNAP 用 NIfTI/RAS**——任何读写 `spine_*.stl` 的代码都必须配对此转换，漏掉=网格错位。
- [ ] **派生影像必须同时复制 origin + spacing + direction 三件套**。正例 `RegionSegment.cpp`/`xjtSegmentDataProcessService.cxx` 三者成套 set。反模式信号：新建/裁剪/重采样影像只 set 了其中一两个 → 几何错位。
- [ ] **体素↔物理点用官方 API + 有限值/越界检查**。正例 `casepreviewview.cpp`：`TransformPhysicalPointToContinuousIndex` 后 `std::isfinite` 检查 + extent 越界裁剪。反模式信号：手算 `index = (world-origin)/spacing` 忽略 direction 矩阵。

### 5. 单位混淆 mm / 度 / 弧度
- [ ] **角度/弧度入口靠命名前缀区分，传错即全姿态错**。⚠️ `Transforms3d.cpp`：`EulerAngle2Mat`(入参**度**，内部 `Degrees()` 转弧度) vs `Euler2Mat`(入参**弧度**直接用)；`rotate()` 入参度、`AngleAxisPos_to_T` 入参弧度。审查每个调用：函数期望单位 vs 实参单位是否一致。
- [ ] **角度换算常量散落、多份 `#define`**。几何库、动力工具、安全空间标定等模块各自 `#define pi_df/rad_df/deg_df`，新代码又用 `Constants::kPi`——混用易错。建议统一常量。
- [ ] **距离单位**：项目统一 mm（`*Mm` 后缀是好范式）。审查无后缀的长度量是否混入 m/其它。

### 6. 数值健壮性
- [ ] **`acos` 入参必须 `std::clamp(x, -1.0, 1.0)`**。⚠️ 反模式 `QmitkRenderSceneManager.cpp`：`acos(yAxis.dot(dir)/dir.norm())` 无 clamp，浮点误差使入参 >1 → NaN；`Transforms3d.cpp` 轴角 `acos((r11+r22+r33-1)/2)` 同样。正例 `SurgicalNavigatWidget::xAxisDeviationDeg`：`std::clamp(cos,-1,1)` + norm epsilon。
- [ ] **除以模长/向量归一化前查零模长，用 epsilon 而非 `==0`**。⚠️ `angleBetweenVectors`：`v1.dot(v2)/(v1.norm()*v2.norm())` 未查零模、且**只夹上界**(`cosAngle<-1` 时仍 NaN)；`buildRightCoordinateSystem` 用 `if(ydir.norm()==0)`(浮点 `==0` 不可靠)。正例 `SurgicalNavigatToolUtils.cpp`：`isfinite(norm) && norm > kVectorNormEpsilon`。
- [ ] **矩阵→轴角 `1/(2*sin(theta))` 在 theta≈0/π 除零→inf**。`Transforms3d.cpp`、`SecurityPanelCal.cpp`：theta 接近 0 或 π 时需特判。
- [ ] **配准/位姿矩阵进入计算前用 `isValidPoseMatrix` 校验**（有限值 + 末行 [0,0,0,1] + 旋转块行列式 > 阈值），无效返回 `Matrix4d::Zero()` 并被下游拦截——正确防御范式，审查关键变换是否都过了这道校验。

## 本项目易错点速查（识别这些"形状"）

| 形状 | 在哪 | 风险 |
|---|---|---|
| 连乘的相邻因子"接缝坐标系"接不上 / 命名与乘序方向相反 | 各 `getXxx2YyyMatrix` 连乘 | 变换链方向反，整体打偏 |
| 命名 `CtSpine` 却当 `Spine2Ct` 用 | `reg->getCtSpineMatrix` | 命名误导，方向存疑必须读实现 |
| 硬件四元数转矩阵前未 `.normalize()` | `Toolinfo2Matrix4d`(3处重复) vs 正例 `probeQuat.normalize()` | 旋转矩阵非正交，误差逐级放大 |
| float[16] 行序/列序转换串用 | `copyEigenMatrixToColumnMajorFloat`(列) vs NavTools `pose[r*4+c]`(行) | 整块转置，姿态彻底错 |
| vtkMatrix 整块 memcpy/GetData 转 Eigen | vtk↔Eigen 转换处 | 行主序↔列主序未转置 |
| `reinterpret_cast` 按内存布局序列化 4x4 | `commonData.h::eigenMatrix4dToQVariant` | 跨端/版本/对齐无校验 |
| `EulerAngle2Mat`(度) 与 `Euler2Mat`(弧度) 传错单位 | `Transforms3d.cpp` | 角度差 π/180 倍，全姿态错 |
| 同库 `Mat2Euler`(XYZ) vs `Mat2EulerAngle`(ZYX) 轴序不一致 | `Transforms3d.cpp` | 欧拉角往返不可逆 |
| `acos` 入参不 clamp | `QmitkRenderSceneManager.cpp`、`Transforms3d.cpp` 轴角 | 浮点越界→NaN |
| 除模长前不查零 / 浮点 `norm()==0` / 只夹 cos 上界 | `angleBetweenVectors`、`buildRightCoordinateSystem` | 零向量→NaN/inf |
| 派生影像漏复制 origin/spacing/direction 之一 | 影像裁剪/重采样处 | 几何错位 |
| 读写 `spine_*.stl` 漏 VTK↔RAS 配对转换 | `SegmentDataImpl.cpp` 相关 | 网格相对影像错位 |

> 安全范例（"应该怎么写"的对照）：`getScrewDriver2CtMatrix` 的方向自洽连乘、`casepreviewview.cpp` 的 `probeQuat.normalize()`、`SurgicalNavigatWidget::xAxisDeviationDeg` 的 clamp+epsilon、`isValidPoseMatrix` 的矩阵有效性校验、`SegmentDataImpl.cpp` 的 LPS/RAS/VTK 成套转换。

## 输出格式（审查报告）

```
# 坐标系/几何变换审查报告

## 变换关系梳理（我的理解）
- 涉及的坐标系清单 + 每个矩阵变量的方向(A→B)
- 关键连乘链逐因子展开，标出"接缝坐标系"是否接得上
- 每个 4x4 的内存布局(Eigen列/vtk行/float[16]行或列)
- 角度量的单位(度/弧度)

## 🔴 阻断级（变换方向反 / 行列序串用 / 四元数未归一 / 单位错）
1. [文件:函数] 问题：...
   为什么会打偏（几何后果）：...
   修法：...

## 🟠 严重（acos/除零无防护→NaN / 影像三件套漏环 / STL 转换漏配对）
...

## 🟡 建议（统一角度常量 / 避免欧拉角往返 / 加 isValidPoseMatrix 校验）
...

## 验证建议
- 用单位矩阵/已知点手算一遍变换链，核对方向
- 对每个 float[16]/vtk/Eigen 转换写出索引映射确认行列序
- 检查所有 acos/归一化/求逆的数值边界防护
```

## 重要提醒
- **几何 bug 不崩溃只打偏**——必须靠"写出每个矩阵的方向并逐因子核对"，不能靠跑一遍看着像对。
- 矩阵/位姿"是否有效"的判定直接关系导航安全（无效却继续用），与 [[surgical-safety-review]] 强相关——发现 `isValidPoseMatrix`/`m_isMissing` 类校验时两边一起审。
- 四元数/矩阵常在设备 worker 线程产出、被多处共享，注意跨线程读写，配合 [[qt-review]]/[[cpp-concurrency-review]]。
- 不确定某第三方库(VTK/ITK/MITK/Eigen/NDI)的约定(行/列主序、LPS/RAS、四元数 wxyz/xyzw 顺序)时，明确提示需查该库文档，不要臆断——尤其四元数分量顺序(本项目 NDI 是 `q0,qx,qy,qz` 即 wxyz)。
