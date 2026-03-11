# HFD Heat Field Simulator — 方法与实现文档

> 本文档详细描述 HFD（Heat Field Deformation）热场模拟器的物理模型、数值方法、数据处理算法及可视化实现，供论文撰写参考。

---

## 1 物理模型

### 1.1 控制方程

模拟器求解的是二维各向异性对流-扩散方程（2D anisotropic advection-diffusion equation），描述含液流边材中的热量传输：

$$
\frac{\partial T}{\partial t} = D_{tg}\frac{\partial^2 T}{\partial x^2} + D_{ax}\frac{\partial^2 T}{\partial y^2} - v_{th}\frac{\partial T}{\partial y} + \frac{q}{\rho c \cdot \Delta x^2}\,\delta(x_H,y_H)
$$

其中：

| 符号 | 含义 | 单位 |
|------|------|------|
| $T$ | 温度 | °C |
| $x$ | 切向坐标（tangential） | m |
| $y$ | 轴向坐标（axial，液流方向） | m |
| $D_{tg}$ | 切向热扩散系数 | m²/s |
| $D_{ax}$ | 轴向热扩散系数 | m²/s |
| $v_{th}$ | 热速度（thermal velocity） | m/s |
| $q$ | 线热源功率密度（经端部效应修正） | W/m |
| $\rho c$ | 边材容积热容 | J/(m³·K) |
| $\delta(x_H,y_H)$ | 加热针位置的 Dirac 源项 | — |

### 1.2 边材热物性计算

基于木材含水率 $\theta_w$（m³/m³）和干密度 $\rho_{dry}$（kg/m³），采用经验公式计算：

**导热系数（Thermal conductivity）**：

$$
k_{sw} = 0.10 + 0.20 \cdot G_b + 0.60 \cdot \theta_w \quad (\text{W/m/K})
$$

其中 $G_b = \rho_{dry} / \rho_w$ 为基本密度比（basic specific gravity），$\rho_w = 1000$ kg/m³。

**容积热容（Volumetric heat capacity）**：

$$
\rho c_{sw} = \rho_{dry} \cdot c_{dry} + \rho_w \cdot c_w \cdot \theta_w \quad (\text{J/m³/K})
$$

其中 $c_{dry} = 1200$ J/(kg·K)，$c_w = 4186$ J/(kg·K)。

**热扩散系数**：

$$
D_{tg} = \frac{k_{sw}}{\rho c_{sw}}, \quad D_{ax} = D_{tg} \cdot \alpha
$$

$\alpha = D_{ax}/D_{tg}$ 为热扩散各向异性比，默认值 2.0（木材沿纤维方向导热更快）。

### 1.3 热速度

将液流体积速率 $v_{sap}$（cm/h）转化为热速度 $v_{th}$（m/s）：

$$
v_{th} = \frac{\rho_w \cdot c_w}{\rho c_{sw}} \cdot v_{sap}
$$

热速度表征液流对热场的对流传输强度，是有限差分方程中对流项的系数。

### 1.4 加热模型

加热探针视为轴向无限长线热源（line heat source），焦耳热功率为：

$$
P = I^2 R \quad (\text{W})
$$

其中电流 $I = \min(I_{slider},\, V/R)$，受供电电压 $V$ 限制。标称线热源密度：

$$
q_{nom} = P / L \quad (\text{W/m})
$$

$L$ 为加热探针有效长度。

**端部效应修正（End-effect correction）**：

实际加热探针有限长，靠近两端时三维散热使有效热源强度降低。采用指数衰减模型：

$$
f_{end} = \min\left(1,\; 1 - e^{-d/r_0} - e^{-(L-d)/r_0}\right)
$$

其中 $d$ 为测温点距探针外端的径向深度，$r_0 = 5$ mm 为特征衰减长度。修正后的有效热源密度：

$$
q = q_{nom} \cdot \max(0.1,\; f_{end})
$$

### 1.5 探针布局（Nadezhdina HFD 四针法）

按 Nadezhdina et al. (1998) 标准布局：

```
        T1 (+Z_ax, 上游/顺流)
        |
        |  Z_ax
        |
T3 ---- H ----          (切向 Z_tg)
        |
        |  Z_ax
        |
        T2 (-Z_ax, 下游/参考)
```

- **H**：加热探针（中心），线热源
- **T1**：上游轴向测温针，位于 H 上方 $+Z_{ax}$
- **T2**：下游轴向测温针（参考），位于 H 下方 $-Z_{ax}$
- **T3**：切向测温针，位于 H 切向 $+Z_{tg}$

### 1.6 HFD 温差指标

从四针温度计算三个关键温差：

$$
\Delta T_{sym} = T_1 - T_2 \quad (\text{对称温差})
$$
$$
\Delta T_{as} = T_3 - T_2 \quad (\text{非对称温差})
$$
$$
\Delta T_{s-a} = \Delta T_{sym} - \Delta T_{as} = T_1 - T_3
$$

以及比值 $R = \Delta T_{sym} / \Delta T_{as}$，作为 K-diagram 的横轴。

---

## 2 数值方法

### 2.1 空间离散

采用均匀正方形网格，网格间距 $\Delta x$（可调，默认 0.5 mm）。网格尺寸自动计算：

$$
N = \lfloor 70 / \Delta x_{mm} \rfloor
$$

若 $N$ 为偶数则加 1，确保中心网格点恰好位于域中心。探针位置映射为整数网格索引。

### 2.2 稳态求解（Gauss-Seidel with SOR）

稳态场 $\partial T/\partial t = 0$ 采用逐次超松弛（Successive Over-Relaxation, SOR）Gauss-Seidel 迭代法。

对流项采用**一阶迎风格式（first-order upwind scheme）**离散：

$$
a_x = \frac{D_{tg}}{\Delta x^2}, \quad a_y = \frac{D_{ax}}{\Delta x^2}, \quad b_v = \frac{v_{th}}{\Delta x}
$$

根据流向确定迎风系数：

$$
a_{y,N} = a_y - \min(b_v, 0), \quad a_{y,S} = a_y + \max(b_v, 0)
$$

$$
a_c = 2a_x + a_{y,N} + a_{y,S}
$$

Gauss-Seidel 更新公式（含源项 $S$）：

$$
T_n^{(k)} = \frac{a_x(T_{i+1,j}+T_{i-1,j}) + a_{y,N}\,T_{i,j+1} + a_{y,S}\,T_{i,j-1} + S}{a_c}
$$

SOR 加速（松弛因子 $\omega = 1.7$）：

$$
T_{i,j}^{(k+1)} = T_{i,j}^{(k)} + \omega \left(T_n^{(k)} - T_{i,j}^{(k)}\right)
$$

收敛判据为最大修正量 $\max|T^{(k+1)} - T^{(k)}| < 5 \times 10^{-8}$，最大迭代次数 60000。

**边界条件**：四边均为 Dirichlet 条件，$T = T_{ambient}$。

**暖启动（Warm start）**：在日变化回放中，前一时刻的稳态解作为下一时刻的初始猜测，大幅加速收敛。

### 2.3 瞬态求解（显式 FTCS + Upwind）

瞬态模式采用显式前向时间-中心空间（Forward Time, Central Space, FTCS）格式，对流项同样采用一阶迎风：

$$
T_{i,j}^{n+1} = T_{i,j}^n + r_x(T_{i+1,j}-2T_{i,j}+T_{i-1,j})^n + r_y(T_{i,j+1}-2T_{i,j}+T_{i,j-1})^n - C_v \cdot (\text{upwind diff})^n + S \cdot \Delta t
$$

其中：

$$
r_x = \frac{D_{tg}\,\Delta t}{\Delta x^2}, \quad r_y = \frac{D_{ax}\,\Delta t}{\Delta x^2}, \quad C_v = \frac{v_{th}\,\Delta t}{\Delta x}
$$

迎风项根据 $C_v$ 符号选择：

$$
\text{upwind diff} = \begin{cases} T_{i,j} - T_{i,j-1} & C_v \geq 0 \\ T_{i,j+1} - T_{i,j} & C_v < 0 \end{cases}
$$

**时间步长**由 CFL 稳定性条件自动确定：

$$
\Delta t = \min\left(\frac{0.2\,\Delta x^2}{\max(D_{tg}, D_{ax})},\; \frac{0.4\,\Delta x}{|v_{th}|},\; 0.5\text{s}\right)
$$

每帧计算多个子步以保证流畅动画（子步数 $\approx 20 / \max(\Delta t \times 1000, 0.01)$）。

### 2.4 数值格式选择理由

| 方面 | 选择 | 理由 |
|------|------|------|
| 对流离散 | 一阶迎风 | 无条件稳定（无非物理振荡），对 HFD 温差场足够精确 |
| 稳态求解 | Gauss-Seidel + SOR | 内存占用极小（原地更新），$\omega=1.7$ 接近最优，收敛快 |
| 瞬态求解 | 显式 FTCS | 实现简单，配合 CFL 自适应步长保证稳定性 |
| 数据结构 | Float64Array 一维展平 | 高性能连续内存访问，适合浏览器 JIT 优化 |

---

## 3 数据处理与分析

### 3.1 K 值计算（Nadezhdina 方法）

K 值是 HFD 方法的核心校正参数，代表零液流条件下的自然温度梯度贡献。

**算法流程**：

1. **筛选低流量点**：按 $|R| = |\Delta T_{sym}/\Delta T_{as}|$ 从小到大排序，取前 15%（最少 3 个点）
2. **线性回归**：以 $R$ 为自变量，$\Delta T_{as}$ 为因变量，求最小二乘回归
3. **提取截距**：$K = $ 回归截距（$R=0$ 时的 $\Delta T_{as}$ 值）

回归公式（普通最小二乘法）：

$$
K = \frac{\sum y_i \sum x_i^2 - \sum x_i \sum x_i y_i}{n \sum x_i^2 - (\sum x_i)^2}
$$

其中 $x_i = R_i$，$y_i = \Delta T_{as,i}$。当分母趋近于零时，退化为均值估计 $K = \bar{y}$。

4. **修正温差**：$K + \Delta T_{s-a}$ 用于 K-diagram 中校正后的温差线

### 3.2 日变化液流曲线生成

采用升余弦平方（raised-cosine-squared）模型生成平滑的日液流变化曲线：

$$
f(h) = \begin{cases} \cos^2\!\left(\frac{\pi(h - h_{peak})}{2\,W}\right) & |h - h_{peak}| < W \\ 0 & \text{otherwise} \end{cases}
$$

$$
v_{sap}(h) = v_{min} + (v_{peak} - v_{min}) \cdot f(h)
$$

其中 $W = 7.5$ h 为半宽度，$h_{peak}$ 为峰值时刻。时间分辨率为 10 min/点，一天 144 点。

**天气修正模式**：

| 模式 | 修正方法 |
|------|---------|
| **Sunny** | 无修正，标准 $\cos^2$ 钟形曲线 |
| **Cloudy** | 叠加高斯凹陷：$v \leftarrow v - 0.3(v_{peak}-v_{min}) \cdot \exp\!\left[-\frac{(h-12.5)^2}{2 \times 1.2^2}\right] \cdot f(h)$，模拟正午云遮 |
| **Drought** | $h > 11$h 后指数衰减：$f \leftarrow f \cdot e^{-0.25(h-11)}$，模拟干旱提前关闭气孔 |

**噪声模型**：加性均匀随机噪声 $\epsilon = v_{peak} \cdot \sigma_{noise} \cdot U(-1,1)$。

### 3.3 外部数据上传

支持 CSV/TXT 格式上传实测液流数据：

- **解析格式**：逗号/制表符/分号分隔，第一列为时间戳，第二列为 $v_{sap}$（cm/h）
- **时间戳支持**：完整日期时间（`2024-01-15 08:30`）或仅时间（`08:30`），自动处理午夜跨越
- **数据约束**：最少 144 行（1 天），最多 14400 行
- **多日支持**：自动检测多天数据，时间轴显示天数标签

---

## 4 K-Diagram 可视化

### 4.1 坐标系

K-diagram（Nadezhdina 2018）以温差比值为横轴，各温差为纵轴：

- **x 轴**：$R = \Delta T_{sym} / \Delta T_{as}$
- **y 轴**：温度（°C）

### 4.2 绘制内容

| 图层 | 数据 | 颜色 | 含义 |
|------|------|------|------|
| 散点 $\Delta T_{sym}$ | $R$ vs $T_1 - T_2$ | 黑色 (#222) | 对称温差随流量变化 |
| 散点 $\Delta T_{as}$ | $R$ vs $T_3 - T_2$ | 粉色 (#e91e90) | 非对称温差 |
| 散点 $\Delta T_{s-a}$ | $R$ vs $T_1 - T_3$ | 灰色 (#aaa) | 对称减非对称 |
| 散点 $K + \Delta T_{s-a}$ | $R$ vs $K + (T_1 - T_3)$ | 绿色 (#5a8a20) | K 校正后温差 |
| K 水平线 | $y = K$ | 红色虚线 | 零流量截距 |
| 零线 | $y = 0$ | 蓝色虚线 | 温度零参考 |

### 4.3 物理意义

- 零流量时（$R \approx 0$）：$\Delta T_{sym} \approx 0$，$\Delta T_{as}$ 和 $K + \Delta T_{s-a}$ 应在 K 值处交汇
- 高流量时（$R \gg 0$）：$\Delta T_{sym}$ 增大，热场明显向流向偏移
- $K + \Delta T_{s-a}$ 与 $\Delta T_{as}$ 的交点验证 K 值估计的准确性

---

## 5 热场可视化

### 5.1 色图（Colormap）

采用改进的 Jet 色图，将温升 $\Delta T = T - T_{ambient}$ 归一化到 $[0, 1]$ 后映射：

| 范围 $t$ | 颜色过渡 |
|----------|---------|
| $[0, 0.05)$ | 暗底 → 深蓝（从黑色渐入，避免零值处突变） |
| $[0, 0.125)$ | 深蓝 → 蓝 |
| $[0.125, 0.375)$ | 蓝 → 青 |
| $[0.375, 0.625)$ | 青 → 黄 |
| $[0.625, 0.875)$ | 黄 → 红 |
| $[0.875, 1.0]$ | 红 → 深红 |

### 5.2 双线性插值渲染

为避免网格像素化伪影，渲染时对每个屏幕像素进行双线性插值（bilinear interpolation）：

$$
T(x,y) = (1-f_x)(1-f_y)\,T_{00} + f_x(1-f_y)\,T_{10} + (1-f_x)f_y\,T_{01} + f_x f_y\,T_{11}
$$

通过 `ImageData` 逐像素写入 RGBA 数据，利用 HTML5 Canvas 2D API 的 `putImageData()` 高效渲染。

### 5.3 等温线（Isotherms）

在色图上叠加等温线，共 12 条，均匀分布于 $[T_{ambient},\, T_{ambient}+\Delta T_{max}]$。

等温线提取采用 **Marching Squares 算法**：逐网格单元检查四角温度值与等值面的交叉关系，通过 4-bit 索引（16 种情况）确定等温线段的起止端点，线性插值确定精确位置。

### 5.4 Canvas 自适应尺寸

热场画布尺寸根据浏览器窗口自适应计算：

$$
sz = \max\!\big(180,\; \min(W_{available}, H_{available}, 560)\big)
$$

窗口大小变化时自动调整并重绘。

---

## 6 时间序列图

### 6.1 双 y 轴设计

- **左 y 轴**：温差（°C），绘制 $\Delta T_{sym}$、$\Delta T_{as}$、$\Delta T_{s-a}$ 三条曲线
- **右 y 轴**：液流速率 $v_{sap}$（cm/h），绘制为绿色填充面积图
- **x 轴**：时间（h），自动适应单日或多日数据

### 6.2 刻度自适应

坐标刻度间隔通过对数量级自适应算法确定：

$$
step = \begin{cases} 0.2p & r/p < 2 \\ 0.5p & r/p < 5 \\ p & \text{otherwise} \end{cases}
\quad \text{where } p = 10^{\lfloor\log_{10}(range)\rfloor}
$$

---

## 7 技术架构

### 7.1 实现环境

| 项目 | 技术 |
|------|------|
| 语言 | JavaScript (ES6+) |
| 渲染 | HTML5 Canvas 2D API |
| 数值数组 | Float64Array（IEEE 754 双精度） |
| 动画 | requestAnimationFrame（瞬态）/ setTimeout（日变化回放） |
| 部署 | 单页面 HTML，GitHub Pages 静态托管 |
| 依赖 | 零外部依赖，完全自包含 |

### 7.2 性能优化

- **一维数组展平**：二维网格以行优先方式存储在一维 `Float64Array` 中，索引 $idx = i \times N_Y + j$，利用连续内存局部性加速缓存命中
- **SOR 暖启动**：日变化模式中复用前一稳态解作为初始猜测，相邻时刻温度场变化小，收敛所需迭代数大幅减少
- **批处理子步**：瞬态模式每帧计算多个物理时间步后再渲染一次，平衡物理精度与帧率
- **降采样绘制**：日变化预览曲线在数据点多于画布像素时自动降采样

### 7.3 输入参数范围

| 参数 | 最小值 | 最大值 | 默认值 | 单位 |
|------|--------|--------|--------|------|
| $Z_{ax}$ | 5 | 30 | 15 | mm |
| $Z_{tg}$ | 2 | 15 | 5 | mm |
| Heater L | 20 | 120 | 100 | mm |
| Supply V | 1.0 | 12.0 | 6.0 | V |
| Current I | 20 | 150 | 50 | mA |
| Resistance R | 40 | 300 | 120 | Ω |
| $\theta_w$ | 0.10 | 0.80 | 0.40 | m³/m³ |
| $\rho_{dry}$ | 200 | 900 | 500 | kg/m³ |
| $D_{ax}/D_{tg}$ | 1.0 | 4.0 | 2.0 | — |
| $v_{sap}$ | −20 | 150 | 0 | cm/h |
| $T_{ambient}$ | 0 | 40 | 20 | °C |
| Grid $\Delta$ | 0.3 | 1.0 | 0.5 | mm |

---

## 8 参考文献

1. Nadezhdina, N., Čermák, J., & Nadezhdin, V. (1998). Heat field deformation method for sap flow measurements. *Proceedings of the 4th International Workshop on Measuring Sap Flow in Intact Plants*, 72–92.
2. Nadezhdina, N., Vandegehuchte, M. W., & Steppe, K. (2012). Sap flux density measurements based on the heat field deformation method. *Trees*, 26, 1439–1448.
3. Nadezhdina, N. (2018). Revisiting the Heat Field Deformation (HFD) method for measuring sap flow. *iForest – Biogeosciences and Forestry*, 11, 118–130.
4. ICT International. HFD8-100 Heat Field Deformation Sap Flux Meter. Product specifications.
5. Patankar, S. V. (1980). *Numerical Heat Transfer and Fluid Flow*. Hemisphere Publishing.
6. LeVeque, R. J. (2007). *Finite Difference Methods for Ordinary and Partial Differential Equations*. SIAM.
