---
layout: index
title: KevinsBobo - Coding博客
---

<img src = "assets/img/9348be98ed46c1cd972743184b76fe73.png" style="width:80%;max-width:350px;" alt="coding" />

> 技术：一忍不住就折腾，能偷懒的必偷懒；<br>博文：不写前人写过的，无法超越就学习。<br><br>这是一个存在于互联网角落的博客，<br>如果被你发现了，说明梯子搭的好。<br><br>Enjoy it ^_^

---

## 文章目录

{% for category in site.categories %}
{% if category.first != 'hidden' %}
<h3>{{ category | first }} ({{ category | last | size }})</h3>
<ul>
{% for post in category.last %}
{% if post.jump  %}
    <li><h4>{{ post.date | date: '%b %Y' }} <a href="{{ post.jumpurl }}" target="view_window">{{ post.title }}</a></h4></li>
{% else  %}
    <li><h4>{{ post.date | date: '%b %Y' }} <a href="{{ post.url }}">{{ post.title }}</a></h4></li>
{% endif %}
{% endfor %}
</ul>
{% endif %}
{% endfor %}

