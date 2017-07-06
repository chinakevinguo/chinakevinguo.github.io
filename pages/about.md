---
layout: page
title: About
description: Just dow it now ! 
keywords: KevinGuo
comments: true
menu: 关于
permalink: /about/
---

## 我只是个菜鸡

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
