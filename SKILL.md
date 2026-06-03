# 产品大牛文档 → Markdown

将产品大牛（pmdaniu.com）上的 Axure 原型文档转成结构化 Markdown，适合 RAG 检索和团队 AI 共享。

## 前置条件

- **Chrome 已登录产品大牛**（pmdaniu.com）
- **web-access skill 已安装**（提供 CDP Proxy 能力）
  - 安装地址：https://github.com/eze-is/web-access
  - 安装完成后确保 CDP Proxy 运行在 3456 端口
- **Python 3 可用**（macOS/Linux 通常自带；Windows 从 https://python.org 安装）
- **当前模型具备视觉能力**（能理解截图图片）
- 输出目录：`outputs/产品大牛文档/`（自动创建）

> Step 0 会自动检测以上依赖，缺什么给你说清楚怎么装。

---

## 核心技术细节

### Vision 方式

**使用当前主模型的视觉能力直接理解截图。** 不要在主模型之外额外调用第三方 Vision API——额外接口会引入幻觉风险。各运行时传图方式不同（Claude 用 Read 工具读本地文件，其他运行时有各自方式），按当前环境执行即可，Skill 不做限定。

### 平台特征

- 文档类型：Axure RP 原型，本质是**线框图 + 便利贴标注**，不是纯文字 PRD
- 纯文字提取（innerText）质量差，必须用 **Vision 截图理解**
- 每个文档有多个子页面，通过 `$axure.document.sitemap` 获取完整树结构
- 页面间导航：**修改 `window.location.hash`** 为 `#id=<pageId>&p=<encodedPageName>`，无需重新加载
- 每次打开新 URL 都有反诈 jump 页，需点击 `.J_jump` 按钮；hash 导航不触发 jump 页

### URL 结构

```
短链：https://u.pmdaniu.com/XXXXX
长链：https://www.pmdaniu.com/clouds/{uid}/{hash}/start.html?id={pageId}&p={encodedName}
```

### CDP 操作关键点（Python urllib，跨平台）

```python
import urllib.request, urllib.parse, json

PROXY = "http://localhost:3456"

# 1. 打开链接
res = json.loads(urllib.request.urlopen(f"{PROXY}/new?url={urllib.parse.quote(url, safe=':/?=&%')}").read())
target_id = res["targetId"]

# 2. 点击 jump 页（等 3 秒再点）
req = urllib.request.Request(f"{PROXY}/click?target={target_id}", data=b".J_jump", method="POST")
urllib.request.urlopen(req)

# 3. 获取 sitemap
req = urllib.request.Request(f"{PROXY}/eval?target={target_id}",
      data=b"JSON.stringify($axure.document.sitemap)", method="POST")
result = json.loads(urllib.request.urlopen(req).read()).get("value", "")

# 4. hash 导航到指定页（无 jump 页触发）
js = f'window.location.hash = "#id={page_id}&p={encoded_name}"; "ok"'
req = urllib.request.Request(f"{PROXY}/eval?target={target_id}", data=js.encode(), method="POST")
urllib.request.urlopen(req)  # 等待 2 秒加载

# 5. 截图（file 用系统临时目录绝对路径）
params = urllib.parse.urlencode({"target": target_id, "file": file_path})
urllib.request.urlopen(f"{PROXY}/screenshot?{params}")

# 6. 关闭 tab
urllib.request.urlopen(f"{PROXY}/close?target={target_id}")
```

---

## 执行流程 — MANDATORY

### Step 0：环境自检

```python
import sys, urllib.request, json

print("=== 环境自检 ===")
print(f"✓ Python {sys.version.split()[0]}")

try:
    with urllib.request.urlopen("http://localhost:3456/targets", timeout=5) as resp:
        targets = json.loads(resp.read())
        COUNT = len(targets)
        print(f"✓ CDP Proxy 已运行（当前 {COUNT} 个 tab）")
except Exception as e:
    print("""
✗ CDP Proxy 未响应（localhost:3456）

  需要安装并启动 web-access skill：
  → https://github.com/eze-is/web-access

  安装完成并启动 CDP Proxy 后，重新运行此 skill。
""")
    sys.exit(1)

print("=== 环境自检通过 ===")
```

### Step 1：验证 Chrome 连接

```python
if COUNT >= 0:
    print(f"✓ Chrome 已连接，当前有 {COUNT} 个 tab")
else:
    print("""✗ 无法连接 Chrome，请检查：
  1. 确认 Chrome 已打开
  2. 地址栏输入：chrome://inspect/#remote-debugging
  3. 勾选「Allow remote debugging for this browser instance」
  4. 若刚勾选，重启 Chrome 后重试""")
    sys.exit(1)
```

### CDP 工具函数（后续各步通用，在 Step 2 前定义）

```python
import urllib.request, urllib.parse, json, time, os, tempfile, re

PROXY = "http://localhost:3456"

def cdp_get(path):
    with urllib.request.urlopen(f"{PROXY}{path}", timeout=30) as r:
        return json.loads(r.read())

def cdp_post(path, data):
    body = data.encode("utf-8") if isinstance(data, str) else data
    req = urllib.request.Request(f"{PROXY}{path}", data=body, method="POST")
    with urllib.request.urlopen(req, timeout=30) as r:
        return json.loads(r.read())

def eval_js(target, js):
    res = cdp_post(f"/eval?target={target}", js)
    return res.get("value", "")

def screenshot(target, file_path):
    # 统一转正斜杠，兼容 Windows 路径
    fp = str(file_path).replace("\\", "/")
    params = urllib.parse.urlencode({"target": target, "file": fp})
    cdp_get(f"/screenshot?{params}")

def new_tab(url):
    return cdp_get(f"/new?url={urllib.parse.quote(url, safe=':/?=&%')}")

def close_tab(target):
    cdp_get(f"/close?target={target}")
```

### Step 2：打开文档，处理 jump 页

```python
TARGET = new_tab("<用户给的链接>")["targetId"]
time.sleep(3)
cdp_post(f"/click?target={TARGET}", ".J_jump")
time.sleep(4)
# 确认已进入 Axure viewer（title 变为文档名）
info = cdp_get(f"/info?target={TARGET}")
print(info)
```

### Step 3：读取 sitemap，扁平化页面列表

```python
sitemap_raw = eval_js(TARGET, "JSON.stringify($axure.document.sitemap)")
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

# 截图目录：系统临时目录（跨平台）+ ASCII 文件名（含中文路径截图无法保存）
doc_name = re.sub(r'[^\x00-\x7F]', '_', pages[0]["name"] if pages else "doc")
shot_dir = os.path.join(tempfile.gettempdir(), f"pmdaniu_{doc_name}")
os.makedirs(shot_dir, exist_ok=True)
with open(os.path.join(shot_dir, "pages.json"), "w", encoding="utf-8") as f:
    json.dump(pages, f, ensure_ascii=False, indent=2)
```

### Step 4：逐页截图（含滚动截图 + 外嵌文档检测）

```python
def _screenshot_with_scroll(target_id, path_prefix):
    """对单个 tab 做滚动截图。超出一屏时分段拍，20% 重叠避免截断。
    自动处理飞书文档虚拟滚动容器（.bear-web-x-container）。
    """
    feishu_dims = eval_js(target_id,
        'JSON.stringify((()=>{'
        'const el=document.querySelector(".bear-web-x-container");'
        'return el?{sh:el.scrollHeight,vh:el.clientHeight,is_feishu:true}:'
        '{sh:document.body.scrollHeight,vh:window.innerHeight,is_feishu:false};'
        '})()'
        ')'
    )
    dims = json.loads(feishu_dims)
    scroll_h, view_h, is_feishu = dims["sh"], dims["vh"], dims.get("is_feishu", False)

    def _scroll_to(pos):
        if is_feishu:
            eval_js(target_id,
                f'document.querySelector(".bear-web-x-container").scrollTop={pos}')
        else:
            eval_js(target_id, f'window.scrollTo(0, {pos})')

    if scroll_h <= view_h * 1.15:
        screenshot(target_id, f"{path_prefix}.png")
    else:
        step = int(view_h * 0.8)
        pos, seg = 0, 0
        while pos < scroll_h:
            _scroll_to(pos)
            time.sleep(0.4)
            screenshot(target_id, f"{path_prefix}_s{seg:02d}.png")
            pos += step
            seg += 1
        _scroll_to(0)  # 复位


for i, page in enumerate(pages):
    encoded = urllib.parse.quote(page["name"])
    eval_js(TARGET, f'window.location.hash = "#id={page["id"]}&p={encoded}"; "ok"')
    time.sleep(2)

    # ── 检测是否有外嵌文档（飞书、腾讯文档等） ──────────────────────────
    # querySelectorAll("iframe[src]") 检测不到（Axure mainFrame src="about:blank"）
    # 正确方式：fetch data.js 文件从中正则匹配外链 URL
    encoded_name = urllib.parse.quote(page["name"])
    data_js_raw = eval_js(TARGET,
        f'fetch("files/{encoded_name}/data.js").then(r=>r.text()).then(t=>t).catch(()=>"")'
    )
    iframes = []
    if data_js_raw:
        urls = re.findall(r'https?://[^\s\'"<>]+', data_js_raw)
        iframes = [u for u in urls if any(d in u for d in
                   ["feishu.cn", "larksuite.com", "docs.qq.com", "shimo.im"])]

    if iframes:
        for iframe_url in iframes:
            print(f"  → 检测到嵌入外链：{iframe_url}")
            embed_target = new_tab(iframe_url).get("targetId")
            if not embed_target:
                continue
            time.sleep(3)

            _screenshot_with_scroll(embed_target, os.path.join(shot_dir, f"page_{i:02d}_embed"))

            # 额外保存 innerText 供 Vision 核对 URL 和数字
            raw_text = eval_js(embed_target, 'document.body.innerText')
            with open(os.path.join(shot_dir, f"page_{i:02d}_embed_text.txt"), "w", encoding="utf-8") as f:
                f.write(raw_text)

            close_tab(embed_target)

        # 外嵌页面本身也截一张（显示 Axure 主框架）
        screenshot(TARGET, os.path.join(shot_dir, f"page_{i:02d}_frame.png"))
    else:
        _screenshot_with_scroll(TARGET, os.path.join(shot_dir, f"page_{i:02d}"))

    print(f"  ✓ page_{i:02d} {page['name']}")
```

### Step 5：视觉模型理解 → Markdown

**将截图 PNG 逐一传给当前视觉模型理解。** 使用主模型自身的视觉能力，不要额外调用第三方 Vision 接口（会引入幻觉风险）。各运行时传图片的具体方式不同，按当前环境执行即可。

先用 Python 列出所有截图文件，按文件名排序处理：

```python
import os
files = sorted(f for f in os.listdir(shot_dir) if f.endswith(".png"))
```

**文件命名规则：**

| 文件名模式 | 含义 | 处理方式 |
|---|---|---|
| `page_NN.png` | 单屏普通页面 | 读一张，直接输出 |
| `page_NN_s00.png`, `page_NN_s01.png`… | 长页面分段（从上到下） | 依次读完所有段，合并为一个 section，去重重叠内容 |
| `page_NN_frame.png` | 外嵌页面的 Axure 框架层 | 读取，仅提取框架中非嵌入内容 |
| `page_NN_embed_s00.png`… | 嵌入的外链文档分段 | 依次读，作为该页的主体内容输出 |

**理解规则：**

> ⚠️ **忠实转录三原则（最高优先级）**
>
> **1. 看到什么写什么，绝不编造**
> - **纯文字段落**（背景说明、需求描述、飞书文档正文）：**必须逐字转录，不得意译、压缩、改写**
> - **便利贴/粉色标注**：转录原文，不改写成"注意事项"
> - **线框图区域**（控件布局、字段）：描述可见信息，不推断未显示的内容
>
> **2. 不确定就标注，不猜**
> - 截图模糊/截断：标注为 `[模糊]` 或 `[截断]`，绝不补全或推断
> - URL 中有字符看不清：写 `[?]` 占位，不基于"看起来像"来猜
>
> **3. URL/数字/ID 用 innerText 校验**
> - 嵌入飞书文档已保存 `_embed_text.txt`，Vision 理解时对照其中的 URL 和数字
> - 不一致时以 innerText 为准

- 线框图中的表格字段 → Markdown 表格（含示例数据行）
- 便利贴/箭头/粉色标注 → 逐字转录，放入「注意事项」或「交互说明」小节
- 流程图页 → ASCII 流程图 + 节点文字，注明"此页为流程图"
- 飞书/腾讯文档嵌入内容 → **逐字转录**，按原有层级还原 Markdown（标题/表格/列表）
- 去掉左侧菜单、顶部面包屑、Axure 导航栏等 UI 框架
- 分段截图有重叠区域，**识别重复内容后去重**，不要把同一段文字输出两遍

页面数量 > 15 且内容复杂时，可考虑分批派发子 Agent 并行处理以提速。

### Step 6：组装输出文件

```python
import os
doc_title = pages[0]["name"] if pages else "未命名文档"
out_dir = os.path.join("outputs", "产品大牛文档")
os.makedirs(out_dir, exist_ok=True)
outpath = os.path.join(out_dir, f"{doc_title}.md")

# 层级规则：
# 文档根页   → H1（文件标题）
# 一级文件夹 → H2
# 二级文件夹 → H3
# Wireframe 页 → 比其父级深一层（最浅 H2）
```

### Step 7：清理

```python
close_tab(TARGET)
# 截图目录可保留供复查，确认无误后手动删：
# import shutil; shutil.rmtree(shot_dir)
```

---

## 输出规范

- 文件路径：`outputs/产品大牛文档/{文档名}.md`
- 标题层级：文档名 → H1，文件夹 → H2/H3，页面内容从 H3/H4 开始
- 每页内容包含：功能说明正文、Markdown 表格（如有）、注意事项（来自便利贴/标注）
- 流程图页：ASCII 流程图 + 节点文字，注明"此页为流程图"
- 表格字段：列名对应原型中的字段名，示例数据行用原型中的测试数据

---

## 已知局限

- **纯图片页**：无文字内容的图片页无法有效转换，Vision 会描述图片但无法还原业务语义，建议手动补充
- **流程图箭头关系**：Vision 能识别节点文字但无法精确重建箭头拓扑，仅输出节点顺序
- **嵌入文档需登录**：若飞书/腾讯文档是私有文档，需 Chrome 中已登录对应账号，否则 embed tab 打开后是登录页

---

## 站点经验（踩过的坑）

- **CDP Proxy 懒连接**：Proxy 启动后 `chromePort: null`，必须先调 `/targets` 才真正建立 Chrome 连接。Step 0 已内置此修复
- **长页面内容截断**：Step 4 已内置滚动截图（按视口高度分段，20% 重叠）
- **飞书文档虚拟滚动**：飞书用 `.bear-web-x-container` 作为滚动容器，`window.scrollTo()` 无效，必须用 `el.scrollTop = pos`
- **Axure 嵌入文档检测**：`querySelectorAll("iframe[src]")` 检测不到（mainFrame src="about:blank"），正确方式是 fetch `files/{pageName}/data.js` 正则匹配 URL
- **截图目录路径必须用 ASCII**：含中文的路径会导致截图无法保存，Step 3 已自动转换
- jump 页每次通过 URL 参数导航都会触发，hash 导航不触发
- Windows 上临时目录由 `tempfile.gettempdir()` 自动获取，不硬编码 `/tmp/`
