---
layout: page
show_meta: false
title: "我的学习 | My Learning"
subheadline: "太多东西不会 太少时间学习"
teaser: "上了十八年的学，却一无所成。我把我在学习、实践中遇到的困难和问题写在这里。"
header:
   image_fullwidth: "header_study.jpg"
permalink: "/study/"
---
<ul>
    {% for post in site.categories.study %}
    <li><a href="{{ site.url }}{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
</ul>