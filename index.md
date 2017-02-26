---
layout: default
spacious: true
---
{% for post in site.posts %}
<div class="post-container">
    <h2>
      <a class="post-link" href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
    </h2>
    <div class="post-excerpt">
      <p>
        {{ post.content | strip_html | truncatewords: 50 }}
      </p>
    </div>
    </div>
{% endfor %}
