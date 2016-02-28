---
layout: page
show_meta: false
title: "乐高 | LEGO"
subheadline: "大朋友也可以玩乐高"
teaser: "拼砌乐高的快乐，我会在这里分享。"
header:
   image_fullwidth: "header_lego.jpg"
permalink: "/lego/"
---
<ul>
    {% for post in site.tags.lego %}
    <li><a href="{{ site.url }}{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
</ul>