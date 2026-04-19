---
name: student-knowledge-graph
description: >
  根据课程知识图谱三元组表（knowledge_graph_triples）和学生知识点掌握情况表（student_summary_profile），
  为指定学生生成交互式知识画像网络图。画像以网状知识图谱形式展现，节点为知识点，节点颜色和大小
  反映掌握程度（优秀/良好/一般/薄弱/未掌握），边反映知识点间的关系（包含、前置、对比等）。
  输出为可在浏览器中交互操作的 HTML 文件，支持节点拖拽、缩放、悬停提示和图例。
  当用户提到"知识画像"、"知识网络"、"知识图谱可视化"、"学生掌握情况图"、"知识点关系图"、
  "学情可视化"时，务必触发本技能。即使用户只说"帮我看看这个学生哪里学得好哪里学得差"或
  "把学生的知识掌握情况画出来"，只要涉及以图形方式展示学生知识点掌握情况，也应使用本技能。
compatibility: "claude.ai, Claude Desktop, Cowork — 需要 Python 3.8+、pandas、json 标准库"
---

# 学生知识画像生成技能

## 功能概述

本技能将两类数据融合，生成交互式知识网络画像：

1. **课程知识图谱三元组表**（knowledge_graph_triples）：定义知识点之间的结构关系
2. **学生知识掌握情况表**（student_summary_profile）：记录每个学生对各知识点的掌握等级

输出：一个独立 HTML 文件，内嵌 D3.js 力导向图，节点颜色代表掌握程度，支持交互操作。

---

## 输入数据格式

### 知识图谱三元组表（knowledge_graph_triples.csv）

| 字段 | 说明 |
|------|------|
| subject | 关系主体知识点名称 |
| relation | 关系类型：CONTAINS / IS_A / PREREQUISITE_OF / CONTRASTS_WITH / USED_FOR |
| object | 关系客体知识点名称 |
| subject_type | 主体类型（概念/章节/协议/方法等） |
| object_type | 客体类型 |
| source | 来源 |
| confidence | 置信度（high/medium/low） |

### 学生掌握情况表（student_summary_profile.csv）

| 字段 | 说明 |
|------|------|
| student_id | 学生ID |
| student_name | 学生姓名 |
| exam_id | 考试ID |
| total_score_rate | 总分率（0~1） |
| excellent_kps | 优秀掌握的知识点，分号分隔 |
| good_kps | 良好掌握的知识点，分号分隔 |
| fair_kps | 一般掌握的知识点，分号分隔 |
| weak_kps | 薄弱知识点，分号分隔 |
| poor_kps | 未掌握知识点，分号分隔 |
| top3_strength | 最强的3个知识点 |
| top3_weakness | 最弱的3个知识点 |
| top3_high_risk | 高风险知识点（已掌握但易错） |
| overall_level | 整体水平（excellent/good/fair/poor） |

---

## 执行步骤

### Step 1：读取并解析数据

```python
import pandas as pd
import json

# 读取数据
triples_df = pd.read_csv("knowledge_graph_triples.csv")
profile_df = pd.read_csv("student_summary_profile.csv")
```

### Step 2：确定目标学生

- 如果用户指定了学生姓名或 ID，按指定筛选
- 如果未指定，默认取第一个学生，并提示用户可切换
- 如果要生成全班画像，循环生成每个学生的 HTML 文件

### Step 3：构建掌握程度映射

```python
def build_mastery_map(student_row):
    """将学生掌握情况解析为 {知识点: 掌握等级} 字典"""
    mastery = {}
    level_fields = {
        'excellent': 'excellent_kps',
        'good':      'good_kps',
        'fair':      'fair_kps',
        'weak':      'weak_kps',
        'poor':      'poor_kps',
    }
    for level, field in level_fields.items():
        val = student_row.get(field, '')
        if pd.notna(val) and str(val).strip():
            for kp in str(val).split(';'):
                kp = kp.strip()
                if kp:
                    mastery[kp] = level
    return mastery
```

### Step 4：筛选相关节点和边

只保留该学生画像中出现的知识点以及它们之间的关系边：

```python
def build_graph(triples_df, mastery_map):
    """从三元组中筛选与该学生相关的节点和边"""
    known_kps = set(mastery_map.keys())

    # 筛选两端均为已知知识点的边
    edges = []
    related_kps = set()
    for _, row in triples_df.iterrows():
        s, o = row['subject'], row['object']
        # 只要其中一端是学生已知知识点就纳入
        if s in known_kps or o in known_kps:
            edges.append({
                'source': s,
                'target': o,
                'relation': row['relation'],
                'confidence': row['confidence']
            })
            related_kps.add(s)
            related_kps.add(o)

    # 构建节点列表
    nodes = []
    for kp in related_kps:
        nodes.append({
            'id': kp,
            'mastery': mastery_map.get(kp, 'unknown'),
            'type': _get_type(triples_df, kp)
        })
    return nodes, edges

def _get_type(triples_df, kp):
    row = triples_df[triples_df['subject'] == kp]
    if not row.empty:
        return row.iloc[0]['subject_type']
    row = triples_df[triples_df['object'] == kp]
    if not row.empty:
        return row.iloc[0]['object_type']
    return '概念'
```

### Step 5：生成 HTML 可视化

将节点和边数据注入 HTML 模板，使用 D3.js force simulation 渲染。

**颜色方案（掌握程度 → 节点颜色）**：

| 掌握等级 | 颜色 | 说明 |
|---------|------|------|
| excellent | `#22c55e`（绿色）| 优秀掌握 |
| good | `#86efac`（浅绿）| 良好掌握 |
| fair | `#fbbf24`（黄色）| 一般，需加强 |
| weak | `#f97316`（橙色）| 薄弱，需重点复习 |
| poor | `#ef4444`（红色）| 未掌握 |
| unknown | `#cbd5e1`（灰色）| 图谱中有但未评估 |

**边样式（关系类型）**：

| 关系 | 线型 | 说明 |
|------|------|------|
| CONTAINS | 实线，灰色 | 包含关系 |
| PREREQUISITE_OF | 实线，蓝色带箭头 | 前置依赖 |
| IS_A | 虚线，紫色 | 类属关系 |
| CONTRASTS_WITH | 虚线，橙色 | 对比关系 |
| USED_FOR | 点线，青色 | 应用关系 |

### Step 6：写入 HTML 文件

```python
html_content = generate_html(student_name, nodes, edges, stats)
output_path = f"知识画像_{student_name}_{exam_id}.html"
with open(output_path, 'w', encoding='utf-8') as f:
    f.write(html_content)
```

---

## HTML 模板结构

生成的 HTML 需包含以下模块：

### 顶部信息栏
- 学生姓名、考试名称、总分率
- 整体水平标签（excellent/good/fair/poor）

### 图例面板
- 5种掌握等级的颜色说明
- 5种边关系类型说明

### 统计摘要
- 各等级知识点数量统计（饼状文字概要）
- top3 优势/薄弱/高风险知识点

### 主画布（D3.js 力导向图）
- 节点：圆形，颜色 = 掌握等级，大小 = 节点度数（边越多越大）
- 边：箭头线，颜色/虚实 = 关系类型
- 悬停节点：显示知识点名称、掌握等级、关系数量
- 支持拖拽节点、鼠标滚轮缩放、平移

### 控制面板
- 按掌握等级过滤（复选框）
- 按关系类型过滤（复选框）
- 重置布局按钮

---

## 完整 Python 代码示例

以下为生成完整 HTML 画像的核心函数：

```python
def generate_knowledge_portrait(
    triples_path: str,
    profile_path: str,
    student_identifier: str = None,   # 姓名或ID，None则取第一个
    output_dir: str = "."
) -> str:
    """
    主函数：生成学生知识画像 HTML 文件
    返回输出文件路径
    """
    import pandas as pd, json, os

    triples_df = pd.read_csv(triples_path)
    profile_df = pd.read_csv(profile_path)

    # 选定学生
    if student_identifier is None:
        student_row = profile_df.iloc[0]
    elif student_identifier in profile_df['student_name'].values:
        student_row = profile_df[profile_df['student_name'] == student_identifier].iloc[0]
    else:
        student_row = profile_df[profile_df['student_id'] == student_identifier].iloc[0]

    mastery_map = build_mastery_map(student_row)
    nodes, edges = build_graph(triples_df, mastery_map)

    # 统计信息
    stats = {level: sum(1 for n in nodes if n['mastery'] == level)
             for level in ['excellent','good','fair','weak','poor','unknown']}
    stats['total'] = len(nodes)

    html = _render_html_template(student_row, nodes, edges, stats)
    fname = f"知识画像_{student_row['student_name']}_{student_row['exam_id']}.html"
    fpath = os.path.join(output_dir, fname)
    with open(fpath, 'w', encoding='utf-8') as f:
        f.write(html)
    print(f"✅ 已生成: {fpath}（节点 {len(nodes)} 个，边 {len(edges)} 条）")
    return fpath
```

`_render_html_template` 函数需输出完整独立 HTML，将 nodes/edges 序列化为 JSON 内嵌到 `<script>` 中，并使用 CDN 引入 D3.js v7：
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/7.8.5/d3.min.js"></script>
```

---

## 输出质量要求

生成的 HTML 必须满足：

1. **完全独立**：无需网络连接也能打开（D3 可用 CDN，数据全部内嵌）
2. **中文正常显示**：HTML meta charset="utf-8"，文件以 UTF-8 保存
3. **移动端可用**：viewport meta 标签，触摸事件支持
4. **节点不重叠**：force simulation 中 collide 半径 ≥ 节点半径 + 5px
5. **画布自适应**：宽高绑定到窗口大小
6. **图例可见**：固定定位图例不随画布滚动

---

## 常见问题处理

| 问题 | 处理方式 |
|------|---------|
| 知识点过多（>200节点）导致卡顿 | 默认只显示与学生掌握知识点直接相连的一跳节点；提供"展开全图"按钮 |
| 某掌握等级字段为空/NaN | 跳过，不报错 |
| 学生名称未找到 | 列出所有可用学生姓名，提示用户选择 |
| 三元组中知识点与学生表知识点名称不完全匹配 | 做去空格处理；可选模糊匹配（编辑距离 ≤ 1） |
| 边数远多于节点数导致图混乱 | 默认只显示 confidence=high 的边；提供"显示全部关系"开关 |

---

## 触发示例

以下用户请求都应触发本技能：

- "帮我生成张伟的知识画像"
- "把这个学生的知识掌握情况画成图谱"
- "用知识图谱展示一下学生的薄弱点"
- "生成全班的知识网络可视化"
- "我想看看学生的知识点掌握关系图"
- "把 student_summary_profile 和 knowledge_graph_triples 结合起来做个可视化"
