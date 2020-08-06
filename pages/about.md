---
layout: page
title: About
description: 不积跬步无以至千里
keywords: Xinlong Zhu,
comments: true
menu: 关于
permalink: /about/
---

人生

总会有不期而遇的温暖 

和生生不息的希望

## 联系

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
{% if site.url contains 'zhuxinlong' %}
<li>
微信：<br />
<img style="height:192px;width:192px;border:1px solid lightgrey;" src="{{ assets_base_url }}/assets/images/qrcode.jpg" alt="程序员" />
</li>
{% endif %}
</ul>


## Skill Keywords

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
