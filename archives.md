---
layout: default
---

<div class="archives" id="home">
  <h1 class="pageTitle">Archives</h1>
  <ul class="posts noList">
    {% for post in site.posts %}
      <li>
        <span class="date">{{ post.date | date: '%Y-%m-%d  ' }}</span>
        <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a> 
      </li>
    {% endfor %}
  </ul>
</div>
