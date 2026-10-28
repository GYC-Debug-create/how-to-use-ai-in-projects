# 斗罗大陆·万象图鉴 前端重写 Prompt

请基于以下完整的后端数据结构和资源，重写 `public/codex.html` 为一个**单文件、纯前端、零依赖（仅用原生 HTML/CSS/JS，不引入任何框架或CDN）**的图鉴页面。

---

## 一、后端环境

- **Express 服务器**运行在 `http://localhost:3000`，使用 `express.static` 托管 `public/` 目录
- 所有 JSON 数据文件位于 `public/data/`，可直接通过 `/data/xxx.json` 用 `fetch` 获取
- 所有图片位于 `public/assets/` 下，可通过 `/assets/xxx.jpg` 直接访问
- 页面入口：`http://localhost:3000/codex.html`

---

## 二、数据结构

### 1. 武魂数据 `/data/spirit-codex.json`

```json
{
  "meta": {
    "title": "斗罗大陆·武魂图鉴",
    "description": "收录斗罗大陆世界中已知的武魂…",
    "tiers": {
      "low":   { "name": "低级武魂", "color": "#94a3b8", "desc": "凡俗之物，无攻伐之力" },
      "mid":   { "name": "中级武魂", "color": "#6b8e5a", "desc": "器物附灵，粗具战力" },
      "high":  { "name": "高级武魂", "color": "#4a7fb5", "desc": "超凡脱俗，攻伐凌厉" },
      "top":   { "name": "顶级武魂", "color": "#9b6ec5", "desc": "世间绝品，封号可期" },
      "god":   { "name": "神级武魂", "color": "#c9a84c", "desc": "神位所寄，直登神境" }
    }
  },
  "spirits": [
    {
      "id": "water_pot",
      "name": "水壶",
      "tier": "low",
      "type": "器武魂",
      "subtype": "生活器物",
      "maxLevel": 10,
      "attack": 0, "defense": 2, "hp": 60, "speed": 5,
      "rarity": "极常见",
      "color": "#94a3b8",
      "icon": "🫖",
      "desc": "一段详细背景故事…",
      "traits": ["无攻击力", "储水", "生存辅助"],
      "image": "assets/spirits/water_pot.jpg"
    }
    // … 共 70 个
  ]
}
```

### 2. 魂兽数据 `/data/beast-codex.json`

```json
{
  "meta": {
    "title": "斗罗大陆·魂兽图鉴",
    "tiers": {
      "ten":              { "name": "十年魂兽",   "color": "#a09888" },
      "hundred":          { "name": "百年魂兽",   "color": "#b8a030" },
      "thousand":         { "name": "千年魂兽",   "color": "#8a5eb8" },
      "ten_thousand":     { "name": "万年魂兽",   "color": "#6a6a7a" },
      "hundred_thousand": { "name": "十万年魂兽", "color": "#b83030" }
    }
  },
  "beasts": [
    {
      "id": "spirit_deer",
      "name": "灵鹿",
      "tier": "ten",
      "type": "草食系",
      "habitat": "星斗大森林外围",
      "attack": 5, "defense": 3, "hp": 30, "speed": 12,
      "rarity": "极常见",
      "color": "#fbbf24",
      "icon": "🦌",
      "ringLevel": 1,
      "ringColor": "白色",
      "desc": "一段详细背景故事…",
      "traits": ["被动", "温顺", "速度较快"],
      "specialAbility": "无",
      "image": "assets/beasts/spirit_deer.jpg"
    }
    // … 共 30 个
  ]
}
```

### 3. 魂环数据 `/data/ring-codex.json`

```json
{
  "meta": {
    "title": "斗罗大陆·魂环图鉴",
    "tiers": {
      "white":   { "name": "十年白环", "color": "#f5f5f5" },
      "yellow":  { "name": "百年黄环", "color": "#facc15" },
      "purple":  { "name": "千年紫环", "color": "#a855f7" },
      "black":   { "name": "万年黑环", "color": "#1f2937" },
      "red":     { "name": "十万年红环", "color": "#dc2626" },
      "special": { "name": "特殊魂环", "color": "#fbbf24" }
    }
  },
  "rings": [
    {
      "id": "white_ring",
      "name": "十年白环",
      "tier": "white",
      "color": "#f5f5f5",
      "minLevel": 10, "maxLevel": 29,
      "statBonus": { "attack": 2, "defense": 1, "hp": 20 },
      "dropRate": 0.7,
      "sourceBeasts": ["灵鹿", "独眼狼", "野兔", "蓝银草蛇"],
      "rarity": "常见",
      "icon": "⚪",
      "desc": "一段详细背景故事…",
      "absorptionRisk": "极低",
      "specialNote": "大多数魂师的第一个魂环",
      "image": "assets/rings/white_ring.jpg"
    }
    // … 共 12 个
  ]
}
```

---

## 三、资源清单

### 图片目录
- 武魂图片 (70张): `assets/spirits/*.jpg` (2048×2048)
- 魂兽图片 (30张): `assets/beasts/*.jpg` (2048×2048)
- 魂环图片 (12张): `assets/rings/*.jpg` (2048×2048)
- 图鉴封面: `assets/beast_codex_master.jpg`, `assets/ring_codex_master.jpg`

### 每个条目的 `image` 字段就是图片的相对路径，直接用作 `<img src="...">` 即可。

---

## 四、页面功能需求

### 4.1 整体架构
- **纯单文件** `codex.html`，内联所有 CSS 和 JS
- **三标签页切换**: 武魂典 / 魂兽录 / 魂环谱
- 点击标签时 fetch 对应 JSON，渲染卡片网格
- 页面加载时默认显示武魂图鉴

### 4.2 顶部区域
- 标题（毛笔书法风格大字，如"斗罗大陆·万象图鉴"）
- 副标题随标签切换变化（取自 JSON 的 `meta.description`）
- **统计条**: 显示各等级条目数量，点击可快速筛选到对应等级
- **封面 banner**: 每个标签页有一张主视觉图
  - 武魂: `assets/spirits/blue_silver_grass.jpg`
  - 魂兽: `assets/beast_codex_master.jpg`
  - 魂环: `assets/ring_codex_master.jpg`
  - banner 带底部渐变遮罩，hover 时放大并提亮

### 4.3 筛选与搜索
- **等级筛选标签**: 根据当前图鉴类型的 tiers 动态生成，点击切换，"全部"显示所有等级
- **搜索框**: 实时搜索 name / type / subtype / habitat / traits / desc 等字段
- 筛选+搜索可叠加使用

### 4.4 卡片展示
- 按等级分区，每个分区有等级标题+颜色标记+描述
- 卡片网格，响应式（`auto-fill, minmax(170px, 1fr)`）
- 每张卡片包含：
  - **图片区域**: 有 image 字段就显示 `<img>`，无则显示 icon（emoji）
  - **名称**: 毛笔风格字体
  - **类型行**: 根据类型显示不同信息
    - 武魂: `{type} · {subtype} · {rarity}`
    - 魂兽: `{type} · {habitat} · {rarity}`
    - 魂环: `Lv.{minLevel}-{maxLevel} · {rarity} · {absorptionRisk}`
  - **特性标签**: 显示 traits 数组前3个
- 图片 hover 效果：从泛旧色调（sepia/暗淡）渐变到正常彩色，轻微放大

### 4.5 详情弹窗
- 点击卡片弹出详情面板（古卷展开动画）
- 弹窗包含：
  - **左侧大图** + **右侧标题+等级标签+类型信息**
  - **背景故事**: 完整 desc 文字，首行缩进两字符
  - **属性面板**: 2列网格
    - 武魂: 攻击/防御/生命/速度/潜力(maxLevel)级/类型
    - 魂兽: 攻击/防御/生命/速度/栖息地/魂环颜色
    - 魂环: 攻击加成/防御加成/生命加成/掉落概率/吸收等级/吸收风险
  - **额外信息**（如果有）:
    - 魂兽: `specialAbility`（非"无"时显示）, 左侧竖线装饰
    - 魂环: `sourceBeasts` 数组, `specialNote`
  - **特性标签**: 完整 traits 数组
- 点击遮罩或按 ESC 关闭弹窗

### 4.6 掉落概率格式化函数
```
formatDropRate(rate):
  percent = rate × 100
  if percent === 0 → "0%"
  if percent < 0.1 → 显示两位小数，去除末尾0
  if percent < 1 → 显示一位小数，去除末尾0
  else → 四舍五入取整 + "%"
```
示例: 0.7→"70%", 0.18→"18%", 0.06→"6%", 0.012→"1.2%", 0.001→"0.1%"

---

## 五、设计风格

### 5.1 视觉风格：古朴典籍/古卷翻阅感
- **色调**: 暗褐色底（`#1a1408`），金叶色点缀（`#c9a84c`），墨色文字（`#d4c5a0`），暗淡灰（`#8a7d62`）
- **纸张纹理**: `body::before` 叠加 repeating-linear-gradient 模拟竖线纤维纹理 + 径向渐变模拟边缘暗角
- **字体**:
  - 标题/等级名/卡片名: **Ma Shan Zheng**（Google Fonts，毛笔书法体）
  - 正文/标签/数据: **Noto Serif SC**（衬线宋体，Google Fonts）
  - fallback: `SimSun, 宋体, serif`
- **金色装饰**: 标题下方金色短线、等级标签金边、卡片 hover 金色边框光晕
- **印章红色**: `#8b2500` 作为强调辅助色（如副标题下划线）

### 5.2 动画与交互
- **页面入场**: 等级分区依次淡入上移（`fadeInUp`，`animation-delay` 递增 80ms）
- **卡片 hover**: `translateY(-4px)` + 金色阴影光晕 + 金色边框
- **图片 hover**: 从 `sepia(.15) saturate(.9) brightness(.85)` 渐变到 `sepia(0) saturate(1.1) brightness(1)` + `scale(1.05)`
- **弹窗打开**: `scale(.95)→scale(1)` + `translateY(10px)→0`，350ms ease
- **标签切换**: 下划线从 `scaleX(0)` 到 `scaleX(1)` 的过渡动画
- **筛选标签 hover**: 轻微上移 + 边框变金色

### 5.3 滚动条
- 自定义暗色滚动条，hover 时变金色

### 5.4 响应式
- 移动端: `max-width:600px` 时卡片网格变为 `minmax(140px, 1fr)`，标题缩小

---

## 六、技术约束

1. **纯原生**: 不使用 React/Vue/任何框架，不引入 Tailwind，不引入任何 CDN
2. **单文件**: 所有 CSS 写在 `<style>` 里，所有 JS 写在 `<script>` 里
3. **fetch 加载数据**: 页面内用 `fetch('/data/spirit-codex.json')` 等获取数据
4. **图片路径**: 直接使用数据中的 `image` 字段值作为 `<img src>`
5. **兼容性**: 保证在现代 Chrome/Edge/Safari 正常工作
6. **`itemsKey`**: 武魂用 `spirits`，魂兽用 `beasts`，魂环用 `rings`

---

## 七、连通性要求

页面必须能正确连接到：
1. 三个 JSON 数据端点（`/data/spirit-codex.json`、`/data/beast-codex.json`、`/data/ring-codex.json`）
2. 112 张图片资源（`/assets/spirits/*.jpg`、`/assets/beasts/*.jpg`、`/assets/rings/*.jpg`）
3. 首页链接（`/` 返回 `index.html`，即游戏主页）
4. 返回按钮 `<a href="/">← 归去</a>` 能回到游戏主页

---

## 八、输出

请直接输出完整的 `codex.html` 文件内容（一个完整的 HTML 文件），包含所有内联 CSS 和 JS。确保代码可复制粘贴后直接使用。
