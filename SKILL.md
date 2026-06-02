# 产品大牛文档 → Markdown

将产品大牛（pmdaniu.com）上的 Axure 原型文档转成结构化 Markdown，适合 RAG 检索和团队 AI 共享。

## 前置条件

- **Chrome 已登录产品大牛**（pmdaniu.com）
- **web-access skill 已安装**（提供 CDP Proxy 能力）
  - 安装地址：https://github.com/eze-is/web-access
  - 安装完成后确保 CDP Proxy 运行在 3456 端口
- **python3 可用**（macOS 通常自带；从 https://python.org 安装）
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

---

## 执行流程 — MANDATORY

### Step 0：环境自检

```bash
echo "=== 环境自检 ==="

# 检测 python3
if command -v python3 &>/dev/null; then
    echo "✓ python3 $(python3 --version)"
else
    echo ""
    echo "✗ 未检测到 python3"
    echo "  macOS:  brew install python3"
    echo "  其他:   https://python.org"
    exit 1
fi

# 检测 CDP Proxy（同时触发懒连接）
TARGETS=$(curl -s --max-time 5 "http://localhost:3456/targets" 2>/dev/null)
COUNT=$(echo "$TARGETS" | python3 -c "import sys,json; print(len(json.load(sys.stdin)))" 2>/dev/null)

if [ -n "$COUNT" ]; then
    echo "✓ CDP Proxy 已运行（当前 $COUNT 个 tab）"
else
    echo ""
    echo "✗ CDP Proxy 未响应（localhost:3456）"
    echo ""
    echo "  需要安装并启动 web-access skill："
    echo "  → https://github.com/eze-is/web-access"
    echo ""
    echo "  安装完成并启动 CDP Proxy 后，重新运行此 skill。"
    exit 1
fi

echo "=== 环境自检通过 ==="
```

### Step 1：验证 Chrome 连接

```bash
# CDP Proxy 已在 Step 0 触发连接，这里只做确认
if [ -n "$COUNT" ] && [ "$COUNT" -ge 0 ]; then
    echo "✓ Chrome 已连接，当前有 $COUNT 个 tab"
else
    echo "✗ 无法连接 Chrome，请检查："
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
# 确认已进入 Axure viewer（title 变为文档名）
curl -s "http://localhost:3456/info?target=$TARGET"
```

### Step 3：读取 sitemap，扁平化页面列表

```python
import subprocess, json, urllib.parse, os, re

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

# 截图目录只用 ASCII 路径（含中文路径会导致截图无法保存）
doc_name = re.sub(r'[^\x00-\x7F]', '_', pages[0]["name"] if pages else "doc")
shot_dir = f"/tmp/pmdaniu_{doc_name}"
os.makedirs(shot_dir, exist_ok=True)
with open(f"{shot_dir}/pages.json", "w") as f:
    json.dump(pages, f, ensure_ascii=False, indent=2)
```

### Step 4：逐页截图（含滚动截图 + 外嵌文档检测）

```python
import time, urllib.parse, re as _re

def _screenshot_with_scroll(target_id, path_prefix, eval_js_fn):
    """对单个 tab 做滚动截图。超出一屏时分段拍，20% 重叠避免截断。
    自动处理飞书文档虚拟滚动容器（.bear-web-x-container）。
    """
    feishu_dims = eval_js_fn(target_id,
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
            eval_js_fn(target_id,
                f'document.querySelector(".bear-web-x-container").scrollTop={pos}'
            )
        else:
            eval_js_fn(target_id, f'window.scrollTo(0, {pos})')

    if scroll_h <= view_h * 1.15:
        subprocess.run(
            f'curl -s "http://localhost:3456/screenshot?target={target_id}&file={path_prefix}.png"',
            shell=True
        )
    else:
        step = int(view_h * 0.8)
        pos, seg = 0, 0
        while pos < scroll_h:
            _scroll_to(pos)
            time.sleep(0.4)
            subprocess.run(
                f'curl -s "http://localhost:3456/screenshot?target={target_id}&file={path_prefix}_s{seg:02d}.png"',
                shell=True
            )
            pos += step
            seg += 1
        _scroll_to(0)  # 复位


for i, page in enumerate(pages):
    encoded = urllib.parse.quote(page["name"])
    subprocess.run(
        f'curl -s -X POST "http://localhost:3456/eval?target={target}" '
        f'-d \'window.location.hash = "#id={page["id"]}&p={encoded}"; "ok"\'',
        shell=True
    )
    time.sleep(2)

    # ── 检测是否有外嵌文档（飞书、腾讯文档等） ──────────────────────────
    # querySelectorAll("iframe[src]") 检测不到（Axure mainFrame src="about:blank"）
    # 正确方式：fetch data.js 文件从中正则匹配外链 URL
    encoded_name = urllib.parse.quote(page["name"])
    data_js_raw = eval_js(target,
        f'fetch("files/{encoded_name}/data.js").then(r=>r.text()).then(t=>t).catch(()=>"")'
    )
    iframes = []
    if data_js_raw:
        urls = _re.findall(r'https?://[^\s\'"<>]+', data_js_raw)
        iframes = [u for u in urls if any(d in u for d in
                   ["feishu.cn", "larksuite.com", "docs.qq.com", "shimo.im"])]

    if iframes:
        for iframe_url in iframes:
            print(f"  → 检测到嵌入外链：{iframe_url}")
            embed_raw = subprocess.run(
                f'curl -s "http://localhost:3456/new?url={urllib.parse.quote(iframe_url, safe=":/")}"',
                shell=True, capture_output=True, text=True
            ).stdout
            embed_target = json.loads(embed_raw).get("targetId")
            if not embed_target:
                continue
            time.sleep(3)

            _screenshot_with_scroll(embed_target, f"{shot_dir}/page_{i:02d}_embed", eval_js)

            # 额外保存 innerText 供 Vision 核对 URL 和数字
            raw_text = eval_js(embed_target, 'document.body.innerText')
            with open(f"{shot_dir}/page_{i:02d}_embed_text.txt", "w", encoding="utf-8") as f:
                f.write(raw_text)

            subprocess.run(f'curl -s "http://localhost:3456/close?target={embed_target}"', shell=True)

        # 外嵌页面本身也截一张（显示 Axure 主框架）
        subprocess.run(
            f'curl -s "http://localhost:3456/screenshot?target={target}&file={shot_dir}/page_{i:02d}_frame.png"',
            shell=True
        )
    else:
        _screenshot_with_scroll(target, f"{shot_dir}/page_{i:02d}", eval_js)

    print(f"  ✓ page_{i:02d} {page['name']}")
```

### Step 5：视觉模型理解 → Markdown

**将截图 PNG 逐一传给当前视觉模型理解。** 使用主模型自身的视觉能力，不要额外调用第三方 Vision 接口（会引入幻觉风险）。各运行时传图片的具体方式不同，按当前环境执行即可。

先用 `ls {shot_dir}/` 列出所有文件，按文件名顺序处理：

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
os.makedirs("outputs/产品大牛文档", exist_ok=True)
outpath = f"outputs/产品大牛文档/{doc_title}.md"

# 层级规则：
# 文档根页   → H1（文件标题）
# 一级文件夹 → H2
# 二级文件夹 → H3
# Wireframe 页 → 比其父级深一层（最浅 H2）
```

### Step 7：清理

```bash
curl -s "http://localhost:3456/close?target=$TARGET"
# 截图目录可保留供复查，确认无误后手动删：
# rm -rf /tmp/pmdaniu_{doc_name}/
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
- subprocess 中的 curl 命令必须用 `shell=True`
