---
layout: default
---

Hi, I'm Ramakant. I enjoy building and understanding systems from the hardware up to the application layer...

### Recent Posts
{% for post in site.posts %}
* [{{ post.title }}]({{ post.url }}) — {{ post.date | date: "%b %d, %Y" }}
{% endfor %}
