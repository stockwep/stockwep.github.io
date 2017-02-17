---
layout: page
title: Tags
permalink: /tags/
---

<ul class="tags">
{% for tag_page in site.tag_pages %}
  {% assign posts = site.tags[tag_page.title] %}
  <li><a href="{{ tag_page.url | relative_url }}">{{tag_page.title}} ({{ posts | size }} posts) - {{tag_page.summary}}</a></li>
{% endfor %}
</ul>
