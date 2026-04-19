---
name: exam-kp-analysis
description: >
  在试卷成绩分析的基础上，融合知识点统计表、学生知识画像、知识图谱三类文件，
  生成一份带丰富可视化图表的"试卷知识点深度分析报告"（单文件 HTML）。
  报告涵盖：知识点班级得分率排名、高风险知识点热点、Bloom 认知层次分析、
  教学章节掌握雷达图、学生个人知识画像明细、以及知识图谱驱动的教学路径建议。
  支持六类输入文件的任意组合（按可用文件自动降级分析深度）。

  当用户上传以下任意组合文件，并提及"知识点分析"、"知识点掌握情况"、
  "哪些知识点没学好"、"薄弱知识点"、"认知能力分析"、"章节掌握情况"、
  "学生知识画像"、"知识图谱"等需求时，务必触发本技能。
  即使用户只说"分析一下这次考试哪里没教好"，只要同时存在成绩单与知识点相关文件，
  也应使用本技能。
---

# 试卷知识点分析技能（Exam KP Analysis Skill）

## 一、技能定位

本技能是 **exam-analysis**（基础试卷分析）的**增强版**，在其基础上增加：

| 增量分析维度 | 新增文件 | 核心图表 |
|---|---|---|
| 知识点班级得分率排名 | `knowledge_point_statistics.csv` | 薄弱知识点柱状图、最强知识点柱状图 |
| Bloom 认知层次分布 | `knowledge_point_statistics.csv` | 认知层次折线图、饼图、柱状图 |
| 教学章节掌握全貌 | `knowledge_graph_triples.csv` | 章节雷达图、气泡图、章节明细表 |
| 班级高风险知识点 | `student_summary_profile.csv` | 高风险知识点横向柱状图 |
| 学生个人知识画像 | `student_summary_profile.csv` | 学生画像表（优势/薄弱/高风险/认知） |
| 知识图谱先修关系 | `knowledge_graph_triples.csv` | 教学路径建议文字段落 |

---

## 二、输入文件说明（六类）

### 基础三类（来自 exam-analysis 技能）

| 文件 | 关键列 | 用途 |
|---|---|---|
| 试卷 `.docx` | 题型/满分/题号 | 解析大题结构、小题分值 |
| 成绩单（任意粒度）`.csv` | `学号,姓名,总分` 或含大题/小题列 | 计算各题得分率 |
| — | — | — |

### 新增三类（知识点维度）

| 文件 | 关键列 | 说明 |
|---|---|---|
| `knowledge_point_statistics.csv` | `knowledge_point, question_count, question_ids, total_score, difficulty_distribution, avg_cognitive_level` | 每个知识点对应哪些题、分值、难度、Bloom 层次 |
| `student_summary_profile.csv` | `student_id, student_name, total_score_rate, excellent_kps, good_kps, fair_kps, weak_kps, poor_kps, top3_strength, top3_weakness, top3_high_risk, cognitive_strength, cognitive_weakness, overall_level` | 每位学生在各知识点上的掌握等级 |
| `knowledge_graph_triples.csv` | `subject, relation, object, subject_type, object_type, source` | 知识图谱三元组：章节包含关系、先修关系、概念层次 |

---

## 三、文件可用性降级规则

```
完整6文件 → 全量分析（7个图表分组、学生画像、知识图谱路径）
缺 knowledge_graph_triples → 无章节雷达图、无先修关系建议
缺 student_summary_profile → 无高风险知识点、无学生画像
缺 knowledge_point_statistics → 无Bloom分析，仅保留基础试卷分析
仅有成绩单+试卷 → 退回 exam-analysis 技能
```

---

## 四、执行流程

### 步骤 1：读取全部文件

```python
import pandas as pd, numpy as np, json

kps_df = pd.read_csv("knowledge_point_statistics.csv")
sp_df  = pd.read_csv("student_summary_profile.csv")
kg_df  = pd.read_csv("knowledge_graph_triples.csv")
sc_df  = pd.read_csv("成绩单3_小题分_总分.csv")  # 优先使用小题分粒度
```

### 步骤 2：建立题号→列名映射

```python
# 根据试卷结构（SECTIONS 配置），将 question_ids（1-based 全局题号）映射到成绩单列名
SECTIONS = [("选择题",20,20,1,"选择"),("判断题",10,10,1,"判断"),
            ("填空题",10,5,2,"填空"),("简答题",30,5,6,"简答"),("综合题",30,3,10,"综合")]
qmap = {}   # {题号: "选择1", ...}
qno = 1
for (sec_name, full, cnt, item_full, prefix) in SECTIONS:
    for i in range(cnt):
        qmap[qno] = f"{prefix}{i+1}"
        qno += 1
```

### 步骤 3：计算各知识点班级得分率

```python
kp_stats = []
for _, row in kps_df.iterrows():
    kp   = row['knowledge_point']
    qids = [int(x) for x in str(row['question_ids']).split(';') if x.strip().isdigit()]
    cols = [qmap[q] for q in qids if q in qmap and qmap[q] in sc_df.columns]
    if not cols:
        continue
    
    def col_full(c):   # 获取该列的满分
        for (sn,full,cnt,ifull,pfx) in SECTIONS:
            for i in range(cnt):
                if c == f"{pfx}{i+1}": return ifull
        return 1
    
    total_possible = sum(col_full(c) for c in cols) * len(sc_df)
    total_got      = sum(sc_df[c].sum() for c in cols)
    class_rate     = round(total_got / total_possible * 100, 1) if total_possible > 0 else 0
    
    kp_stats.append({
        "kp": kp, "q_count": int(row['question_count']),
        "total_score": float(row['total_score']),
        "diff_dist": row['difficulty_distribution'],
        "cog_level": row['avg_cognitive_level'],
        "class_rate": class_rate,
    })
```

### 步骤 4：从学生画像提取班级高风险知识点

```python
# top3_high_risk 列：每位学生最容易失分的3个知识点
risk_counter = {}
for _, row in sp_df.iterrows():
    if pd.notna(row['top3_high_risk']):
        for kp in str(row['top3_high_risk']).split(';'):
            kp = kp.strip()
            if kp:
                risk_counter[kp] = risk_counter.get(kp, 0) + 1

top_risk = sorted(risk_counter.items(), key=lambda x: -x[1])[:10]
# → [(知识点名, 人次), ...]，人次越高说明越多学生在此处高风险
```

### 步骤 5：知识图谱 → 章节→知识点映射

```python
# 利用 CONTAINS 关系建立章节→子知识点的多级映射
chapter_rows = kg_df[kg_df['relation'] == 'CONTAINS']
top_chapters = ['物理层', '数据链路层', '网络层', '运输层', '应用层', '网络安全']

chapter_map = {}
for ch in top_chapters:
    # 一级子节点
    children = chapter_rows[chapter_rows['subject'] == ch]['object'].tolist()
    all_kps  = set(children)
    # 二级子节点
    for c in children:
        grandchildren = chapter_rows[chapter_rows['subject'] == c]['object'].tolist()
        all_kps.update(grandchildren)
    chapter_map[ch] = list(all_kps)

# 计算每章节的平均得分率
chapter_rates = {}
kp_rate_dict  = {s['kp']: s['class_rate'] for s in kp_stats}
for ch, kp_list in chapter_map.items():
    rates = [kp_rate_dict[kp] for kp in kp_list if kp in kp_rate_dict]
    if rates:
        chapter_rates[ch] = round(np.mean(rates), 1)

# 先修关系（用于教学建议）
prereq_rows = kg_df[kg_df['relation'] == 'PREREQUISITE_OF']
# → 例如"物理层 PREREQUISITE_OF 数据链路层"，若数据链路层薄弱，需先检查物理层
```

### 步骤 6：Bloom 认知层次汇总

```python
cog_map = {
    'remember':  '记忆（Bloom L1）',
    'understand':'理解（Bloom L2）',
    'apply':     '应用（Bloom L3）',
    'analyze':   '分析（Bloom L4）',
    'evaluate':  '评价（Bloom L5）',
    'create':    '创造（Bloom L6）',
}
cog_order = ['remember','understand','apply','analyze','evaluate','create']

cog_agg = {}
for s in kp_stats:
    lvl = s['cog_level']
    if lvl not in cog_agg:
        cog_agg[lvl] = {'count':0, 'rates':[], 'score':0}
    cog_agg[lvl]['count']  += 1
    cog_agg[lvl]['rates'].append(s['class_rate'])
    cog_agg[lvl]['score']  += s['total_score']

cog_summary = sorted([{
    'level': lvl, 'label': cog_map.get(lvl, lvl),
    'kp_count': d['count'],
    'avg_rate': round(np.mean(d['rates']), 1),
    'total_score': d['score']
} for lvl, d in cog_agg.items()],
key=lambda x: cog_order.index(x['level']) if x['level'] in cog_order else 99)
```

### 步骤 7：生成 HTML 报告

报告使用**标签页（Tab）导航**结构，共 6 个 Tab：

| Tab | 内容 | 主要图表 |
|---|---|---|
| 📊 总览 | 统计摘要卡片 + 四个核心图 | 水平饼图、认知层次柱状图、章节雷达图、高风险横向柱状图 |
| ⚠️ 知识点薄弱 | 最弱 20 个 + 最强 10 个 + 明细表 | 两组柱状图、带颜色进度条表格 |
| 📚 章节分析 | 章节得分率 + 气泡图 + 明细表 | 柱状图、气泡图（x=知识点数, y=得分率, r=分值）|
| 🧩 认知层次 | Bloom 分析 + 学生认知优劣饼图 | 柱状图×2、折线图、饼图×2、汇总表 |
| 👤 学生画像 | 每位学生的优势/薄弱/高风险/认知 | 全量汇总表 |
| 📝 总结建议 | 五段式结构化文字总结 | 无图表，纯文字 |

---

## 五、图表规范（Chart.js 4.x）

### 5.1 图表总览

| 图表 | 类型 | 数据源 | 颜色规则 |
|---|---|---|---|
| 学生整体水平分布 | **饼图** | `overall_level` 频次 | good=绿，fair=黄，weak=红 |
| 各认知层次平均得分率 | **柱状图** | `cog_summary.avg_rate` | ≥75%=绿，≥65%=黄，<65%=红 |
| 各章节掌握情况 | **雷达图** | `chapter_rates` | 单色蓝色半透明填充 |
| 班级高风险知识点 | **横向柱状图** | `top_risk` | ≥10人=红，≥5人=黄，其余=蓝 |
| 最薄弱 20 知识点 | **竖向柱状图** | `kp_stats` 升序前20 | 与颜色规则一致 |
| 最强 10 知识点 | **竖向柱状图** | `kp_stats` 降序前10 | 固定绿色 |
| 章节得分率 | **柱状图** | `chapter_rates` | 同颜色规则 |
| 章节气泡图 | **气泡图** | x=知识点数, y=得分率, r=√分值×1.5 | 每章节固定颜色 |
| 认知层次知识点数 | **柱状图** | `cog_summary.kp_count` | 四层固定颜色 |
| 认知层次分值占比 | **饼图** | `cog_summary.total_score` | 同上 |
| 认知层次得分率趋势 | **折线图** | `cog_summary.avg_rate` | 颜色规则 |
| 认知优势分布 | **饼图** | `cognitive_strength` 频次 | 按层次固定色 |
| 认知薄弱分布 | **饼图** | `cognitive_weakness` 频次 | 红/橙 |

### 5.2 章节雷达图实现

```javascript
new Chart(ctx, {
  type: "radar",
  data: {
    labels: Object.keys(chapterRates),
    datasets: [{
      label: "章节掌握得分率(%)",
      data: Object.values(chapterRates),
      backgroundColor: "rgba(37,99,168,.15)",
      borderColor: "#2563a8",
      pointBackgroundColor: "#2563a8",
      borderWidth: 2
    }]
  },
  options: {
    plugins: { legend: { display: false }},
    scales: { r: { min: 0, max: 100, ticks: { stepSize: 20 }}}
  }
});
```

### 5.3 章节气泡图实现

```javascript
// x = 该章节的知识点数量
// y = 该章节的平均得分率
// r = Math.sqrt(该章节总分值) * 1.5  ← 面积正比于分值
new Chart(ctx, {
  type: "bubble",
  data: {
    datasets: chapters.map((ch, i) => ({
      label: ch.name,
      data: [{ x: ch.kp_count, y: ch.rate, r: Math.sqrt(ch.total_score) * 1.5 }],
      backgroundColor: colors[i] + "99"
    }))
  },
  options: {
    plugins: { tooltip: { callbacks: {
      label: c => `${c.dataset.label}: ${c.raw.y}% (${c.raw.x}个知识点)`
    }}},
    scales: {
      x: { title: { display: true, text: "知识点数量" }},
      y: { min: 60, max: 85, title: { display: true, text: "平均得分率(%)" }}
    }
  }
});
```

### 5.4 进度条（内嵌 HTML）

表格中的得分率列使用内嵌进度条代替纯数字，增强可读性：

```javascript
function pbar(rate) {
  const color = rate >= 75 ? "#16a34a" : rate >= 65 ? "#f59e0b" : "#dc2626";
  return `<div class="pbar-wrap">
    <div class="pbar" style="width:${Math.min(rate,100)}%;background:${color}"></div>
  </div> ${rate}%`;
}
```

---

## 六、报告结构（六个 Tab 详述）

### Tab 1：总览
```
统计摘要卡片（8张）：知识点总数、最低/最高得分率、高风险人次、学生水平分布、记忆层均分
四图并排：水平饼图 | 认知层次柱状图 | 章节雷达图 | 高风险横向柱状图
```

### Tab 2：知识点薄弱
```
最薄弱 20 知识点柱状图（升序，颜色编码）
最强 10 知识点柱状图（绿色）
需重点关注知识点明细表（得分率 < 65%，含进度条、认知层标签）
```

### Tab 3：章节分析
```
章节得分率柱状图
章节气泡图（知识点数 × 得分率 × 分值）
各章节知识点明细表（含先修关系提示）
```

### Tab 4：认知层次（Bloom）
```
认知层次知识点数柱状图 | 认知层次分值饼图 | 认知层次得分率折线图（三图并排）
学生认知优势分布饼图 | 学生认知薄弱分布饼图（两图并排）
认知层次汇总表
```

### Tab 5：学生画像
```
学生画像汇总表（每行一名学生）：学号 | 姓名 | 总分率 | 综合水平 |
  优势知识点TOP3 | 薄弱知识点TOP3 | 高风险知识点 | 认知优势 | 认知薄弱
综合水平颜色编码：good=绿，fair=黄，weak=红
```

### Tab 6：总结建议（五段式）
```
段落1：知识点整体掌握概况（均分、分布、记忆→应用递减趋势）
段落2：高风险知识点重点说明（TOP3热点及原因分析）
段落3：章节掌握差异分析（最强/最弱章节及原因）
段落4：认知能力结构问题（优势vs薄弱的悖论：懂原理但不会计算）
段落5：5条针对性教学改进建议（可操作、具体到知识点）
```

---

## 七、总结段落自动生成规则

| 触发条件 | 自动生成的文字要点 |
|---|---|
| 某知识点 `class_rate < 60%` | 标注为"过低"，建议重点专项训练 |
| 某知识点出现在 `top3_high_risk` ≥ 8 人次 | 标注为"全班共性薄弱点"，建议全体复习 |
| 某章节 `avg_rate < 70%` | 标注章节为薄弱，结合先修关系给出梯度复习建议 |
| 应用层 `avg_rate` < 记忆层 `avg_rate` - 5% | 输出"理论与实践落差明显"分析 |
| `cognitive_weakness` 集中在 apply/analyze | 输出"认知跃升障碍"建议（阶梯练习法）|
| `weak` 整体水平学生 ≥ 2 人 | 输出个性化辅导建议 |
| 某章节存在 `PREREQUISITE_OF` 先修关系且薄弱 | 输出"需先补全前置知识"建议 |

---

## 八、Python 完整脚本骨架

```python
import pandas as pd, numpy as np, json

# ── 0. 读取所有文件 ──────────────────────────────────
kps_df = pd.read_csv("/path/to/knowledge_point_statistics.csv")
sp_df  = pd.read_csv("/path/to/student_summary_profile.csv")
kg_df  = pd.read_csv("/path/to/knowledge_graph_triples.csv")
sc_df  = pd.read_csv("/path/to/成绩单.csv")

SECTIONS = [("选择题",20,20,1,"选择"),("判断题",10,10,1,"判断"),
            ("填空题",10,5,2,"填空"),("简答题",30,5,6,"简答"),("综合题",30,3,10,"综合")]

# ── 1. 题号→列名映射 ─────────────────────────────────
qmap = {}
qno = 1
for (sn,full,cnt,ifull,pfx) in SECTIONS:
    for i in range(cnt):
        qmap[qno] = f"{pfx}{i+1}"; qno += 1

# ── 2. 各知识点班级得分率 ────────────────────────────
kp_stats = []   # [{kp, q_count, total_score, diff_dist, cog_level, class_rate}, ...]
# （见步骤3详细代码）

# ── 3. 高风险知识点统计 ──────────────────────────────
risk_counter = {}
for _, row in sp_df.iterrows():
    if pd.notna(row['top3_high_risk']):
        for kp in str(row['top3_high_risk']).split(';'):
            kp=kp.strip(); risk_counter[kp]=risk_counter.get(kp,0)+1 if kp else risk_counter
top_risk = sorted(risk_counter.items(), key=lambda x: -x[1])[:10]

# ── 4. 章节→知识点映射与章节得分率 ─────────────────
chapter_rows   = kg_df[kg_df['relation']=='CONTAINS']
top_chapters   = ['物理层','数据链路层','网络层','运输层','应用层','网络安全']
chapter_map    = {}   # {章节: [知识点列表]}
chapter_rates  = {}   # {章节: 平均得分率}
# （见步骤5详细代码）

# ── 5. Bloom 认知层次汇总 ────────────────────────────
cog_summary = []   # [{level, label, kp_count, avg_rate, total_score}, ...]
# （见步骤6详细代码）

# ── 6. 学生画像整理 ──────────────────────────────────
student_rows = []
for _, row in sp_df.iterrows():
    student_rows.append({
        "id":       row['student_id'],
        "name":     row['student_name'],
        "rate":     float(row['total_score_rate']),
        "level":    row['overall_level'],
        "str":      str(row['top3_strength'])  if pd.notna(row['top3_strength'])  else "—",
        "weak":     str(row['top3_weakness'])  if pd.notna(row['top3_weakness'])  else "—",
        "risk":     str(row['top3_high_risk']) if pd.notna(row['top3_high_risk']) else "—",
        "cog_str":  row['cognitive_strength'],
        "cog_weak": row['cognitive_weakness'],
    })

# ── 7. 序列化为 JSON，嵌入 HTML ──────────────────────
report_data = json.dumps({
    "kp_stats":     kp_stats,
    "top_risk":     [{"kp":k,"count":c} for k,c in top_risk],
    "chapter_rates": chapter_rates,
    "cog_summary":  cog_summary,
    "students":     student_rows,
}, ensure_ascii=False)

# ── 8. 生成 HTML，写入文件 ────────────────────────────
OUTPUT_PATH = "/mnt/user-data/outputs/exam_knowledge_report.html"
with open(OUTPUT_PATH, "w", encoding="utf-8") as f:
    f.write(html_template.replace("__DATA__", report_data))
```

---

## 九、输出文件

| 文件 | 说明 |
|---|---|
| `exam_knowledge_report.html` | 带6个Tab的单文件交互式报告，可直接用浏览器打开 |

使用 `present_files` 将文件展示给用户。

---

## 十、与 exam-analysis 的关系

| 维度 | exam-analysis | exam-kp-analysis |
|---|---|---|
| 分析角度 | 题目维度（大题/小题） | 知识点维度（知识图谱驱动） |
| 输入文件 | 试卷 + 成绩单（1~3个） | 试卷 + 成绩单 + 知识点统计 + 学生画像 + 知识图谱（最多6个） |
| 核心图表 | 9种图表 | 13种图表（含雷达图、气泡图、认知层次组合） |
| 学生维度 | 无 | 每位学生的完整知识画像 |
| 建议粒度 | 题型级别 | 知识点级别 + 章节级别 + 认知层次级别 |
| 适用场景 | 快速了解考试表现 | 深度教研分析、下学期教学规划 |

两个技能可独立使用，也可先运行 exam-analysis 快速出报告，再运行本技能深度分析。

---

## 十一、注意事项

1. **`question_ids` 为全局题号（1-based）**：选择题1-20、判断题21-30、填空题31-35……需根据 `SECTIONS` 配置正确建立映射，否则知识点得分率计算错误。
2. **学生画像中 kp 列以 `;` 分隔**：解析时需 `str.split(';')` 并 `.strip()`，注意 NaN 处理。
3. **知识图谱 `CONTAINS` 关系是多级的**：章节→子章节→知识点，需两层遍历才能获取全部叶子知识点。
4. **Bloom 层次英文标签**：`remember/understand/apply/analyze/evaluate/create`，统一转换为中文后输出。
5. **Tab 切换用纯 JS 实现**（无需框架），`display:none/block` 控制显示，与 Chart.js 兼容。
6. **样本量 < 30 时**：高风险知识点统计结果仅供参考，报告中自动注明。
