# 趣运运营二组 组织共享配置

本仓库是组织级别的共享配置中心，存放所有仓库通用的 GitHub Actions 工作流和团队文档。

> 组织：`quyun-ops-2`（趣运运营二组）· 通知通道：钉钉群机器人（加签）

---

## 目录结构

```
.github/
├── .github/workflows/
│   └── notify-dingtalk.yml    # 钉钉通知可复用工作流（所有仓库共享）
├── DINGTALK_NOTIFY_GUIDE.md   # 钉钉通知接入指南
├── review-rules.md            # AI Code Review 规则
└── README.md                  # 本文件
```

---

## 钉钉推送通知

所有仓库的钉钉通知逻辑统一维护在 `.github/workflows/notify-dingtalk.yml`。

触发事件：

- **Push**：推送代码到 master / main 分支时通知
- **PR**：PR 新建、重新打开、合并、关闭时通知

新仓库接入只需在仓库内添加一个 `.github/workflows/notify-dingtalk.yml`，内容如下：

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

详细接入说明见 [DINGTALK_NOTIFY_GUIDE.md](./DINGTALK_NOTIFY_GUIDE.md)。

---

## 组织级 Secret

在 `Settings → Secrets and variables → Actions` 配置了两个组织级 Secret（所有仓库通过
`secrets: inherit` 自动继承，无需逐个仓库配置）：

| Secret 名 | 说明 |
| --- | --- |
| `DINGTALK_WEBHOOK_URL` | 钉钉机器人 Webhook 地址（含 access_token） |
| `DINGTALK_SIGN_SECRET` | 钉钉机器人加签密钥（SEC 开头） |

---

## 注意事项

- 本仓库必须保持 **Public**，否则其他仓库无法调用可复用工作流
- 修改 `notify-dingtalk.yml` 后所有仓库自动生效，无需逐个更新
- 换钉钉群 / 换机器人时，只需更新上面两个 Secret，不用改任何工作流文件
