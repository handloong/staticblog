---
title: PColor 伪彩色图技术分享
date: 2026-07-20
description: 解析 ECharts 在 BI 平台中实现非规则网格伪彩色图的混合方案——以 ECharts 负责坐标系与图例，Canvas 2D API 自行渲染色彩填充层，并通过 convertToPixel 桥接两者精确对齐
toc: true
slug: pcolor-pseudocolor-map
categories:
    - Dev
    - 技术分享
    - 数据可视化
tags:
    - echarts
    - pcolormesh
    - canvas
    - 伪彩色图
    - 数据可视化
---

# 🎨 PColor 伪彩色图技术分享

> ECharts + Canvas 2D · 非规则网格填色 · BI 平台科学可视化  
> 核心思路：框架负责坐标系，自定义图层负责色彩，convertToPixel 桥接对齐

---

## 一、什么是伪彩色图

伪彩色图（Pseudocolor Map / Pcolormesh）是一种将**二维网格数值**映射为**连续色谱颜色**的可视化图表，常用于：

- 地球物理勘探（地震属性、储层分布）
- 气象热力分布（温度、风速场）
- 传感器数据的空间分布展示

与热力图的核心区别在于：伪彩色图的数据点带有**真实物理坐标**（X/Y），而不是简单的行列索引，因此需要处理不规则网格的坐标映射与空洞插值。

---

## 二、整体架构

```text
后端返回数据 (pcolorData)
    │
    ├── pcolormesh { xGrid, yGrid, zGrid }   ← 网格物理坐标 + 数值
    └── scatter    { x, y, z, point }        ← 叠加散点（可选）
    │
    ▼
PColor.drawChart()                            ← 生成 ECharts option
    ├── computeRange()                        ← 计算 x/y/z 的值域范围
    ├── parseCustomStops()                    ← 解析用户自定义色谱
    ├── 构建 xAxis / yAxis / visualMap        ← 坐标轴 + 图例
    ├── 构建 scatter series（可选叠加散点）
    └── option.__pcolorRawData                ← 附带原始数据供后续渲染
    │
    ▼
ChartComponentColor.vue（renderEcharts）
    ├── echarts.init()  renderer: 'canvas'    ← 强制 canvas 渲染器
    ├── 正方形 grid 修正                      ← 动态计算边距使绘图区为正方形
    ├── myChart.setOption(option)
    └── myChart.one('finished', ...)          ← ECharts 布局稳定后
            │
            ├── convertToPixel()             ← 获取 grid 像素坐标
            ├── renderToDataURL()            ← 将网格渲染为 PNG DataURL
            └── setOption({ graphic: [...] }) ← 将图片精确贴入 grid 区域
```

---

## 三、核心算法详解

### 3.1 色谱插值 —— `interpolateColor()`

系统内置 **Jet 色谱**，由 6 个关键色标（归一化位置 + RGB）定义：

| 位置 | 颜色     | RGB           |
|------|----------|---------------|
| 0.0  | 深蓝     | (0, 0, 143)   |
| 0.125| 蓝       | (0, 0, 255)   |
| 0.375| 青       | (0, 255, 255) |
| 0.625| 黄       | (255, 255, 0) |
| 0.875| 红       | (255, 0, 0)   |
| 1.0  | 深红     | (128, 0, 0)   |

对于任意归一化值 `t ∈ [0, 1]`，找到其所在的两个相邻色标区间，做**线性插值**：

```text
f = (t - t0) / (t1 - t0)
R = round(R0 + f × (R1 - R0))
G = round(G0 + f × (G1 - G0))
B = round(B0 + f × (B1 - B0))
```

用户也可以在图表样式配置中提供自定义颜色数组，`parseCustomStops()` 会将其等间距转换为同样格式的 stops，替换默认 Jet 色谱。

---

### 3.2 网格渲染为图片 —— `renderToDataURL()`

这是整个模块的核心，分为**有坐标**和**无坐标**两种模式。

#### 无坐标模式（简单回退）

当没有 xGrid/yGrid 时，按行列索引直接映射：

```text
像素(j, srcRows-1-i)  ←→  数据点 zGrid[i][j]
```

注意行号取反（`py = srcRows - 1 - i`），因为图像坐标 Y 轴向下，而数据通常 Y 轴向上。

#### 有坐标模式（主要路径）

**第一步：正向映射（物理坐标 → 像素坐标）**

```typescript
pxi = round(((x - xMin) / xRange) × (OUT_W - 1))
pyi = round(((yMax - y)  / yRange) × (OUT_H - 1))
```

每个数据点投影到输出像素网格，落到同一像素的多个点取 z 均值，结果存入按像素行分组的 `rowBuckets`。

**第二步：水平限距插值**

对每个像素行，将 rowBucket 中的点按 px 排序后，对相邻两点之间的空白像素做线性插值。  
关键阈值：

```text
maxFillH = ceil(OUT_W / srcCols × 2.5) + 2
```

相邻点间距超过 `maxFillH` 时，认为是真实数据空洞，**不插值**，保留透明。

**第三步：垂直限距插值**

逐列扫描，对上下两个有值行之间的空白像素做线性插值，同样受 `maxFillV` 约束。

**第四步：写入 ImageData**

```typescript
t = (z - zMin) / (zMax - zMin)
[r, g, b] = interpolateColor(t, stops)
buf[idx * 4 + 0] = r
buf[idx * 4 + 1] = g
buf[idx * 4 + 2] = b
buf[idx * 4 + 3] = 255   // 不透明；空洞像素 alpha = 0
```

最终调用 `canvas.toDataURL('image/png')` 导出 PNG Base64 字符串。

---

### 3.3 正方形 Grid 修正

伪彩色图通常要求绘图区为**正方形**（物理坐标等比展示）。ECharts 默认 grid 是矩形，因此在 `setOption` 前动态计算：

```text
availW = totalW - LEFT_LABEL(50) - RIGHT_VM(90)
availH = totalH - TOP_PAD(10)  - BOTTOM_LABEL(40)
squareSide = min(availW, availH)

grid.left   = 50  + floor((availW - squareSide) / 2)
grid.right  = 90  + ceil ((availW - squareSide) / 2)
grid.top    = 10  + floor((availH - squareSide) / 2)
grid.bottom = 40  + ceil ((availH - squareSide) / 2)
```

多余空间均分到对应边，使绘图区严格居中且为正方形。

---

### 3.4 精确贴图 —— `convertToPixel` + `graphic.image`

ECharts 的 `finished` 事件在所有动画和布局完成后触发，此时坐标系已确定。利用：

```typescript
const topLeft     = myChart.convertToPixel({ gridIndex: 0 }, [axisXMin, axisYMax])
const bottomRight = myChart.convertToPixel({ gridIndex: 0 }, [axisXMax, axisYMin])
```

可以精确获得数据范围对应的像素区域（`gridW × gridH`），用于：

1. 以该尺寸调用 `renderToDataURL()`，生成与 grid 等大的 PNG
2. 通过 `setOption({ graphic })` 将图片以 `x/y/width/height` 定位到 grid 区域
3. 图片 `z: -1`（`silent: true`），位于坐标轴和散点之下，不响应鼠标事件

---

## 四、数据流全链路

```text
API 响应 (res.data = pcolorData[])
    │
    ▼ ChartComponentColor.vue: calcData()
res.data 包装为标准 chart.data 结构
    │
    ▼ PColor.drawChart()
解析 pcolormesh / scatter
计算 xMin/xMax/yMin/yMax/zMin/zMax
生成 ECharts option（含 __pcolorRawData 附加字段）
    │
    ▼ echarts.init() + setOption()       [canvas renderer]
ECharts 完成布局
    │
    ▼ myChart.one('finished', callback)
convertToPixel() 获取 grid 像素坐标
renderToDataURL() 生成 PNG（W=gridW, H=gridH）
    │
    ▼ setOption({ graphic[id='pcolor-bg'] })
PNG 图片精确覆盖 grid 区域
散点层叠加在图片之上
```

---

## 五、关键设计决策

| 问题 | 决策 | 原因 |
|------|------|------|
| 为何用 Canvas 渲染器？ | `renderer: 'canvas'` 强制指定 | SVG 模式下 `graphic.image` 无法正常显示 |
| 为何等 `finished` 再渲染？ | 监听 `myChart.one('finished', ...)` | `setOption` 后布局未立即完成，`convertToPixel` 须在布局稳定后调用 |
| 空洞如何处理？ | 双向插值 + 限距阈值 | 超过 2.5 倍平均间距的空白认为是真实缺失数据，保留透明而非强制插值 |
| 如何支持自定义色谱？ | 用户配置的 hex 颜色数组 → 等间距 stops | 与 Jet 色谱采用相同插值逻辑，无缝替换 |
| 为何要正方形 grid？ | 物理坐标等比呈现 | 若 grid 为矩形，X/Y 轴比例失真，地理/物理含义错误 |
| 输出尺寸上限 2048 | `Math.min(..., 2048)` | 防止超大数据集导致内存占用过高或 canvas 创建失败 |

---

## 六、代码结构速览

```text
PColor（extends EChartView）
├── interpolateColor(t, stops)      色谱线性插值
├── parseCustomStops(fieldColorData) 自定义色谱解析
├── computeRange(grid)              数值范围计算
├── renderToDataURL(...)            网格 → PNG DataURL（核心渲染）
└── drawChart(drawOptions)          生成 ECharts option

ChartComponentColor.vue
├── calcData()                      请求数据 + 数据包装
├── renderChart()                   选择渲染器
└── renderEcharts()
    ├── 正方形 grid 修正
    ├── myChart.setOption(option)
    └── finished 事件
        ├── convertToPixel()
        ├── renderToDataURL()
        └── setOption(graphic)
```

---

## 七、总结

伪彩色图的实现绕开了 ECharts 原生不支持非规则网格填色的限制，核心思路是：

1. **用 ECharts 只负责坐标系、轴标签、visualMap 图例、散点**，这些是它擅长的
2. **自行用 Canvas 2D API 渲染色彩填充层**，精确控制每个像素的颜色
3. **通过 `convertToPixel` 桥接两个坐标系**，确保色彩层与坐标轴严格对齐
4. **双向限距插值**填补非规则网格在像素域的空洞，兼顾视觉连续性与数据真实性

这种"ECharts 框架 + 自定义 Canvas 图层"的混合方案，为在 BI 平台中扩展复杂科学可视化提供了一种可复用的思路。
