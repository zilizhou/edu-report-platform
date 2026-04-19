---
name: exam-analysis
description: >
  根据试卷文件和成绩单，自动生成一份结构化的试卷分析报告（HTML），
  涵盖总分分布、各大题得分率、题目难度与区分度、知识点掌握情况等维度，
  并内嵌柱状图、折线图、饼图、箱形图、散点图等多种可视化图表，以及一份文字试卷总结。
  支持三种成绩单格式：仅含总分、含大题分+总分、含小题分+总分，
  根据数据粒度自动选择分析深度与图表种类。
  当用户上传试卷和成绩单，并提及"试卷分析"、"考试分析"、"成绩分析"、
  "出题质量"、"题目难度"、"学生整体表现"、"班级成绩"等需求时，务必触发本技能。
  即使用户只说"帮我看看这次考试考得怎么样"或"分析一下成绩单"，
  只要同时涉及试卷结构与班级成绩数据，也应使用本技能。
---

# 试卷分析技能（Exam Analysis Skill）

## 一、技能目标

读取一份试卷（`.docx`）和对应的成绩单（`.csv`），输出一份完整的 HTML 试卷分析报告。
报告面向任课教师，帮助其了解：
- 班级整体成绩情况（分布、均值、优秀率/不及格率等）
- 各大题的整体得分率，识别"拉分题"
- 各小题/知识点的难度系数与区分度（如数据充足）
- 可视化图表：柱状图、折线图、饼图、箱形图、散点图
- 文字版试卷总结与教学建议

---

## 二、输入文件说明

| 文件类型 | 内容 | 分析能力 |
|---|---|---|
| 成绩单1：仅总分 | `学号, 姓名, 总分` | 总分分布分析 + 饼图/柱状图/折线图 |
| 成绩单2：大题分+总分 | `学号, 姓名, 大题1(X分), ..., 总分` | + 各大题得分率分组柱状图/折线图/箱形图 |
| 成绩单3：小题分+总分 | `学号, 姓名, 题1, 题2, ..., 总分` | + 各小题 P/D 散点图、P值/D值柱状图 |

**试卷文件**（`.docx`）用于提取：各大题名称与满分、各小题题号与分值。

---

## 三、执行流程

### 步骤 1：读取试卷

```bash
extract-text /path/to/exam.docx
```

提取大题结构，整理为 Python 配置：

```python
SECTIONS = [
    # (大题名, 满分, 题目数, 每题满分, 列名前缀)
    ("选择题", 20, 20, 1,  "选择"),
    ("判断题", 10, 10, 1,  "判断"),
    ("填空题", 10,  5, 2,  "填空"),   # 每题含2空，各1分
    ("简答题", 30,  5, 6,  "简答"),
    ("综合题", 30,  3, 10, "综合"),
]
```

### 步骤 2：读取成绩单，判断数据粒度

```python
import pandas as pd
df = pd.read_csv("/path/to/scores.csv")
cols = df.columns.tolist()
# 只有"总分" → 粒度=总分
# 含"大题(X分)"列 → 粒度=大题
# 含"选择1","判断1"等列 → 粒度=小题
```

### 步骤 3：计算所有统计指标

#### 3A. 总分统计（所有粒度）

```python
import numpy as np
scores = df["总分"]
n = len(scores)

# 基础统计
avg    = round(float(scores.mean()), 2)
median = float(scores.median())
std    = round(float(scores.std(ddof=1)), 2)
s_max, s_min = int(scores.max()), int(scores.min())

# 分数段分布
bins   = [0, 60, 70, 80, 90, 101]
labels = ["不及格(<60)", "及格(60~69)", "中等(70~79)", "良好(80~89)", "优秀(≥90)"]
df["等级"] = pd.cut(scores, bins=bins, labels=labels, right=False)
grade_dist = df["等级"].value_counts().reindex(labels).fillna(0).astype(int)
grade_pct  = (grade_dist / n * 100).round(1)

# 排名序列（用于折线图）
sorted_scores = sorted([int(x) for x in scores.tolist()])
```

#### 3B. 大题得分率（粒度 ≥ 大题）

```python
section_summary = []
for (sec_name, full, cnt, item_full, prefix) in SECTIONS:
    cols = [f"{prefix}{i+1}" for i in range(cnt)]
    available = [c for c in cols if c in df.columns]
    sec_scores = df[available].sum(axis=1)
    avg_s  = float(sec_scores.mean())
    rate   = avg_s / full * 100
    section_summary.append({
        "name": sec_name, "full": full,
        "avg": round(avg_s, 2), "rate": round(rate, 1),
        "scores": [int(x) for x in sec_scores.tolist()]  # 用于箱形图
    })
```

#### 3C. 小题难度系数 P 与区分度 D（粒度 = 小题）

```python
top_n  = max(1, int(n * 0.27))
df_s   = df.sort_values("总分")
high_g = df_s.tail(top_n)   # 高分组（前27%）
low_g  = df_s.head(top_n)   # 低分组（后27%）

item_rows = []
for (sec_name, full, cnt, item_full, prefix) in SECTIONS:
    for i in range(cnt):
        col = f"{prefix}{i+1}"
        if col not in df.columns: continue
        P = float(df[col].mean() / item_full)
        D = float((high_g[col].mean() - low_g[col].mean()) / item_full)
        item_rows.append({
            "sec": sec_name, "col": col, "item_full": item_full,
            "avg": round(float(df[col].mean()), 3),
            "P": round(P, 3), "D": round(D, 3),
            "warn": int(P < 0.30 or P > 0.90 or D < 0.20)
        })
```

**难度/区分度评级标准：**

| P 值 | 难度评级 | D 值 | 区分度评级 |
|---|---|---|---|
| P > 0.85 | 过易 | D ≥ 0.40 | 优秀 |
| 0.70 ~ 0.85 | 较易 | 0.30 ~ 0.39 | 良好 |
| 0.40 ~ 0.70 | 适中（理想） | 0.20 ~ 0.29 | 尚可 |
| 0.25 ~ 0.40 | 较难 | D < 0.20 | 差，建议修改 |
| P < 0.25 | 过难 | | |

---

### 步骤 4：生成 HTML 报告（含图表）

报告统一使用 **Chart.js 4.x**（从 CDN 加载），将统计数据序列化为 JSON 嵌入 HTML，
在页面 `<script>` 中绘制所有图表，无需服务端支持，单文件可直接用浏览器打开。

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js"></script>
```

---

## 四、图表规范（Chart.js 实现）

### 4.1 图表总览（按粒度分级）

| 图表 | 类型 | 所需粒度 | 说明 |
|---|---|---|---|
| 分数段人数分布 | **柱状图** | 总分 | 各等级人数，不同颜色区分 |
| 成绩等级占比 | **饼图** | 总分 | 各等级百分比 |
| 学生成绩排名 | **折线图** | 总分 | 升序排列，红点=不及格，绿点=优良 |
| 各大题平均得分 vs 满分 | **分组柱状图** | 大题 | 满分+平均得分并排对比 |
| 各大题得分率趋势 | **折线图** | 大题 | 颜色编码：绿=良好，黄=一般，红=薄弱 |
| 各大题得分分布 | **箱形图** | 大题 | 自定义 Plugin 实现，显示 Q1/中位/Q3/须线 |
| 各小题 P vs D | **散点图** | 小题 | 按大题分色，叠加"理想区域"矩形 |
| 各大题平均难度系数 | **柱状图** | 小题 | 颜色编码：绿=适中，黄=偏易，红=过易 |
| 各大题平均区分度 | **柱状图** | 小题 | 颜色编码：绿=良好，黄=尚可，红=差 |

---

### 4.2 柱状图（分数段人数）

```javascript
new Chart(ctx, {
  type: "bar",
  data: {
    labels: ["不及格(<60)", "及格(60~69)", "中等(70~79)", "良好(80~89)", "优秀(≥90)"],
    datasets: [{
      label: "人数",
      data: gradeCounts,                          // [3, 6, 8, 3, 0]
      backgroundColor: ["#ef4444","#f59e0b","#3b82f6","#22c55e","#8b5cf6"],
      borderRadius: 7,
      borderSkipped: false
    }]
  },
  options: {
    plugins: { legend: { display: false },
      tooltip: { callbacks: { label: c => `${c.raw}人（${gradePcts[c.dataIndex]}%）` }}},
    scales: { y: { beginAtZero: true, ticks: { stepSize: 1 },
                   title: { display: true, text: "人数" }}}
  }
});
```

---

### 4.3 饼图（等级占比）

```javascript
new Chart(ctx, {
  type: "pie",
  data: {
    labels: gradeLabels,
    datasets: [{
      data: gradePcts,                            // 百分比数组
      backgroundColor: gradeColors,
      borderWidth: 2, borderColor: "#fff"
    }]
  },
  options: {
    plugins: {
      tooltip: { callbacks: { label: c => `${c.label}: ${c.raw}%` }},
      legend: { position: "bottom" }
    }
  }
});
```

---

### 4.4 折线图（成绩排名曲线）

```javascript
new Chart(ctx, {
  type: "line",
  data: {
    labels: sortedScores.map((_, i) => `第${i+1}名`),
    datasets: [{
      label: "总分", data: sortedScores,
      borderColor: "#2563a8", backgroundColor: "rgba(37,99,168,.10)",
      fill: true, tension: 0.35,
      // 红色=不及格，绿色=优良，蓝色=其余
      pointBackgroundColor: sortedScores.map(s =>
        s < 60 ? "#ef4444" : s >= 80 ? "#22c55e" : "#3b82f6"),
      pointRadius: 5, pointHoverRadius: 8
    }]
  },
  options: {
    plugins: { legend: { display: false }},
    scales: {
      y: { min: 40, max: 100, title: { display: true, text: "分数" }},
      x: { title: { display: true, text: "排名（升序）" }}
    }
  }
});
```

---

### 4.5 分组柱状图（大题平均得分 vs 满分）

```javascript
new Chart(ctx, {
  type: "bar",
  data: {
    labels: sectionData.map(s => s.name),
    datasets: [
      { label: "满分",   data: sectionData.map(s => s.full),
        backgroundColor: "rgba(203,213,225,.55)", borderRadius: 5 },
      { label: "平均得分", data: sectionData.map(s => s.avg),
        backgroundColor: "#2563a8", borderRadius: 5 }
    ]
  },
  options: {
    plugins: { legend: { position: "bottom" }},
    scales: { y: { beginAtZero: true, title: { display: true, text: "分数" }}}
  }
});
```

---

### 4.6 折线图（各大题得分率）

```javascript
new Chart(ctx, {
  type: "line",
  data: {
    labels: sectionData.map(s => s.name),
    datasets: [{
      label: "得分率(%)", data: sectionData.map(s => s.rate),
      borderColor: "#16a34a", backgroundColor: "rgba(22,163,74,.10)",
      fill: true, tension: 0.3, pointRadius: 7,
      pointBackgroundColor: sectionData.map(s =>
        s.rate >= 80 ? "#16a34a" : s.rate >= 65 ? "#f59e0b" : "#ef4444")
    }]
  },
  options: {
    plugins: { legend: { display: false }},
    scales: {
      y: { min: 0, max: 100, title: { display: true, text: "得分率 (%)" }},
      x: { title: { display: true, text: "题型" }}
    }
  }
});
```

---

### 4.7 箱形图（Box Plot — 自定义 Plugin）

Chart.js 原生不支持箱形图，通过 `afterDatasetsDraw` 钩子手绘：

```javascript
// 箱形统计函数
function boxStats(arr) {
  const s = [...arr].sort((a, b) => a - b), n = s.length;
  const q = p => {
    const i = p * (n - 1), lo = Math.floor(i), hi = Math.ceil(i);
    return s[lo] + (s[hi] - s[lo]) * (i - lo);
  };
  return { min: s[0], q1: q(.25), med: q(.5), q3: q(.75), max: s[n-1] };
}

const boxPlugin = {
  id: "customBox",
  afterDatasetsDraw(chart) {
    const { ctx, scales: { x, y } } = chart;
    boxes.forEach((b, i) => {
      const cx  = x.getPixelForValue(i);
      const bw  = Math.min(x.width / labels.length * 0.38, 38);
      const [pMin, pQ1, pMed, pQ3, pMax] =
        [b.min, b.q1, b.med, b.q3, b.max].map(v => y.getPixelForValue(v));

      ctx.save();
      // IQR 矩形（Q1~Q3）
      ctx.fillStyle = color + "33"; ctx.strokeStyle = color; ctx.lineWidth = 2;
      ctx.beginPath(); ctx.roundRect(cx - bw, pQ3, bw * 2, pQ1 - pQ3, 4);
      ctx.fill(); ctx.stroke();
      // 中位线（红色）
      ctx.beginPath(); ctx.moveTo(cx - bw, pMed); ctx.lineTo(cx + bw, pMed);
      ctx.strokeStyle = "#dc2626"; ctx.lineWidth = 2.5; ctx.stroke();
      // 须线（虚线）及端帽
      ctx.strokeStyle = "#64748b"; ctx.lineWidth = 1.5; ctx.setLineDash([4, 3]);
      [[pQ3, pMax], [pQ1, pMin]].forEach(([from, to]) => {
        ctx.beginPath(); ctx.moveTo(cx, from); ctx.lineTo(cx, to); ctx.stroke();
        ctx.setLineDash([]);
        ctx.beginPath(); ctx.moveTo(cx - bw * .42, to); ctx.lineTo(cx + bw * .42, to); ctx.stroke();
        ctx.setLineDash([4, 3]);
      });
      ctx.restore();
    });
  }
};

new Chart(ctx, {
  type: "scatter",          // 底层用 scatter，Plugin 在上面画箱
  plugins: [boxPlugin],
  data: { datasets: [{ data: [] }] },
  options: {
    plugins: { legend: { display: false }, tooltip: { enabled: false }},
    scales: {
      x: { type: "linear", min: -0.6, max: labels.length - 0.4,
           ticks: { stepSize: 1, callback: (_, i) => labels[i] || "" },
           title: { display: true, text: "题型" }},
      y: { title: { display: true, text: "得分" }}
    }
  }
});
```

---

### 4.8 散点图（P 值 vs D 值）+ 理想区域遮罩

```javascript
// 理想区域 Plugin（P 0.40~0.85，D ≥ 0.30）
const idealPlugin = {
  id: "ideal",
  beforeDatasetsDraw(chart) {
    const { ctx, scales: { x, y } } = chart;
    ctx.save();
    const x1 = x.getPixelForValue(0.40), x2 = x.getPixelForValue(0.85);
    const y1 = y.getPixelForValue(0.30), y2 = y.getPixelForValue(yAxisMax);
    ctx.fillStyle   = "rgba(22,163,74,.07)";
    ctx.strokeStyle = "rgba(22,163,74,.4)";
    ctx.lineWidth = 1.5; ctx.setLineDash([5, 4]);
    ctx.fillRect(x1, y2, x2 - x1, y1 - y2);
    ctx.strokeRect(x1, y2, x2 - x1, y1 - y2);
    ctx.setLineDash([]);
    ctx.fillStyle = "rgba(22,163,74,.7)"; ctx.font = "11px sans-serif";
    ctx.fillText("理想区域", x1 + 4, y2 + 14);
    ctx.restore();
  }
};

// 按大题分色的散点数据集
const datasets = Object.keys(secColors).map(sec => ({
  label: sec,
  data: itemData.filter(it => it.sec === sec).map(it => ({ x: it.P, y: it.D, col: it.col })),
  backgroundColor: secColors[sec] + "bb",
  pointRadius: 6, pointHoverRadius: 9
}));

new Chart(ctx, {
  type: "scatter", plugins: [idealPlugin],
  data: { datasets },
  options: {
    plugins: {
      legend: { position: "bottom" },
      tooltip: { callbacks: { label: c => `${c.raw.col}  P=${c.raw.x}  D=${c.raw.y}` }}
    },
    scales: {
      x: { min: -0.05, max: 1.05,
           title: { display: true, text: "难度系数 P（越大越容易）" },
           ticks: { callback: v => v.toFixed(1) }},
      y: { title: { display: true, text: "区分度 D（越大越好）" },
           ticks: { callback: v => v.toFixed(1) }}
    }
  }
});
```

---

## 五、报告完整结构

```
1. 页眉（科目、年级、专业、参考人数、满分）
2. 一、总分统计摘要卡片（平均分/中位数/标准差/最高最低/及格率/不及格率/优良率/优秀率）
3. 二、成绩分布分析
     柱状图：分数段人数
     饼图：等级占比
     折线图：成绩排名曲线
4. 三、各大题得分率分析（粒度 ≥ 大题）
     分组柱状图：平均得分 vs 满分
     折线图：各大题得分率
     箱形图：各大题得分分布
     汇总表（均值/最高/最低/中位/得分率/评级）
5. 四、小题难度与区分度分析（粒度 = 小题）
     散点图：P vs D（含理想区域）
     柱状图：各大题平均 P 值
     柱状图：各大题平均 D 值
     明细表（全部小题，⚠️标注需关注项）
6. 五、试卷总结（文字）
     整体成绩概况
     等级分布特点
     各大题表现分析
     题目质量问题（区分度/难度异常题目）
     教学改进建议
```

---

## 六、试卷总结撰写规则

总结分为五个固定段落，依据统计结果自动填入关键数据：

| 段落 | 核心内容 | 触发关注标记的条件 |
|---|---|---|
| 整体成绩概况 | 人数、均值、中位数、标准差、分数区间 | std > 12 → "分化明显" |
| 等级分布特点 | 各等级人数与占比，优秀/不及格占比 | fail_rate > 20% → 红色高亮 |
| 各大题表现分析 | 逐大题点评，标注最低得分率题型 | rate < 65% → "主要失分点" |
| 题目质量问题 | 区分度差/难度异常的题目数量与分布 | warn_count > 10 → 重点关注 |
| 教学改进建议 | 对应薄弱点给出 3~5 条可操作建议 | 始终输出 |

---

## 七、Python 完整脚本骨架

```python
import pandas as pd
import numpy as np
import json, re

# ── 配置（根据试卷实际情况修改）──────────────────────────────
EXAM_TITLE  = "XXX期末考试"
EXAM_GRADE  = "XX级"
EXAM_MAJOR  = "XXX专业"
SCORES_PATH = "/mnt/user-data/uploads/成绩单.csv"
OUTPUT_PATH = "/mnt/user-data/outputs/exam_analysis_report.html"

SECTIONS = [
    ("选择题", 20, 20, 1,  "选择"),
    ("判断题", 10, 10, 1,  "判断"),
    ("填空题", 10,  5, 2,  "填空"),
    ("简答题", 30,  5, 6,  "简答"),
    ("综合题", 30,  3, 10, "综合"),
]

# ── 1. 读取数据 ───────────────────────────────────────────────
df     = pd.read_csv(SCORES_PATH)
scores = df["总分"]
n      = len(df)

# ── 2. 总分统计 ───────────────────────────────────────────────
labels_clean = ["不及格(<60)","及格(60~69)","中等(70~79)","良好(80~89)","优秀(≥90)"]
df["等级"]   = pd.cut(scores, bins=[0,60,70,80,90,101], labels=labels_clean, right=False)
grade_dist   = df["等级"].value_counts().reindex(labels_clean).fillna(0).astype(int)
grade_pct    = (grade_dist / n * 100).round(1)

# ── 3. 大题得分率 ─────────────────────────────────────────────
section_summary = []
for (sec_name, full, cnt, item_full, prefix) in SECTIONS:
    cols      = [f"{prefix}{i+1}" for i in range(cnt)]
    available = [c for c in cols if c in df.columns]
    if not available: continue
    sec_sc = df[available].sum(axis=1)
    section_summary.append({
        "name": sec_name, "full": full,
        "avg":  round(float(sec_sc.mean()), 2),
        "rate": round(float(sec_sc.mean()) / full * 100, 1),
        "scores": [int(x) for x in sec_sc.tolist()]
    })

# ── 4. 小题 P / D ─────────────────────────────────────────────
top_n = max(1, int(n * 0.27))
df_s  = df.sort_values("总分")
high_g, low_g = df_s.tail(top_n), df_s.head(top_n)

item_rows = []
for (sec_name, full, cnt, item_full, prefix) in SECTIONS:
    for i in range(cnt):
        col = f"{prefix}{i+1}"
        if col not in df.columns: continue
        P    = float(df[col].mean() / item_full)
        D    = float((high_g[col].mean() - low_g[col].mean()) / item_full)
        item_rows.append({
            "sec": sec_name, "col": col, "item_full": item_full,
            "avg": round(float(df[col].mean()), 3),
            "P": round(P, 3), "D": round(D, 3),
            "warn": int(P < 0.30 or P > 0.90 or D < 0.20)
        })

# ── 5. 序列化数据，嵌入 HTML ──────────────────────────────────
data_json = json.dumps({
    "gradeLabels":  labels_clean,
    "gradeCounts":  [int(x) for x in grade_dist.tolist()],
    "gradePcts":    [float(x) for x in grade_pct.tolist()],
    "sortedScores": sorted([int(x) for x in scores.tolist()]),
    "sections":     section_summary,
    "items":        item_rows,
}, ensure_ascii=False)

# ── 6. 将 data_json 插入 HTML 模板，写入文件 ──────────────────
# （在 <script> 标签中：const DATA = <data_json>;）
# 然后用 DATA 驱动所有 Chart.js 图表的绘制
with open(OUTPUT_PATH, "w", encoding="utf-8") as f:
    f.write(html_content)   # html_content 参见报告模板
```

---

## 八、输出文件

| 文件 | 说明 |
|---|---|
| `exam_analysis_report.html` | 完整单文件报告，可直接用浏览器打开，无需网络（Chart.js 从 CDN 加载除外） |

使用 `present_files` 工具将报告展示给用户。

---

## 九、注意事项

1. **样本量较少（n < 30）**：区分度 D 统计意义有限，报告中自动注明"样本量较小，结论供参考"。
2. **列名规范**：大题列名含满分（如`选择题(20分)`）；小题列名为`{前缀}{编号}`（如`选择1`）。
3. **缺考/异常值**：总分为 0 或空值时提示用户确认是否为缺考，可选择排除后重新统计。
4. **Chart.js 箱形图**：原生不支持，须使用 `afterDatasetsDraw` Plugin 手绘，底层 type 用 `scatter`。
5. **报告语言**：默认中文；若用户要求英文，切换所有标签、提示文字及总结段落。
6. **图表颜色编码一致性**：
   - 不及格 → `#ef4444`（红），及格 → `#f59e0b`（黄），中等 → `#3b82f6`（蓝），良好 → `#22c55e`（绿），优秀 → `#8b5cf6`（紫）
   - 得分率良好（≥80%）→ 绿，一般（65~79%）→ 黄，薄弱（<65%）→ 红
   - P值适中（0.40~0.70）→ 绿，偏易（0.70~0.85）→ 黄，过易（>0.85）→ 红
   - D值良好（≥0.30）→ 绿，尚可（0.20~0.29）→ 黄，差（<0.20）→ 红
