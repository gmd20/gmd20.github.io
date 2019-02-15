---
layout: default
---

<div class="archives" id="home">
  <h1 class="pageTitle">Archives</h1>
  <ul class="posts noList">
    {% for post in site.posts %}
      <li>
        <span>{{ post.date | date: '%Y-%m-%d  ' }}</span>
        <a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a> 
      </li>
    {% endfor %}
  </ul>
</div>
