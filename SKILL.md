---
name: 产品大牛转md
description: "将产品大牛（pmdaniu.com）上托管的 Axure 原型文档转换为结构化 Markdown，供 AI 知识库（RAG）和团队共享使用。 支持单个文档链接转换，自动处理多页 sitemap、截图、Vision 理解、Markdown 输出。 触发场景：用户给出产品大牛链接（u.pmdaniu.com 短链或 pmdaniu.com 长链）并要求转为 Markdown 或导入知识库。
"
---

# 产品大牛文档 → Markdown

将 Axure 原型文档转成结构化 Markdown，适合 RAG 检索和团队 AI 共享。

## 前置条件

- 用户 Chrome 已登录产品大牛（pmdaniu.com）
- web-access skill 的 CDP Proxy 可用（`bash ~/.myagents/projects/My-agnets/.claude/skills/web-access/scripts/check-deps.sh`）
- 输出目录：`outputs/产品大牛文档/`（自动创建）

## 核心技术细节（已验证）

### Vision 方式

**直接用 `Read` 工具读取截图 PNG 文件即可。** Claude 本身是多模态模型，不需要任何外部 Vision API、不需要子 Agent 加载 web-access skill，不需要 ANTHROPIC_API_KEY。所有外部 Vision API（Zhipu、DashScope 等兼容端点）均不可靠，已验证 Zhipu 会严重幻觉。

### 平台特征

- 文档类型：Axure RP 原型，本质是**线框图+便利贴标注**，不是纯文字 PRD
- 纯文字提取（innerText）质量差，必须用 **Vision 截图理解**
- 每个文档有多个子页面，通过 `$axure.document.sitemap` 获取完整树结构
- 页面间导航：**修改 `window.location.hash`** 为 `#id=<pageId>&p=<encodedPageName>`，无需重新加载
- 每次打开新 URL 都有反诈 jump 页，需点击 `.J_jump` 按钮；hash 导航不触发 jump 页

### URL 结构

```
短链：https://u.pmdaniu.com/XXXXX
长链：https://www.pmdaniu.com/clouds/{uid}/{hash}/start.html?id={pageId}&p={encodedName}
```

### CDP 操作关键点

```bash
# 1. 打开链接
curl -s "http://localhost:3456/new?url=<URL>"

# 2. 点过 jump 页（等 3 秒再点）
curl -s -X POST "http://localhost:3456/click?target=<ID>" -d '.J_jump'

# 3. 获取 sitemap
curl -s -X POST "http://localhost:3456/eval?target=<ID>" -d 'JSON.stringify($axure.document.sitemap)'

# 4. hash 导航到指定页（无 jump 页触发）
curl -s -X POST "http://localhost:3456/eval?target=<ID>" \
  -d 'window.location.hash = "#id=<pageId>&p=<encodedName>"; "ok"'
# 等待 2 秒加载

# 5. 截图
curl -s "http://localhost:3456/screenshot?target=<ID>&file=/tmp/pmdaniu_<docname>/page_<n>.png"

# 6. 关闭 tab
curl -s "http://localhost:3456/close?target=<ID>"
```

## 执行流程 — MANDATORY

### Step 1：启动并验证 CDP Proxy + Chrome 连接

```bash
# 启动 Proxy（已运行则跳过）
bash ~/.myagents/projects/My-agnets/.claude/skills/web-access/scripts/check-deps.sh

# 强制触发懒连接，并验证 Chrome 是否可达
# Proxy 启动后与 Chrome 是懒连接（health 返回 chromePort: null），
# 必须发一个实际请求才能建立连接，否则后续所有操作都会超时失败。
TARGETS=$(curl -s --max-time 5 "http://localhost:3456/targets" 2>/dev/null)
COUNT=$(echo "$TARGETS" | python3 -c "import sys,json; print(len(json.load(sys.stdin)))" 2>/dev/null)

if [ -n "$COUNT" ]; then
    echo "✓ Chrome 已连接，当前有 $COUNT 个 tab"
else
    echo "✗ 无法连接 Chrome，请按以下步骤检查："
    echo "  1. 确认 Chrome 已打开"
    echo "  2. 地址栏输入：chrome://inspect/#remote-debugging"
    echo "  3. 勾选「Allow remote debugging for this browser instance」"
    echo "  4. 若刚勾选，重启 Chrome 后重试"
    exit 1
fi
```

### Step 2：打开文档，处理 jump 页

```bash
TARGET=$(curl -s "http://localhost:3456/new?url=<用户给的链接>" | python3 -c "import sys,json;print(json.load(sys.stdin)['targetId'])")
sleep 3
curl -s -X POST "http://localhost:3456/click?target=$TARGET" -d '.J_jump'
sleep 4
# 确认已进入 Axure viewer（title 变为文档名，非"产品大牛 - 让产品..."）
curl -s "http://localhost:3456/info?target=$TARGET"
```

### Step 3：读取 sitemap，扁平化页面列表

```python
import subprocess, json, urllib.parse, os

def eval_js(target, js):
    r = subprocess.run(
        f'curl -s -X POST "http://localhost:3456/eval?target={target}" -d \'{js}\'',
        shell=True, capture_output=True, text=True
    )
    return json.loads(r.stdout).get("value", "")

sitemap_raw = eval_js(target, "JSON.stringify($axure.document.sitemap)")
sitemap = json.loads(sitemap_raw)

def flatten(nodes, path=[]):
    result = []
    for n in nodes:
        if n["type"] == "Folder":
            result.extend(flatten(n.get("children", []), path + [n["pageName"]]))
        elif n["type"] == "Wireframe":
            result.append({"id": n["id"], "name": n["pageName"], "path": path})
            result.extend(flatten(n.get("children", []), path + [n["pageName"]]))
    return result

pages = flatten(sitemap["rootNodes"])

# 持久化页面列表，供断点续接
shot_dir = f"/tmp/pmdaniu_{doc_name}"
os.makedirs(shot_dir, exist_ok=True)
with open(f"{shot_dir}/pages.json", "w") as f:
    json.dump(pages, f, ensure_ascii=False, indent=2)
```

### Step 4：逐页截图（含横向+纵向滚动 + 外嵌文档检测）

```python
import time, urllib.parse, re as _re

for i, page in enumerate(pages):
    encoded = urllib.parse.quote(page["name"])
    nav_js = 'window.location.hash = "#id=%s&p=%s"; "ok"' % (page["id"], encoded)
    subprocess.run(
        'curl -s -X POST "http://localhost:3456/eval?target=%s" -d \'%s\'' % (target, nav_js),
        shell=True
    )
    time.sleep(2)

    # ── 1. 检测是否有外嵌文档（飞书、腾讯文档等） ────────────────────────
    fetch_js = 'fetch("files/%s/data.js").then(r=>r.text()).then(t=>t).catch(()=>"")' % encoded
    data_js_raw = eval_js(target, fetch_js)
    iframes = []
    if data_js_raw:
        urls = _re.findall(r'https?://[^\s\'"<>]+', data_js_raw)
        iframes = [u for u in urls if any(d in u for d in ["feishu.cn", "larksuite.com", "docs.qq.com", "shimo.im"])]

    if iframes:
        for iframe_url in iframes:
            print("  -> embed: %s" % iframe_url)
            safe_url = urllib.parse.quote(iframe_url, safe=":/")
            embed_raw = subprocess.run('curl -s "http://localhost:3456/new?url=%s"' % safe_url,
                shell=True, capture_output=True, text=True).stdout
            embed_target = json.loads(embed_raw).get("targetId")
            if not embed_target: continue
            time.sleep(3)
            _screenshot_fullpage(embed_target, "%s/page_%02d_embed" % (shot_dir, i), eval_js, is_axure=False)
            raw_text = eval_js(embed_target, 'document.body.innerText')
            with open("%s/page_%02d_embed_text.txt" % (shot_dir, i), "w") as f:
                f.write(raw_text)
            subprocess.run('curl -s "http://localhost:3456/close?target=%s"' % embed_target, shell=True)
        subprocess.run('curl -s "http://localhost:3456/screenshot?target=%s&file=%s/page_%02d_frame.png"' % (target, shot_dir, i), shell=True)
    else:
        # ── 2. 普通页面：全尺寸截图（横向+纵向自动处理） ───────────────
        _screenshot_fullpage(target, "%s/page_%02d" % (shot_dir, i), eval_js, is_axure=True)

    print("  OK page_%02d %s" % (i, page["name"]))


def _screenshot_fullpage(target_id, path_prefix, eval_js_fn, is_axure=True):
    """对单个 tab 做完整截图，自动处理：
    
    1. **横向超出**（Axure 无限画布）：Axure 用 #base 绝对定位元素撑开画布，
       实际可滚动容器是 mainFrame 内的 documentElement（html 元素）。
       横向 scrollWidth > clientWidth 时，分段左→右截图，20% 重叠。
       
    2. **纵向超出**（长页面）：纵向 scrollHeight > 视口时，分段上→下截图，20% 重叠。
       
    3. **飞书虚拟滚动**：飞书文档用 .bear-web-x-container 做滚动容器，
       window.scrollTo / CDP scroll 均无效，必须用 el.scrollTop。
    
    组合策略：先横向分 N 列，每列内再纵向分 M 行 → 输出 N×M 张图。
    文件命名：{prefix}_c{col}r{row}.png（单屏时简化为 {prefix}.png）
    """
    if is_axure:
        # Axure 页面：在 mainFrame iframe 内操作
        dims_js = ('(function(){'
            'var mf=document.getElementById("mainFrame");'
            'var doc=mf?mf.contentDocument:document;'
            'var html=doc.documentElement;'
            'var body=doc.body;'
            'var feishu=doc.querySelector(".bear-web-x-container");'
            'return JSON.stringify({'
            '  sw:html.scrollWidth,cw:html.clientWidth,'
            '  sh:feishu?feishu.scrollHeight:body.scrollHeight,'
            '  vh:feishu?feishu.clientHeight:window.innerHeight,'
            '  isF:!!feishu'
            '});'
            '})()')
    else:
        # 普通页面（飞书等外嵌文档）
        dims_js = ('(function(){'
            'var feishu=document.querySelector(".bear-web-x-container");'
            'return JSON.stringify({'
            '  sw:document.documentElement.scrollWidth,'
            '  cw:document.documentElement.clientWidth,'
            '  sh:feishu?feishu.scrollHeight:document.body.scrollHeight,'
            '  vh:feishu?feishu.clientHeight:window.innerHeight,'
            '  isF:!!feishu'
            '});'
            '})()')

    dims = json.loads(eval_js_fn(target_id, dims_js))
    sw, cw = dims["sw"], dims["cw"]
    sh, vh = dims["sh"], dims["vh"]
    is_f = dims.get("isF", False)

    def _scroll_xy(x, y):
        if is_axure:
            if is_f:
                eval_js_fn(target_id,
                    '(function(){'
                    'var mf=document.getElementById("mainFrame");'
                    'var doc=mf.contentDocument;'
                    'doc.querySelector(".bear-web-x-container").scrollTop=%d;'
                    'doc.documentElement.scrollLeft=%d;return"ok"})()' % (y, x))
            else:
                eval_js_fn(target_id,
                    '(function(){'
                    'var mf=document.getElementById("mainFrame");'
                    'var doc=mf.contentDocument;'
                    'doc.documentElement.scrollLeft=%d;window.scrollTo(0,%d);return"ok"})()' % (x, y))
        else:
            if is_f:
                eval_js_fn(target_id,
                    'document.querySelector(".bear-web-x-container").scrollTop=%d;document.documentElement.scrollLeft=%d;"ok"' % (y, x))
            else:
                eval_js_fn(target_id,
                    'document.documentElement.scrollLeft=%d;window.scrollTo(0,%d);"ok"' % (x, y))

    need_h = sw > cw * 1.15   # 需要横向分段
    need_v = sh > vh * 1.15   # 需要纵向分段

    if not need_h and not need_v:
        # 单屏，直接截
        subprocess.run('curl -s "http://localhost:3456/screenshot?target=%s&file=%s.png"' % (target_id, path_prefix), shell=True)
        return

    # 计算分段参数
    h_step = int(cw * 0.8) if need_h else cw      # 横向步长（80%视口宽度，20%重叠）
    v_step = int(vh * 0.8) if need_v else vh       # 纵向步长（80%视口高度，20%重叠）
    h_max = sw - cw if need_h else 0               # 横向最大滚动距离
    v_max = sh - vh if need_v else 0               # 纵向最大滚动距离

    col, row = 0, 0
    y_pos = 0
    while y_pos <= v_max:
        x_pos = 0
        col = 0
        while x_pos <= h_max:
            _scroll_xy(x_pos, y_pos)
            time.sleep(0.4)

            if not need_h and not need_v:
                fname = "%s.png" % path_prefix
            elif not need_h:
                fname = "%s_s%02d.png" % (path_prefix, row)
            elif not need_v:
                fname = "%s_c%02d.png" % (path_prefix, col)
            else:
                fname = "%s_c%02dr%02d.png" % (path_prefix, col, row)

            subprocess.run('curl -s "http://localhost:3456/screenshot?target=%s&file=%s"' % (target_id, fname), shell=True)
            x_pos += h_step
            col += 1
        y_pos += v_step
        row += 1

    # 复位到左上角
    _scroll_xy(0, 0)
```

#### 横向截图原理（重要！）

**问题**：Axure 支持无限画布，原型内容可以远超浏览器视口宽度。CDP screenshot 只截视口大小（~1600px），右边的内容（便利贴说明、手机预览、注释等）全部丢失。

**根因**：Axure RP 生成的 viewer 中：
- `#base` div 使用 `position: absolute`，宽度可达 5000px+
- 实际的**水平滚动容器**是 mainFrame iframe 内的 **`documentElement`（即 `<html>` 元素）**
- `html.scrollWidth` 可达 4000-5000px，而 `html.clientWidth` 只有 ~1600px
- **body.scrollLeft 无效**，必须操作 **`documentElement.scrollLeft`**

**修复方案**：截图前检测 `scrollWidth vs clientWidth`，超出时按视口 80% 步长横向分段截图（与纵向滚动逻辑对称）。文件名加 `_c00`/`_c01` 后缀标识列。

**已验证数据**（创建/编辑运费模板页面）：
| 维度 | 像素 |
|------|------|
| html.scrollWidth | 4992px |
| html.clientWidth | 1616px |
| body.scrollWidth | 5134px |
| #base 宽度 | 5134px |
| 超出比例 | **3.1 倍视口宽度** |

不修复的话右边 ~70% 内容丢失。

### Step 5：Vision 理解 → Markdown

**直接在主 Agent 串行处理，用 `Read` 工具读取每张 PNG。** 不需要子 Agent，不需要外部 API。

Step 4 截图后，先用 `ls {shot_dir}/` 列出所有文件，按文件名顺序处理每个 page：

**文件命名规则与读取方式：**

| 文件名模式 | 含义 | 处理方式 |
|---|---|---|
| `page_NN.png` | 单屏普通页面（不超视口） | 读一张，直接输出 |
| `page_NN_s00.png`, `page_NN_s01.png`… | 纵向长页面分段截图（从上到下） | **按行优先**依次读完所有段，合并为一个 Markdown section |
| `page_NN_c00.png`, `page_NN_c01.png`… | 横向宽页面分段截图（从左到右） | **按列优先**依次读完所有段，合并为一个 Markdown section |
| `page_NN_c00r00.png`, `page_NN_c01r00.png`… | 横向+纵向都超出的大画布 | **先行后列**（先读完第 0 行的所有列，再读第 1 行），合并为一个 section |
| `page_NN_frame.png` | 外嵌页面的 Axure 框架层 | 读取，仅提取框架中非嵌入内容 |
| `page_NN_embed_s00.png`… | 嵌入的外链文档分段 | 依次读，作为该页的主体内容输出 |

**理解规则：**

> ⚠️ **忠实转录三原则（最高优先级）**：
>
> **1. 看到什么写什么，绝不编造**
> - **纯文字段落**（背景说明、需求描述、飞书文档正文）：**必须逐字转录，不得意译、压缩、改写**。
> - **便利贴/粉色标注**：转录原文标注内容，不要改写成"注意事项"。
> - **线框图区域**（控件布局、字段标注）：无法逐字转录，描述可见信息，不推断未显示的内容。
>
> **2. 不确定就标注，不猜**
> - 截图模糊/截断/看不清的内容：标注为 `[模糊]` 或 `[截断]`，绝不补全或推断。
> - URL 里有字符看不清：写 `[?]` 占位，不基于"看起来像"来猜。
> - 示例：`https://www.tapd.cn/tapd_fe/[模糊]/story/detail/[模糊]`
>
> **3. URL/数字/ID 用 innerText 校验，不纯靠 Vision**
> - Vision 对结构和布局准，但对数字/链接容易出错（会"重建"相似内容）。
> - 对于嵌入的飞书文档 tab，截图后额外执行一次 innerText 提取保存原始文本：
>   ```python
>   raw_text = eval_js(embed_target, 'document.body.innerText')
>   with open(f"{shot_dir}/page_{i:02d}_embed_text.txt", "w") as f:
>       f.write(raw_text)
>   ```
> - Vision 理解时优先核对 innerText 中的 URL 和数字，不一致时以 innerText 为准。

- 线框图中的表格字段 → Markdown 表格（含示例数据行）
- 便利贴/箭头/粉色标注 → 逐字转录标注内容。**判断归属**：看箭头指向 + 文字关键词匹配目标控件；能确定归属的挂到对应控件下面作为子节，不确定的用 `> [连线关联]` 标注后单独成节
- 流程图页 → ASCII 流程图 + 节点文字
- 飞书/腾讯文档嵌入内容 → **逐字转录**，按原有层级还原为 Markdown，保留标题/表格/列表
- 去掉左侧菜单、顶部面包屑、Axure 导航栏等 UI 框架
- 分段截图有重叠区域，**识别重复内容后去重**，不要把同一段文字输出两遍

**何时考虑并行子 Agent**：页面数量 > 15 且内容复杂时，可考虑分批派发子 Agent。对于普通文档（≤15 页），主 Agent 串行处理更快（子 Agent 有冷启动开销）。

### Step 6：组装输出文件

按 sitemap 层级拼接所有页面的 Markdown：

```python
# 层级规则：
# 文档根页 → H1（文件标题）
# 一级文件夹 → H2
# 二级文件夹 → H3
# Wireframe 页 → 比其父级深一层（最浅 H2）

doc_title = pages[0]["name"] if pages else "未命名文档"
outpath = f"outputs/产品大牛文档/{doc_title}.md"
```

### Step 7：清理

```bash
curl -s "http://localhost:3456/close?target=$TARGET"
rm -rf /tmp/pmdaniu_{docname}/   # 可选，保留供复查
```

## 输出规范

- 文件路径：`outputs/产品大牛文档/{文档名}.md`
- 标题层级：文档名 → H1，文件夹 → H2/H3，页面内容从 H3/H4 开始
- 每页内容包含：功能说明正文、Markdown 表格（如有）、注意事项（来自便利贴/箭头标注）
- 流程图页：用 ASCII 流程图 + 节点文字描述，注明"此页为流程图"
- 表格字段：列名对应原型中的字段名，示例数据行用原型中的测试数据

## 已知局限

- **纯图片页**：无文字内容的图片展示页无法有效转换，Vision 会描述图片但无法还原业务语义，建议手动补充
- **流程图箭头关系**：Vision 能识别节点文字但无法精确重建箭头拓扑，仅输出节点顺序
- **嵌入文档需登录**：若飞书/腾讯文档是私有文档，需要 Chrome 中已登录对应账号，否则 embed tab 打开后是登录页
- **便利贴/标注的连线归属（2026-06-05 确认）**：Axure 原型中便利贴（黄色便签）常通过箭头连线指向它所解释的控件。Vision **能看到箭头的存在和大致方向**（从 A 指向 B），但**无法精确重建箭头拓扑关系**——尤其是多条交叉箭头时。实际处理中依赖两个信号推断归属：（1）**位置邻近性**（便利贴就在控件旁边）；（2）**文字内容关键词**（便利贴里提到"按金额"→ 关联到「按金额」选项卡）。大多数情况下够用，但复杂布局可能误关联。遇到不确定的归属时，在 Markdown 中用 `> [连线关联]` 标注说明这是根据箭头+关键词推断的，而非精确解析。

## 站点经验

- **CDP Proxy 懒连接**：Proxy 启动后 `chromePort: null`，必须先调 `/targets` 才真正建立 Chrome 连接。跳过这步会导致后续所有操作超时，用户看到"连接超时"却不知道原因。Step 1 已内置此修复。
- **长页面内容截断**：Axure viewer 页面超出视口的内容单张截图会丢失。Step 4 已内置滚动截图（按视口高度分段，20% 重叠），Vision 读取时需识别重叠区域去重。
- **宽画布内容截断（2026-06-05 修复）**：Axure 无限画布可远超视口宽度（实测 3 倍+），CDP screenshot 只截视口，右边 70% 内容丢失（便利贴、手机预览、注释等）。根因：Axure 用 `#base` 绝对定位撑开画布，水平滚动容器是 mainFrame 内的 `documentElement`（非 body）。修复：Step 4 的 `_screenshot_fullpage()` 同时检测横向和纵向超出，自动执行 N×M 分段截图（先行后列，20% 重叠），文件名 `_cNNrMM` 标识位置。
- **外嵌飞书/腾讯文档**：Axure 页面可能嵌入 iframe 展示外部文档。Step 4 会自动检测 `iframe[src]`，在新 tab 打开外链并做滚动截图，保证嵌入内容不丢失。
- jump 页每次通过 URL 参数导航都会触发，hash 导航不触发
- `$axure.document.sitemap` 在进入 viewer 后立即可读
- `mainFrame` iframe 的 `contentDocument.innerText` 可提取文字，但质量差（已废弃）
- 不要用 `$axure.player.navigateTo()` 和 `$axure.loadDocument()`，两者均报错
- subprocess 中的 curl 命令必须用 `shell=True`，URL 含中文/特殊字符时 list 形式会导致截图文件不保存
- 外部 Vision API（Zhipu 兼容端点等）会严重幻觉，输出与实际内容完全不符，绝对不用
- **Axure 嵌入文档检测**：Axure mainFrame 的 `src="about:blank"`（内容由 JS 注入），`querySelectorAll("iframe[src]")` 检测不到嵌入文档。正确方式：用 `fetch("files/{pageName}/data.js")` 读取 Axure 页面数据文件，从中正则匹配 HTTP URL，再过滤出飞书/腾讯文档等域名
- **飞书文档虚拟滚动**：飞书文档使用 `.bear-web-x-container` 作为滚动容器（scrollHeight 可达数千 px），`window.scrollTo()` 和 CDP `/scroll` 均无效。必须用 `el.scrollTop = pos` 通过 JS eval 直接设置容器的滚动位置。`_screenshot_with_scroll` 函数已内置自动检测和处理
- 截图目录路径必须用 ASCII，含中文的路径（如 `/tmp/pmdaniu_易宝/`）会导致截图无法保存