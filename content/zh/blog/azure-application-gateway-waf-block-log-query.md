---
title: "Azure Application Gateway WAF 拦截日志查询与规则关闭方法"
date: 2026-06-25
draft: false
tags: ["Azure", "Application Gateway", "WAF", "KQL", "Log Analytics"]
categories: ["云计算"]
---

## 前言

在使用 Azure Application Gateway WAF 时，有时正常业务请求会被 WAF 规则拦截。遇到这种情况，最直接的排查方式是先查 WAF 拦截日志，找到触发拦截的 `Rule ID`，再到 WAF Policy 中关闭对应规则或配置例外。

本文记录一个常见排查流程：先用 KQL 查询被拦截的请求，再根据日志中的规则 ID 到 WAF 策略里处理对应规则。

<!--more-->

## 一、查询 WAF 拦截日志

进入 Azure Portal，打开对应的 **Log Analytics Workspace**，进入 **Logs** 页面后，可以使用 KQL 查询 WAF 日志。

如果已经知道具体的规则 ID 和请求路径，可以使用下面的查询：

```kql
AzureDiagnostics
| where ruleId_s == "<RULE_ID>"
| where action_s == "Blocked"
| where requestUri_s == "<REQUEST_PATH>"
```

字段说明：

| 字段 | 说明 |
|------|------|
| `ruleId_s` | WAF 命中的规则 ID，例如 `<RULE_ID>` |
| `action_s` | WAF 执行动作，`Blocked` 表示请求被拦截 |
| `requestUri_s` | 被拦截的请求路径，例如 `<REQUEST_PATH>` |

如果还不知道是哪条规则触发了拦截，可以先查询所有被拦截的请求：

```kql
AzureDiagnostics
| where action_s == "Blocked"
| project TimeGenerated, clientIp_s, requestUri_s, ruleId_s, message_s, transactionId_g
| order by TimeGenerated desc
```

这条语句可以快速看到被拦截的时间、来源 IP、请求路径、命中的规则 ID 和规则说明。

## 二、确认是否为误拦截

查到日志后，重点看下面几个字段：

| 字段 | 排查用途 |
|------|----------|
| `ruleId_s` | 后续关闭规则时需要用到 |
| `requestUri_s` | 确认是哪个接口或页面被拦截 |
| `message_s` | 查看规则触发原因 |
| `clientIp_s` | 判断是否来自正常访问来源 |
| `transactionId_g` | 用于进一步关联排查 |

如果确认请求是正常业务流量，并且同一个 `Rule ID` 持续命中，就可以考虑关闭该规则，或者为指定路径配置例外。

## 三、根据 Rule ID 关闭 WAF 规则

拿到 `Rule ID` 后，进入 Azure Portal 执行以下操作：

1. 打开对应的 **Application Gateway**。
2. 找到关联的 **WAF Policy**。
3. 进入 **Managed rules**。
4. 找到对应规则组。
5. 根据日志里的 `Rule ID` 定位具体规则。
6. 将该规则设置为 **Disabled**。
7. 保存配置。

保存后等待配置生效，再回到 Log Analytics 中继续观察日志。

## 四、建议的处理方式

如果只是某个接口被误拦，建议优先考虑最小范围放行，而不是直接关闭整组规则。

常见做法包括：

1. 只关闭单个 `Rule ID`。
2. 只对指定请求路径配置例外。
3. 只对指定参数配置排除。
4. 只对可信来源做放行。

这样可以减少对整体 WAF 防护能力的影响。

## 五、排查流程总结

完整流程可以按下面顺序执行：

1. 使用 KQL 查询 `action_s == "Blocked"` 的 WAF 日志。
2. 找到被拦截请求对应的 `ruleId_s`。
3. 判断该请求是否为正常业务请求。
4. 到 WAF Policy 的 Managed rules 中定位规则。
5. 禁用对应规则或配置例外。
6. 保存后继续观察日志。

## 六、总结

Azure Application Gateway WAF 误拦截排查的关键是先查日志，再根据 `Rule ID` 定位规则。

核心 KQL 如下：

```kql
AzureDiagnostics
| where ruleId_s == "<RULE_ID>"
| where action_s == "Blocked"
| where requestUri_s == "<REQUEST_PATH>"
```

通过这类查询，可以快速确认是哪条 WAF 规则拦截了哪个请求。确认是误报后，再到 WAF Policy 中关闭对应规则或配置例外即可。
