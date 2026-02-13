---
layout: page
title: Tags
permalink: /tags/
---

<div class="tag-sphere-wrapper">
  <div class="tag-sphere">
    {% assign max_count = 0 %}
    {% for tag in site.tags %}
      {% if tag[1].size > max_count %}
        {% assign max_count = tag[1].size %}
      {% endif %}
    {% endfor %}
    {% for tag in site.tags %}
      {% assign count = tag[1].size %}
      {% assign ratio = count | times: 100 | divided_by: max_count %}
      {% if ratio > 80 %}
        {% assign size_class = "tag-sphere-5" %}
      {% elsif ratio > 60 %}
        {% assign size_class = "tag-sphere-4" %}
      {% elsif ratio > 40 %}
        {% assign size_class = "tag-sphere-3" %}
      {% elsif ratio > 20 %}
        {% assign size_class = "tag-sphere-2" %}
      {% else %}
        {% assign size_class = "tag-sphere-1" %}
      {% endif %}
      <a class="tag-sphere-item {{ size_class }}" href="#{{ tag[0] | slugify }}" style="--i: {{ forloop.index0 }}; --total: {{ site.tags.size }};">{{ tag[0] }}</a>
    {% endfor %}
  </div>
</div>

<div class="tags-page">
{% for tag in site.tags %}
  <h3 id="{{ tag[0] | slugify }}">{{ tag[0] }}</h3>
  <ul>
    {% for post in tag[1] %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}
</div>
