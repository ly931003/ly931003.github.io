---
layout: page
title: About
description: 闲的时候可以写一点博客试一下效果。
keywords: Lian Yin
comments: true
menu: 关于
permalink: /about/
---

闲的时候可以写一点博客试一下效果。

## 联系

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
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
