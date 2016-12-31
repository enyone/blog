# The Blog
This is a repository containing pages and sources of my blog. Nothing more.

Blog can be accessed at http://blog.rainio.org and sources at https://github.com/enyone/blog

Posts
---

<ul class="posts">
{% for post in site.tags.question limit: 5 %}
  <div class="post_info">
    <li>
         <a href="{{ post.url }}">{{ post.title }}</a>
         <span>({{ post.date | date:"%Y-%m-%d" }})</span>
    </li>
    </div>
  {% endfor %}
</ul>
