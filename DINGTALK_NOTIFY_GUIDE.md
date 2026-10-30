# GitHub 提交通知接入钉钉指南

## 效果预览

每次有人往仓库 push 代码或提 PR，钉钉群会自动收到一条卡片消息：

**Push 通知**

```
📦 quyun-ops-2/你的仓库名 有新提交
─────────────────────────────
分支：main
提交人：张三

提交信息：fix: 修复登录页面跳转异常

[查看提交]
```

**PR 通知**

```
🔀 新 PR 待审查          ← 新建/重开为橙色，合并绿色，关闭红色
─────────────────────────────
仓库：quyun-ops-2/你的仓库
状态：opened（目标分支 main）

提交人：张三
PR 标题：feat: 新增用户权限管理模块

[查看 PR]
```

---

## 接入步骤

> **前提**：你的仓库必须在 `quyun-ops-2` 组织下。个人账号下的仓库无法使用组织的共享配置。

### 只需做一件事：在你的仓库里加一个文件

在仓库根目录创建以下文件（目录不存在就新建）：

**文件路径**：`.github/workflows/notify-dingtalk.yml`

**文件内容**（直接复制，不需要修改任何内容）：

```yaml
name: 钉钉推送通知

on:
  push:
    branches: [master, main]
  pull_request:
    types: [opened, closed, reopened]

jobs:
  notify:
    uses: quyun-ops-2/.github/.github/workflows/notify-dingtalk.yml@main
    with:
      event_name: ${{ github.event_name }}
      repo_name: ${{ github.repository }}
      actor: ${{ github.actor }}
      ref_name: ${{ github.ref_name }}
      commit_message: ${{ github.event.head_commit.message }}
      commit_url: ${{ github.event.head_commit.url }}
      pr_title: ${{ github.event.pull_request.title }}
      pr_url: ${{ github.event.pull_request.html_url }}
      pr_action: ${{ github.event.action }}
      pr_merged: ${{ toJson(github.event.pull_request.merged) }}
      pr_base: ${{ github.event.pull_request.base.ref }}
    secrets: inherit
```

文件提交并 push 后，下一次 push 或 PR 操作就会自动触发钉钉通知，**无需其他任何配置**。

---

## 常见问题

**Q：我需要配置钉钉的 Webhook 地址吗？**

不需要。Webhook 地址和加签密钥已经由管理员统一配置在组织级别 Secret，所有仓库通过
`secrets: inherit` 自动继承，不用单独设置。

**Q：我想监听不同的分支怎么办？**

修改文件里的 `branches` 字段，比如改成 `[develop, release]`。

**Q：PR 合并也会通知吗？**

会。PR 合并时发绿色「✅ PR 已合并」卡片；直接关闭（不合并）则是红色「❌ PR 已关闭」。

**Q：Actions 在哪里看运行记录？**

打开仓库页面 → 顶部点 **Actions** 标签 → 可以看到每次运行的日志和结果。

---

## 背后的原理（可选阅读）

### 为什么不直接用 GitHub 的 Webhook 功能？

GitHub 原生 Webhook 发送的是 GitHub 自己定义的 JSON 格式，而钉钉机器人要求消息必须
包含 `msg_type` 字段且用加签鉴权，两边格式不兼容。**GitHub Actions 充当中间的格式翻译层**：
拿到 GitHub 事件数据 → 拼成钉钉卡片 → 按加签规则算 `timestamp`/`sign` → `python3` 发送。

### 加签是怎么算的？

钉钉自定义机器人开启「加签」安全设置后，每次请求需附带：

```
timestamp = 当前毫秒时间戳
sign      = urlencode( base64( HMAC_SHA256( secret, timestamp + "\n" + secret ) ) )
```

请求 URL 追加 `&timestamp=xxx&sign=yyy` 即可通过鉴权。完整逻辑集中在组织中央仓库，
各仓库无需关心。

### 你的文件为什么只有这么短？

通知的完整逻辑（拼 JSON、区分 push/PR、加签、卡片样式）集中维护在组织的 `.github`
仓库中：

```
quyun-ops-2 组织
├── .github 仓库            ← 完整的通知逻辑在这里
│   └── .github/workflows/notify-dingtalk.yml
│
└── 你的仓库                ← 只有 15 行，调用上面的逻辑
    └── .github/workflows/notify-dingtalk.yml
```

你的文件里的 `uses: quyun-ops-2/.github/...` 就是"去调用组织中央仓库里的工作流"。
这样设计的好处：**通知格式或钉钉地址有变化，只需要改一个地方，所有仓库自动生效**。

### 钉钉地址和加签密钥是怎么传进去的？

Webhook 地址和加签密钥相当于密码，不能明文写在代码文件里。管理员已将其配置为
**组织级 Secret**，存储在 GitHub 的加密环境中。工作流里的 `secrets: inherit` 表示
"把当前仓库能访问的所有 Secret 传给被调用的工作流"。

整体流程：

```
git push
  → Actions 启动
  → 你的 .yml 调用中央工作流，传入提交信息
  → 中央工作流读取组织 Secret（Webhook + 加签密钥）
  → python3 计算加签，发送格式化的钉钉卡片
  → 钉钉群收到通知 ✅
```
