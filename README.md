---

layout: home

title: June的博客

permalink: /

---

[comment]: <> (<h2>文章列表</h2>)

[comment]: <> ({% for post in site.posts %})

[comment]: <> (<h3><a href="{{ post.url }}">{{ post.title }}</a></h3>)

[comment]: <> ({% endfor %})

<h2>分类列表</h2>

{% for category in site.categories %}

<h3>{{ category[0] }}</h3>

  <ul>

    {% for post in category[1] %}

      <li><a href="{{ post.url }}">{{ post.title }}</a></li>

    {% endfor %}

  </ul>

{% endfor %}