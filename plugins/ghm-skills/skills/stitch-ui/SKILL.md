---
name: stitch-ui
description: 基于用户需求的 UI 设计 skill。使用 Stitch MCP 生成 UI 设计稿，两阶段交付（先截图审阅，用户确认后生成代码并通过 Playwright 验证）。只要涉及 Stitch MCP 的使用，都必须加载此 skill。包括但不限于：UI 设计、页面设计、界面原型、设计稿生成、查看页面效果、创建 Stitch 项目、生成 Screen 等任何与 Stitch 相关的操作。
---

# Stitch UI 设计

你是一个 **UI 设计助手**，根据用户需求生成 UI 设计。使用 Stitch 生成界面，分两阶段交付：先截图审阅，用户满意后生成代码并通过 Playwright 验证质量。

## 概览

每次设计请求遵循以下流程：

```
用户描述需求 → 生成截图 → 呈现给用户
                            ├─ 不满意 → 修改后重新生成
                            └─ 满意 → 生成代码 → Playwright 验证 → 交付
```

## 前置条件

**必需：**

* Stitch MCP Server 访问权限（工具前缀为 `mcp__stitch__`）
* Playwright（第二阶段验证）

**可选：**

* `.stitch/DESIGN.md` 文件，定义项目的视觉风格和设计规范

## 重要注意事项

* `generate_screen_from_text` 通常需要 3-15 分钟完成，**超时不代表失败**——后台仍在处理。超时后应每隔 2-3 分钟使用 `list_screens` 轮询检查，在 15 分钟内 `list_screens` 返回空列表是正常的，耐心等待即可
* 提示词长度建议控制在 3000 字符以内，超过 5000 字符容易导致组件遗漏
* 始终使用最新模型 `GEMINI_3_1_PRO`（通过 `modelId` 参数指定）
* `get_screen` 的 `name` 参数格式为 `projects/{projectId}/screens/{screenId}`
* Stitch 生成的是 Web UI，用于桌面应用（如 Qt）时作为视觉参考，需手动将设计语言转换为对应框架的实现

## 核心概念

### stitch.json — 项目状态

`.stitch/stitch.json` 是项目的唯一事实来源，追踪 Stitch 项目 ID、所有已生成的 Screen、生成时使用的提示词及当前状态。

```json
{
  "projectId": "stitch-xxx",
  "screens": [
    {
      "name": "login",
      "status": "draft",
      "prompt": "生成该 Screen 时使用的完整提示词...",
      "screenId": "stitch-screen-xxx",
      "files": {
        "screenshot": ".stitch/queue/login.png"
      }
    }
  ]
}
```

**Screen 状态：**

| 状态 | 含义 |
| --- | --- |
| `draft` | 截图已生成，等待用户审阅 |
| `approved` | 用户已确认设计 |
| `delivered` | 代码已集成到项目并通过验证 |

**规则：**

* 如果 `.stitch/stitch.json` 中已有 `projectId`，绝不重复创建项目
* 必须记录每个 Screen 使用的完整提示词——这是设计历史
* 仅在对应操作确认后才更新状态

### DESIGN.md — 设计规范（可选）

如果项目有 `.stitch/DESIGN.md`，在每次 Stitch 生成时都将其中的设计规范附加到提示词中，以保持跨页面的视觉一致性。如果尚不存在，在第一个 Screen 被用户确认后，建议从该设计中提取视觉语言作为基线创建 `.stitch/DESIGN.md`。

## 执行协议

### 第一阶段：生成设计

#### 步骤 1：理解需求

解析用户的请求，必要时确认：

* 这个页面/组件的用途是什么？
* 是否有特定的布局、风格或内容要求？
* 是否需要遵循已有的 `.stitch/DESIGN.md`？

#### 步骤 2：编写提示词

编写提示词前，先阅读 `references/prompt-guide.md` 了解 Stitch 的提示词最佳实践。

组合 Stitch 生成提示词：

1. 以用户的需求描述为主体
2. 如果存在 `.stitch/DESIGN.md`，附加设计规范内容
3. 如果用户提供了布局结构，明确写入

#### 步骤 3：使用 Stitch 生成

1. **获取或创建项目**：
   * 如果 `.stitch/stitch.json` 存在且包含 `projectId`，直接使用
   * 否则调用 `[prefix]:create_project`，将 ID 保存到 `.stitch/stitch.json`
2. **生成 Screen**：调用 `[prefix]:generate_screen_from_text`，参数：
   * `projectId`：项目 ID
   * `prompt`：步骤 2 中组合的提示词
   * `deviceType`：`DESKTOP`（或按用户指定）
   * `modelId`：始终使用 `GEMINI_3_1_PRO`
3. **获取资源**：调用 `[prefix]:get_screen` 获取：
   * `screenshot.downloadUrl` — 下载并保存为 `.stitch/queue/{name}.png`
4. **更新 `.stitch/stitch.json`**：添加 Screen 条目，状态设为 `draft`，记录完整提示词和 screenId

#### 步骤 4：呈现给用户

向用户展示：

* Stitch 生成的截图（`.stitch/queue/{name}.png`）
* 简要说明生成内容

等待用户反馈：

* **通过** → 在 `.stitch/stitch.json` 中将状态更新为 `approved`，用户要求时进入第二阶段
* **需要修改** → 讨论调整方案，使用修改后的提示词返回步骤 2
* **拒绝** → 移除该 Screen 条目或标记为待重新设计

### 第二阶段：生成代码

仅在用户确认设计后执行。

#### 步骤 1：从 Stitch 获取代码并集成

对已确认的 Screen 调用 `[prefix]:get_screen` 获取：

* `htmlCode.downloadUrl` — 下载代码，集成到项目的实际页面中

#### 步骤 2：Playwright 验证代码

代码集成到项目后，使用 Playwright 在实际运行的页面上验证渲染结果：

1. **启动项目的开发服务器**（根据项目实际框架，如 `npm run dev`、`vite`、`npx serve` 等）
2. **使用 Playwright 截图**，至少覆盖桌面端（1280×800）和移动端（375×812）两个视口，保存到 `.stitch/verify/`
3. **对比验证**：将截图与 `.stitch/queue/{name}.png` 中已确认的 Stitch 截图对比
   * 渲染结果是否与设计一致
   * 是否有控制台错误
   * 元素是否被裁剪或重叠
   * 导航链接是否正确
4. **停止开发服务器**

#### 步骤 3：交付

1. 修复验证中发现的问题
2. 更新 `.stitch/stitch.json`：状态设为 `delivered`
3. 将最终结果告知用户

## 多页面项目

为同一项目设计多个页面时：

1. 所有 Screen 共享 `.stitch/stitch.json` 中的同一个 `projectId`
2. 每个 Screen 独立追踪，拥有各自的状态
3. 使用 `.stitch/DESIGN.md` 保持跨页面的视觉一致性
4. 用户可以按任意顺序处理各个 Screen——确认一些、修改另一些
5. 通过 `.stitch/stitch.json` 查看整体进度：
   ```bash
   cat .stitch/stitch.json | jq '.screens[] | {name, status}'
   ```

## 文件结构

```
project/
└── .stitch/
    ├── stitch.json         # 项目状态——唯一事实来源
    ├── DESIGN.md           # 设计规范和视觉偏好（可选）
    ├── queue/              # Stitch 生成的截图（第一阶段产出）
    │   └── {name}.png
    └── verify/             # Playwright 在实际项目页面上的截图（第二阶段）
        ├── {name}-desktop.png
        └── {name}-mobile.png
```

## 常见错误

* ❌ `.stitch/stitch.json` 中已有项目时重复创建新项目
* ❌ 用户未确认截图就直接生成代码
* ❌ 未在 `.stitch/stitch.json` 中记录提示词（丢失设计历史）
* ❌ 提示词中遗漏 `.stitch/DESIGN.md` 的设计规范

## 故障排查

| 问题 | 解决方案 |
| --- | --- |
| 生成超时 | 不要重试——用 `list_screens` 轮询检查，后台可能仍在处理 |
| 提示词过长导致组件遗漏 | 控制在 3000 字符以内，先生成整体再逐步细化 |
| Stitch 生成失败 | 检查提示词是否包含 `.stitch/DESIGN.md` 中的设计规范 |
| 跨页面风格不一致 | 确保每次生成都使用了 `.stitch/DESIGN.md` |
| 验证发现布局问题 | 调整提示词后重新生成，或在呈现时注明问题 |
| Screen 状态混乱 | 查看 `.stitch/stitch.json`——它是唯一事实来源 |

## 附加资源

### 参考文件

- **`references/prompt-guide.md`** — Stitch 提示词编写最佳实践
