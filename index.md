---
layout: default
---

{% for post in site.posts %}
<p>{{post.url}}</p>
{% endfor %}