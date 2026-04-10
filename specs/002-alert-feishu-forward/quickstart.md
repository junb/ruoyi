# Quickstart: 告警飞书转发

**Feature**: 002-alert-feishu-forward

## 前置条件

- RuoYi-Vue-Plus 后端已启动（端口 8080）
- MySQL 数据库 `ry-vue` 可访问
- 飞书群自定义机器人已创建，Webhook URL 已获取

## 步骤 1：执行数据库脚本

```bash
# 1. 建表 + 字典数据 + 菜单权限
mysql -u root -p ry-vue < RuoYi-Vue-Plus/script/sql/alert.sql
```

## 步骤 2：配置字典

在系统管理 → 字典管理中找到 `alert_config`，配置：

| 键值 | 值 |
|------|-----|
| `FEISHU_WEBHOOK_URL` | `https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxx` |
| `WEBHOOK_TOKEN` | 自定义 token（如 `my-alert-token-2026`） |
| `RETENTION_DAYS` | `90` |
| `DEFAULT_TENANT_ID` | `000000`（或实际租户 ID） |

## 步骤 3：配置 Alertmanager

在 Alertmanager 配置文件 `alertmanager.yml` 中添加 webhook_configs：

```yaml
route:
  receiver: 'webhook'
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h

receivers:
  - name: 'webhook'
    webhook_configs:
      - url: 'http://your-server:8080/webhook/alert/my-alert-token-2026'
        send_resolved: true
```

## 步骤 4：验证

1. 触发一条告警规则（或使用 Alertmanager API 手动发送）
2. 检查飞书群是否收到消息卡片
3. 在管理界面 → 巡检管理 → 告警记录 中查看记录
