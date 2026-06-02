# pmdaniu-to-md

将产品大牛（pmdaniu.com）上的 Axure 原型文档转成结构化 Markdown。

适合团队把 PRD 原型文档喂给 AI 做 RAG 检索，或者整理成可协作的文档格式。

## 效果

- 线框图 + 便利贴标注 → 结构化 Markdown
- 自动处理多级页面树（文件夹 / 子页面）
- 自动识别并提取嵌入的飞书 / 腾讯文档
- 长页面自动滚动截图，不丢内容
- 输出到 `outputs/产品大牛文档/{文档名}.md`

## 前置条件

1. **MyAgents**（https://myagents.io）
2. **web-access skill**（提供 CDP Proxy，驱动 Chrome）
   ```bash
   git clone https://github.com/eze-is/web-access ~/.myagents/skills/web-access
   ```
3. **Chrome** 已打开并登录产品大牛（pmdaniu.com）
4. **python3**（macOS 自带；其他系统从 https://python.org 安装）
5. 当前使用的 AI 模型具备视觉能力（能理解截图图片）

## 安装

```bash
git clone https://github.com/monsignorlaw1015/pmdaniu-to-md ~/.myagents/skills/pmdaniu-to-md
```

然后在你的工作区 `.claude/skills/` 下建软链接：

```bash
ln -sf ~/.myagents/skills/pmdaniu-to-md /path/to/your/workspace/.claude/skills/pmdaniu-to-md
```

## 使用

在 MyAgents 对话框里直接说：

> 帮我把这个产品大牛文档转成 Markdown：https://u.pmdaniu.com/XXXXX

Skill 会自动：
1. 检测环境（CDP Proxy、python3），缺什么告诉你怎么装
2. 打开文档，遍历所有子页面
3. 截图 → 视觉模型理解 → 输出 Markdown

## 注意

- 私有飞书 / 腾讯文档需要 Chrome 里已登录对应账号
- 纯图片页（无文字）无法有效转换，需手动补充
- 流程图箭头拓扑无法精确重建，输出节点文字 + ASCII 示意
