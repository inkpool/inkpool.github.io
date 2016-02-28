---
layout: page
show_meta: false
title: "我的旅行 | My Trip"
subheadline: "世界这么大 我想去看看"
teaser: "俗话说“父母在不远游，游必有方”，我也许只能通过旅行走向远方。"
header:
   image_fullwidth: "header_travel.jpg"
permalink: "/travel/"
---
<ul>
    {% for post in site.categories.travel %}
    <li><a href="{{ site.url }}{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
</ul>