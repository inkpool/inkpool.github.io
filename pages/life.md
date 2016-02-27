---
layout: page
show_meta: false
title: "我的生活 | My Life"
subheadline: "酸甜苦辣与众品 痴笑癫狂我独醉"
teaser: "我把我的生活记录在这里，包括我的琐事、见闻和兴趣爱好。"
header:
   image_fullwidth: "header_mylife.jpg"
permalink: "/life/"
---
<ul>
    {% for post in site.categories.life %}
    <li><a href="{{ site.url }}{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
</ul>