%23 Browser-Use 如何让 LLM 可靠读取网页

下面是对本项目“如何抽取、压缩并把真实页面状态交给 LLM”的要点总结，针对 HTML 过大、JS 动态渲染、Shadow DOM 与 iframe 等问题给出解决方案，并附关键源码位置。

**整体流程**
- 事件驱动取页状态：`BrowserSession.get_browser_state_summary()` 触发 DOM 看门狗并行构建 DOM 与（可选）截图（browser_use/browser/watchdogs/dom_watchdog.py:242）。
- 三源合一构树：`DomService.get_dom_tree()` 通过 CDP 同时获取 DOMSnapshot + DOM + 可访问性（AX）树，统一坐标到 CSS 像素，构建增强 DOM 树（browser_use/dom/service.py:758）。
- 生成 LLM 友好表示：`DOMTreeSerializer.serialize_accessible_elements()` 仅保留可操作/可见的上下文，输出紧凑可读清单与稳定索引（browser_use/dom/serializer/serializer.py:41 → 813）。最终由 `SerializedDOMState.llm_representation()` 产出文本（browser_use/dom/views.py:816）。
- 组装提示词：`AgentMessagePrompt` 将页面统计、标签页、紧凑 DOM 文本和可选截图组合为模型输入（browser_use/agent/prompts.py:180, 380）。

**如何解决“HTML 过大”**
- 保留关键元素：仅序列化交互元素（button/input/link/select/textarea）、可滚动容器与 iframes；跳过 script/style/base64 图片等噪声（browser_use/dom/serializer/serializer.py:813，browser_use/dom/serializer/html_serializer.py）。
- 交互判定多信号融合：标签/ARIA/AX 属性/事件属性/光标样式等综合判断（browser_use/dom/serializer/clickable_elements.py）。
- 可见性与遮挡裁剪：基于绘制顺序（paint order）剔除被遮挡/透明元素（browser_use/dom/serializer/paint_order.py）；对边界盒进行传播/包含度过滤，减少冗余（serializer 核心）。
- 属性限长与去重：属性白名单 + 值截断 + 去重，避免 token 浪费（browser_use/dom/serializer/serializer.py:1100+）。
- 最终安全截断：当交互元素文本超长时在拼装提示阶段截断（browser_use/agent/prompts.py:180）。

**如何覆盖“JS 动态渲染”的数据**
- 抓“渲染后”的真实状态：使用 CDP DOMSnapshot（含 layout、bounds、computed styles、paint order），反映 JS 渲染后的 DOM（browser_use/dom/enhanced_snapshot.py；在 service.py:303 调用）。
- 等待稳定：构建前做最小等待并通过 Performance API 观测 pending 请求，降低“半渲染”读取（browser_use/browser/watchdogs/dom_watchdog.py:242）。
- 兼容 Shadow DOM/iframe：增强树包含 shadow roots（明确 open/closed 标记）与 iframe 的 content document（html_serializer 与 serializer 路径）。
- DPI/坐标统一：计算设备像素比，所有坐标换算为 CSS 像素，保证点击/滚动定位准确（browser_use/dom/service.py:73；enhanced_snapshot 缩放逻辑）。
- 大页面安全措施：限制最大 iframe 数量/深度，防止内存/上下文爆炸（browser_use/dom/service.py:740）。

**稳定的选择器与交互**
- 稳定索引：每个交互元素以 `[*][123]<button … />` 显示，数字对应 selector_map 的 backendNodeId 键（serializer）。
- DOM 变动自愈：通过父链哈希与属性帮助在步骤间找回同一元素（browser_use/dom/views.py:750）。

**截图与视觉补充**
- `use_vision` 开启（或 auto 且包含截图）时，`AgentMessagePrompt` 会附带标签化截图（可按需等比缩放）与页面滚动统计，辅助 LLM 识别视觉结构与版式（browser_use/agent/prompts.py:380, 180）。

**结构化阅读/抽取工具**
- `extract` 工具路径：基于增强树重建“干净 HTML”（保留 shadow/iframe）→ 转换 Markdown → 轻清洗 → 可按 `start_from_char` 分段 → 交给独立的 `page_extraction_llm` 做结构化抽取（browser_use/tools/service.py:654；browser_use/dom/markdown_extractor.py；html_serializer）。

**模型与性能建议**
- 推荐模型：`ChatBrowserUse()`，在浏览器自动化准确性/速度/成本上最佳。
- 生产级性能/隐蔽性：`Browser(use_cloud=True)` 使用 Browser‑Use Cloud 托管浏览器，延迟更低且具备验证码/指纹绕过能力；需 `BROWSER_USE_API_KEY`（cloud.browser-use.com 获取）。
- 抽取专用 LLM：`page_extraction_llm` 可用小模型，因为它只负责“读页面文本”，不参与完整 Agent 推理。

**关键文件（快速索引）**
- DOM 构建：`browser_use/dom/service.py`，`browser_use/dom/enhanced_snapshot.py`
- 序列化：`browser_use/dom/serializer/serializer.py`，`paint_order.py`，`clickable_elements.py`，`html_serializer.py`
- 提示组装：`browser_use/agent/prompts.py`
- 看门狗编排：`browser_use/browser/watchdogs/dom_watchdog.py`
- 抽取工具：`browser_use/tools/service.py`，`browser_use/dom/markdown_extractor.py`

