---
seo_title: 朋友文章
robots: noindex,nofollow
menu_id: post
comments: false
post_list: true # 这就意味着页面会显示首页文章导航栏
sidebar: [welcome, recent]
---

{% timeline type:fcircle api:https://raw.githubusercontent.com/mikaelzero/friends-rss-generator/output/data.json %}
{% endtimeline %}
