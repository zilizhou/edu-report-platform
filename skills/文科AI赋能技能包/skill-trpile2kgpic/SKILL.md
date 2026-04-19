---
name: knowledge-graph-viz
description: >
  读取课程知识三元组 CSV（含 subject / relation / object / subject_type /
  object_type 字段），用 Python 生成一个内嵌 D3.js 的独立交互式 HTML 文件，
  支持力导向网络图、节点拖拽、搜索、关系过滤、节点详情面板。
  节点颜色代表知识点类型，边颜色与线型代表关系类型，节点大小反映连接度。
  当用户提到"知识图谱"、"三元组可视化"、"关系网络图"、"课程结构图"、
  "知识点关系图"，或上传含 subject/relation/object 列的 CSV 并要求
  "画出来"/"生成图"/"可视化"/"生成HTML"时，务必触发本技能。
---

# 知识图谱交互式 HTML 可视化

## 技术方案

- **Python**：读取 CSV，构建节点/边数据，序列化为 JSON，拼接 HTML 字符串后写文件
- **前端**：D3.js v7（CDN），力导向图，全交互，无需安装任何额外 Python 包
- **输出**：单个独立 `.html` 文件，浏览器直接打开即可使用

---

## 输入 CSV 格式

| 字段 | 必填 | 说明 |
|------|------|------|
| `subject` | ✅ | 关系主体节点名称 |
| `relation` | ✅ | 关系类型 |
| `object` | ✅ | 关系客体节点名称 |
| `subject_type` | 推荐 | 主体类型，决定节点颜色 |
| `object_type` | 推荐 | 客体类型，决定节点颜色 |
| `confidence` | 可选 | high / medium / low，可用于过滤行 |

---

## 完整生成脚本

将 `CSV_PATH`、`OUTPUT_PATH`、`TITLE` 三处改为实际值后直接运行。

```python
import pandas as pd, json

# ── 0. 配置 ──────────────────────────────────────────────────────────────
CSV_PATH    = "knowledge_graph_triples.csv"   # <- 输入 CSV 路径
OUTPUT_PATH = "knowledge_graph.html"          # <- 输出 HTML 路径
TITLE       = "课程知识图谱"                   # <- 页面标题

# ── 1. 颜色 / 样式（按需增删）────────────────────────────────────────────
TYPE_COLOR = {
    # 节点用低饱和度深色，让关系连线成为视觉主体
    '章节': '#3730a3', '概念': '#1e40af', '协议': '#0e7490',
    '方法': '#065f46', '技术': '#92400e', '属性': '#831843',
    '模型': '#4c1d95', '应用': '#7c2d12', '实验': '#3f6212',
}
DEFAULT_NODE_COLOR = '#1e3a5f'

# dash 为 SVG stroke-dasharray，空字符串表示实线
# 关系连线用高亮色 + 足够粗的线宽，成为图中视觉焦点
REL_META = {
    'CONTAINS':        {'color': '#94a3b8', 'label': '包含',     'dash': ''},
    'PREREQUISITE_OF': {'color': '#38bdf8', 'label': '前置依赖', 'dash': '10,4'},
    'IS_A':            {'color': '#e879f9', 'label': '类属',     'dash': '6,4'},
    'CONTRASTS_WITH':  {'color': '#fb923c', 'label': '对比',     'dash': '6,4'},
    'USED_FOR':        {'color': '#4ade80', 'label': '应用',     'dash': ''},
}
DEFAULT_REL = {'color': '#94a3b8', 'label': '其他', 'dash': ''}

# ── 2. 读取数据 ───────────────────────────────────────────────────────────
df = pd.read_csv(CSV_PATH)
# 可选：df = df[df['confidence'] == 'high']

# ── 3. 构建节点 / 边 JSON ─────────────────────────────────────────────────
nodes_map, edges_list = {}, []
for _, row in df.iterrows():
    s, o, rel = str(row['subject']), str(row['object']), str(row['relation'])
    st = str(row.get('subject_type', '概念'))
    ot = str(row.get('object_type',  '概念'))
    if s not in nodes_map:
        nodes_map[s] = {'id': s, 'label': s, 'ntype': st,
                        'color': TYPE_COLOR.get(st, DEFAULT_NODE_COLOR)}
    if o not in nodes_map:
        nodes_map[o] = {'id': o, 'label': o, 'ntype': ot,
                        'color': TYPE_COLOR.get(ot, DEFAULT_NODE_COLOR)}
    m = REL_META.get(rel, DEFAULT_REL)
    edges_list.append({'source': s, 'target': o, 'relation': rel,
                       'color': m['color'], 'label': m['label'], 'dash': m['dash']})

nodes_list = list(nodes_map.values())
type_legend = [{'color': TYPE_COLOR[t], 'label': t}
               for t in TYPE_COLOR if any(n['ntype'] == t for n in nodes_list)]
rel_legend  = [{'key': r, 'color': REL_META[r]['color'], 'label': REL_META[r]['label'],
                'dash': REL_META[r]['dash']}
               for r in REL_META if any(e['relation'] == r for e in edges_list)]

NJ  = json.dumps(nodes_list,  ensure_ascii=False)
EJ  = json.dumps(edges_list,  ensure_ascii=False)
TLJ = json.dumps(type_legend, ensure_ascii=False)
RLJ = json.dumps(rel_legend,  ensure_ascii=False)

# ── 4. 生成 HTML（D3.js 力导向图）────────────────────────────────────────
# 注意：模板使用 Python f-string，{{ }} 是字面量大括号
html = f"""<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>{TITLE}</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/7.8.5/d3.min.js"></script>
<style>
*{{box-sizing:border-box;margin:0;padding:0}}
body{{background:#0d1117;color:#e6edf3;font-family:'PingFang SC','Microsoft YaHei',sans-serif;overflow:hidden}}
#topbar{{position:fixed;top:0;left:0;right:0;height:52px;background:#161b22;
  border-bottom:1px solid #30363d;display:flex;align-items:center;
  padding:0 18px;gap:12px;z-index:200}}
#topbar h1{{font-size:16px;font-weight:700;white-space:nowrap}}
.tag{{padding:3px 10px;border-radius:20px;font-size:12px;
  background:#21262d;color:#8b949e;border:1px solid #30363d}}
.spacer{{flex:1}}
#search{{background:#21262d;border:1px solid #30363d;border-radius:8px;
  color:#e6edf3;font-size:13px;padding:5px 32px 5px 12px;width:190px;outline:none}}
#search::placeholder{{color:#484f58}}
#search:focus{{border-color:#58a6ff}}
#search-wrap{{position:relative;display:flex;align-items:center}}
#search-clear{{position:absolute;right:9px;cursor:pointer;color:#484f58;display:none}}
#sidebar{{position:fixed;top:52px;left:0;width:210px;bottom:0;
  background:#161b22;border-right:1px solid #30363d;overflow-y:auto;padding:14px;z-index:100}}
.pt{{font-size:10px;text-transform:uppercase;letter-spacing:1.2px;
  color:#484f58;margin:14px 0 8px;font-weight:600}}
.pt:first-child{{margin-top:0}}
.lr{{display:flex;align-items:center;gap:8px;margin-bottom:6px;font-size:12px;
  cursor:pointer;padding:4px 6px;border-radius:6px;user-select:none}}
.lr:hover{{background:#21262d}}
.lr input{{cursor:pointer;accent-color:#58a6ff}}
.dot{{width:12px;height:12px;border-radius:50%;flex-shrink:0}}
.lp{{width:26px;height:2px;flex-shrink:0;border-radius:1px}}
.sb{{background:#0d1117;border:1px solid #21262d;border-radius:8px;padding:10px 12px;font-size:12px}}
.sr{{display:flex;justify-content:space-between;padding:3px 0;color:#8b949e}}
.sr span:last-child{{font-weight:700;color:#e6edf3}}
#canvas{{position:fixed;top:52px;left:210px;right:0;bottom:0}}
svg{{width:100%;height:100%}}
.btn{{width:100%;padding:7px;background:#21262d;border:1px solid #30363d;
  border-radius:6px;color:#e6edf3;font-size:12px;cursor:pointer;margin-bottom:6px}}
.btn:hover{{background:#30363d}}
#info{{position:fixed;top:62px;right:14px;width:220px;background:#161b22;
  border:1px solid #30363d;border-radius:10px;padding:14px;z-index:150;
  display:none;font-size:13px}}
#info .in{{font-size:15px;font-weight:700;margin-bottom:6px;word-break:break-all}}
#info .ir{{color:#8b949e;margin-bottom:3px}}
#info .ir span{{color:#e6edf3}}
#ic{{position:absolute;top:10px;right:12px;cursor:pointer;color:#484f58}}
#ic:hover{{color:#e6edf3}}
#nb{{margin-top:8px;max-height:140px;overflow-y:auto}}
.nb{{display:inline-block;font-size:11px;padding:2px 7px;border-radius:4px;
  margin:2px;background:#21262d;color:#8b949e;cursor:pointer;border:1px solid #30363d}}
.nb:hover{{border-color:#58a6ff;color:#58a6ff}}
#tip{{position:fixed;pointer-events:none;display:none;background:#161b22;
  border:1px solid #30363d;border-radius:7px;padding:8px 12px;font-size:12px;
  z-index:300;box-shadow:0 4px 16px rgba(0,0,0,.6)}}
</style>
</head>
<body>
<div id="topbar">
  <h1>📡 {TITLE}</h1>
  <span class="tag" id="nc"></span>
  <span class="tag" id="ec"></span>
  <div class="spacer"></div>
  <div id="search-wrap">
    <input id="search" placeholder="搜索知识点…" autocomplete="off">
    <span id="search-clear">✕</span>
  </div>
</div>
<div id="sidebar">
  <div class="pt">统计</div><div class="sb" id="sbox"></div>
  <div class="pt">节点类型</div><div id="tleg"></div>
  <div class="pt">关系类型</div><div id="rleg"></div>
  <div class="pt">操作</div>
  <button class="btn" onclick="resetZoom()">⛶ 适应窗口</button>
  <button class="btn" id="lbtn" onclick="toggleLabels()">🏷 隐藏标签</button>
  <button class="btn" onclick="pinAll(false)">📌 释放所有固定</button>
</div>
<div id="canvas"><svg id="g"></svg></div>
<div id="info">
  <span id="ic" onclick="closeInfo()">✕</span>
  <div class="in" id="iname"></div>
  <div class="ir">类型：<span id="itype"></span></div>
  <div class="ir">连接数：<span id="ideg"></span></div>
  <div id="nb"></div>
</div>
<div id="tip">
  <div id="tn" style="font-weight:700;color:#e6edf3;margin-bottom:3px"></div>
  <div id="tm" style="color:#8b949e"></div>
</div>
<script>
const NODES={NJ};
// RAW_EDGES 始终保持原始字符串 source/target，不被 D3 forceLink 污染
const RAW_EDGES={EJ};
const TL={TLJ};const RL={RLJ};
const deg={{}};NODES.forEach(n=>deg[n.id]=0);
RAW_EDGES.forEach(e=>{{deg[e.source]=(deg[e.source]||0)+1;deg[e.target]=(deg[e.target]||0)+1;}});
const nb={{}};NODES.forEach(n=>nb[n.id]={{o:[],i:[]}});
RAW_EDGES.forEach(e=>{{nb[e.source].o.push(e.target);nb[e.target].i.push(e.source);}});
document.getElementById('nc').textContent=NODES.length+' 节点';
document.getElementById('ec').textContent=RAW_EDGES.length+' 关系';
const tc={{}};NODES.forEach(n=>tc[n.ntype]=(tc[n.ntype]||0)+1);
const rc={{}};RAW_EDGES.forEach(e=>rc[e.label]=(rc[e.label]||0)+1);
document.getElementById('sbox').innerHTML=
  Object.entries(tc).map(([t,c])=>`<div class="sr"><span>${{t}}</span><span>${{c}}</span></div>`).join('')+
  '<div style="border-top:1px solid #21262d;margin:5px 0"></div>'+
  Object.entries(rc).map(([l,c])=>`<div class="sr"><span>${{l}}</span><span>${{c}}</span></div>`).join('');
const tas=new Set(NODES.map(n=>n.ntype));
const ras=new Set(RAW_EDGES.map(e=>e.relation));
function buildLeg(id,items,key,as,refresh){{
  const el=document.getElementById(id);
  items.forEach(item=>{{
    const row=document.createElement('label');row.className='lr';
    const cb=document.createElement('input');cb.type='checkbox';cb.checked=true;
    cb.addEventListener('change',()=>{{cb.checked?as.add(item[key]):as.delete(item[key]);refresh();}});
    row.appendChild(cb);
    if(id==='tleg'){{const d=document.createElement('span');d.className='dot';d.style.background=item.color;row.appendChild(d);}}
    else{{const l=document.createElement('span');l.className='lp';l.style.background=item.color;
      if(item.dash)l.style.backgroundImage=`repeating-linear-gradient(90deg,${{item.color}} 0,${{item.color}} 5px,transparent 5px,transparent 9px)`;
      row.appendChild(l);}}
    const t=document.createElement('span');t.textContent=item.label;row.appendChild(t);
    el.appendChild(row);
  }});
}}
buildLeg('tleg',TL,'label',tas,render);
buildLeg('rleg',RL,'key',ras,render);
const svg=d3.select('#g');const gEl=svg.append('g');
const zm=d3.zoom().scaleExtent([0.1,4]).on('zoom',e=>gEl.attr('transform',e.transform));
svg.call(zm).on('dblclick.zoom',null);
const defs=svg.append('defs');
const flt=defs.append('filter').attr('id','glow').attr('x','-30%').attr('y','-30%').attr('width','160%').attr('height','160%');
flt.append('feGaussianBlur').attr('stdDeviation','2.5').attr('result','blur');
const fm=flt.append('feMerge');
fm.append('feMergeNode').attr('in','blur');
fm.append('feMergeNode').attr('in','SourceGraphic');
const relColors={{}};RAW_EDGES.forEach(e=>relColors[e.relation]=e.color);
Object.entries(relColors).forEach(([r,c])=>{{
  defs.append('marker').attr('id','a-'+r)
    .attr('viewBox','0 -5 10 10').attr('refX',22).attr('refY',0)
    .attr('markerWidth',7).attr('markerHeight',7).attr('orient','auto')
    .append('path').attr('d','M0,-5L10,0L0,5').attr('fill',c).attr('opacity',1);
}});
let sim,ns,ls,lbs,showL=true;
const nodeById=Object.fromEntries(NODES.map(n=>[n.id,n]));
function nodeR(d){{return Math.max(10,Math.min(28,10+(deg[d.id]||0)*2));}}
function render(){{
  const checkedIds=new Set(NODES.filter(n=>tas.has(n.ntype)).map(n=>n.id));
  const filteredRaw=RAW_EDGES.filter(e=>ras.has(e.relation)&&(checkedIds.has(e.source)||checkedIds.has(e.target)));
  const ve=filteredRaw.map(e=>{{return {{...e}};}});
  const visIds=new Set(checkedIds);
  filteredRaw.forEach(e=>{{visIds.add(e.source);visIds.add(e.target);}});
  const vn=[...visIds].map(id=>nodeById[id]).filter(Boolean);
  if(sim)sim.stop();
  const W=document.getElementById('canvas').clientWidth;
  const H=document.getElementById('canvas').clientHeight;
  gEl.selectAll('*').remove();
  sim=d3.forceSimulation(vn)
    .force('link',d3.forceLink(ve).id(d=>d.id).distance(110))
    .force('charge',d3.forceManyBody().strength(-320))
    .force('center',d3.forceCenter(W/2,H/2))
    .force('collide',d3.forceCollide(d=>nodeR(d)+12));
  ls=gEl.append('g').selectAll('line').data(ve).join('line')
    .attr('stroke',d=>d.color)
    .attr('stroke-width',d=>d.relation==='CONTAINS'?1.5:2.8)
    .attr('stroke-opacity',d=>d.relation==='CONTAINS'?0.55:1.0)
    .attr('stroke-dasharray',d=>d.dash||null)
    .attr('filter',d=>d.relation==='CONTAINS'?null:'url(#glow)')
    .attr('marker-end',d=>`url(#a-${{d.relation}})`)
    .on('mouseover',(e,d)=>tip(e,`${{d.source.id||d.source}} → ${{d.target.id||d.target}}`,d.label))
    .on('mousemove',mtip).on('mouseout',htip);
  ns=gEl.append('g').selectAll('circle').data(vn).join('circle')
    .attr('r',d=>nodeR(d)).attr('fill',d=>d.color)
    .attr('stroke',d=>d.color).attr('stroke-width',2.5).attr('cursor','grab')
    .call(d3.drag()
      .on('start',(e,d)=>{{if(!e.active)sim.alphaTarget(0.3).restart();d.fx=d.x;d.fy=d.y;}})
      .on('drag',(e,d)=>{{d.fx=e.x;d.fy=e.y;}})
      .on('end',(e,d)=>{{if(!e.active)sim.alphaTarget(0);}}))
    .on('click',(e,d)=>{{e.stopPropagation();openInfo(d);}})
    .on('mouseover',(e,d)=>tip(e,d.label,d.ntype+' · 度数 '+(deg[d.id]||0)))
    .on('mousemove',mtip).on('mouseout',htip);
  lbs=gEl.append('g').selectAll('text').data(vn).join('text')
    .text(d=>d.label)
    .attr('font-size',d=>nodeR(d)>14?'11px':'9px')
    .attr('fill','#e6edf3').attr('text-anchor','middle')
    .attr('dy',d=>nodeR(d)+12).attr('pointer-events','none')
    .attr('paint-order','stroke').attr('stroke','#0d1117').attr('stroke-width',3)
    .attr('display',showL?null:'none');
  sim.on('tick',()=>{{
    ls.attr('x1',d=>d.source.x).attr('y1',d=>d.source.y)
      .attr('x2',d=>d.target.x).attr('y2',d=>d.target.y);
    ns.attr('cx',d=>d.x).attr('cy',d=>d.y);
    lbs.attr('x',d=>d.x).attr('y',d=>d.y);
  }});
}}
render();
function tip(e,n,m){{
  document.getElementById('tn').textContent=n;document.getElementById('tm').textContent=m;
  const t=document.getElementById('tip');
  t.style.display='block';t.style.left=(e.clientX+14)+'px';t.style.top=(e.clientY-10)+'px';
}}
function mtip(e){{const t=document.getElementById('tip');t.style.left=(e.clientX+14)+'px';t.style.top=(e.clientY-10)+'px';}}
function htip(){{document.getElementById('tip').style.display='none';}}
function openInfo(d){{
  document.getElementById('iname').textContent=d.label;
  document.getElementById('itype').textContent=d.ntype;
  document.getElementById('ideg').textContent=(deg[d.id]||0)+'条';
  const all=[...new Set([...nb[d.id].o,...nb[d.id].i])];
  document.getElementById('nb').innerHTML=all.length
    ?'<div style="color:#484f58;font-size:11px;margin-bottom:5px">相关节点：</div>'+
      all.map(id=>`<span class="nb" onclick="focusNode('${{id}}')">${{id}}</span>`).join('')
    :'<div style="color:#484f58;font-size:11px">无相邻节点</div>';
  document.getElementById('info').style.display='block';
  const rel=new Set([d.id,...nb[d.id].o,...nb[d.id].i]);
  ns.attr('opacity',n=>rel.has(n.id)?1:0.15);
  lbs.attr('opacity',n=>rel.has(n.id)?1:0.08);
  ls.attr('stroke-opacity',e=>{{
    const s=e.source.id||e.source,t=e.target.id||e.target;
    const hit=rel.has(s)&&rel.has(t);
    return hit?(e.relation==='CONTAINS'?0.7:1.0):0.04;
  }});
}}
function closeInfo(){{
  document.getElementById('info').style.display='none';
  if(ns){{ns.attr('opacity',1);lbs.attr('opacity',1);}}
  if(ls)ls.attr('stroke-opacity',d=>d.relation==='CONTAINS'?0.55:1.0);
}}
function focusNode(id){{const n=nodeById[id];if(n)openInfo(n);}}
svg.on('click',closeInfo);
const si=document.getElementById('search'),sc=document.getElementById('search-clear');
si.addEventListener('input',()=>{{
  const q=si.value.trim().toLowerCase();sc.style.display=q?'block':'none';
  if(!ns)return;
  if(!q){{closeInfo();return;}}
  const m=new Set(NODES.filter(n=>n.id.toLowerCase().includes(q)).map(n=>n.id));
  ns.attr('opacity',n=>m.has(n.id)?1:0.1);lbs.attr('opacity',n=>m.has(n.id)?1:0.05);
  ls.attr('stroke-opacity',0.04);
}});
sc.addEventListener('click',()=>{{si.value='';sc.style.display='none';closeInfo();}});
function resetZoom(){{
  try{{
    const b=gEl.node().getBBox(),W=document.getElementById('canvas').clientWidth,H=document.getElementById('canvas').clientHeight;
    const s=Math.min(0.9,0.9*Math.min(W/b.width,H/b.height));
    svg.transition().duration(600).call(zm.transform,
      d3.zoomIdentity.translate(W/2-s*(b.x+b.width/2),H/2-s*(b.y+b.height/2)).scale(s));
  }}catch(e){{}}
}}
function toggleLabels(){{
  showL=!showL;if(lbs)lbs.attr('display',showL?null:'none');
  document.getElementById('lbtn').textContent=(showL?'🏷 隐藏':'🏷 显示')+'标签';
}}
function pinAll(s){{NODES.forEach(n=>{{n.fx=s?n.x:null;n.fy=s?n.y:null;}});if(sim)sim.alpha(0.3).restart();}}
window.addEventListener('resize',()=>render());
</script></body></html>"""