---
layout: page
title: About
description: "No CODE No BB - 来自一位不愿透露姓名的屎带芬"
keywords: Stephen Yin
comments: true
menu: 关于
permalink: /about/
---

> No CODE No BB - 来自一位不愿透露姓名的屎带芬

## 联系

* GitHub：[@stephenyin](https://github.com/stephenyin)
* 博客：[{{ site.title }}]({{ site.url }})
* Email：<stephen.yin.h@gmail.com>

## Skill Keywords

#### Software Engineer Keywords
<div class="btn-inline">
    {% for keyword in site.skill_software_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>

#### Linux Developer Keywords
<div class="btn-inline">
    {% for keyword in site.skill_linux_keywords %}
    <button class="btn btn-outline" type="button">{{ keyword }}</button>
    {% endfor %}
</div>
