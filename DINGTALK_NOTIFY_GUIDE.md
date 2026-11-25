# GitHub 通知接入钉钉

`quyun-ops-2` 各仓库复用此处的钉钉通知工作流。通知包含分支和标签推送，以及同仓 PR 的新建、更新、重开、合并和关闭。合并会同时产生 PR 合并和目标分支推送两个事件。来自 fork 的 PR 运行无法访问机器人 Secret。

## 新仓接入

1. 在 GitHub Actions 选择组织「运营库钉钉通知」模板，或把 [模板 YAML](workflow-templates/notify-dingtalk.yml) 放在新仓 `.github/workflows/notify-dingtalk.yml`。
2. 在新仓 Settings → Secrets and variables → Actions 添加 `DINGTALK_WEBHOOK_URL`。机器人开启加签时，再添加 `DINGTALK_SIGN_SECRET`。当前组织使用 GitHub Free，私有仓使用仓库级 Secret。
3. 把工作流合并到默认分支。在 Actions 选择「钉钉推送通知」并手动运行。运行日志须显示 `errcode=0`。群里收到「GitHub 运营库 · 通知接入验证」后，可在同仓验证实际推送和 PR。
4. 新建仓库后，从组织仓库列表核对是否已启用通知。换机器人时同步各仓的 Secret，再分别运行一次验证。

Webhook 地址和加签密钥由运营库负责人管理。工作流只从 Secret 读取。维护指南记录所需 Secret 名称及操作步骤。

## 排查

- GitHub Actions 通过而群里缺消息时，核对机器人与目标群，并检查 Actions 回执。中央工作流仅在钉钉返回 `errcode=0` 后报告发送成功。
- 如提示关键词校验，核对机器人的关键词设置与消息中的「GitHub 运营库」。提示签名校验时，确认仓库的加签密钥与机器人一致。
- 同类格式统一维护在中央仓。运营库的工具上游检测和提交 PR 流程见 [运营库通知维护](https://github.com/quyun-ops-2/ops-hub/blob/main/kb/operations-library/github-notifications.md)。
