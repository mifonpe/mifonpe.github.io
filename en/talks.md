---
layout: page
title: Talks & Webinars
subtitle: Information about talks & webinars I have participated on
lang: en
lang-ref: talks
---
{% assign talks = site.talks | sort: "order" %}
{% for talk in talks %}
<h2>{{ talk.title }} {% if talk.lang == "es" %}🇪🇸{% else %}🇬🇧{% endif %}</h2>
<div class="blog-tags">
  <span>Tags:</span>
  {% for tag in talk.tags %}
    <a href="{{ '/tags' | relative_url }}#{{ tag }}">{{ tag }}</a>
  {% endfor %}
</div>

<p align="center">
  <iframe style="aspect-ratio: 16 / 9; width: 100%;" src="https://www.youtube.com/embed/{{ talk.video }}{{ talk.video_params }}" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</p>

{% endfor %}
