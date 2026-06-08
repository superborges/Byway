# Phase 3.12 公开 URL 攻略吸收设计

## 目标

Byway 当前能吸收用户粘贴的 Markdown/文本攻略，也能对小红书链接做安全降级。但普通公开网页 URL 仍只保存链接，无法把可访问正文变成 GuideSource。

Phase 3.12 增加最小公开网页正文抽取：

- 用户发送普通 `https://...` URL 时，Proxy 尝试读取公开 HTML。
- 提取 `<title>` 和可读正文片段，剔除 script/style。
- 将提取文本作为 `user_url` GuideSource 交给 MCP `ingest_user_guide_source`。
- 如果抓取失败、非 HTML、超时或是小红书等平台链接，则保存 URL 和用户备注，并明确降级。

## 边界

- 不登录、不绕过权限、不抓评论区。
- 不承诺读取完整页面，只做可访问正文摘要。
- 小红书链接继续走现有 `xiaohongshu_link` 降级路径。

## 验收

- 公开 URL 返回 GuideSummaryArtifact。
- assistant 文案说明“已读取公开网页正文片段”。
- 小红书链接仍说明“无法读取完整笔记正文”。
