---
layout: page
title: Talks & Webinars
subtitle: Information about talks & webinars I have participated on
lang: en
lang-ref: talks
---
{% assign talks = site.talks | sort: "order" %}
{% for talk in talks %}
<h2>{{ talk.title }}</h2>
{% assign lang_tag = "lang-en" %}{% assign lang_emoji = "🇬🇧" %}{% if talk.lang == "es" %}{% assign lang_tag = "lang-es" %}{% assign lang_emoji = "🇪🇸" %}{% endif %}
<div class="blog-tags">
  <span>Tags:</span>
  <a href="{{ '/tags' | relative_url }}#{{ lang_tag }}" title="{{ lang_tag }}">{{ lang_emoji }}</a>
  {% for tag in talk.tags %}
    <a href="{{ '/tags' | relative_url }}#{{ tag }}">{{ tag }}</a>
  {% endfor %}
</div>

<p align="center">
  <iframe style="aspect-ratio: 16 / 9; width: 100%;" src="https://www.youtube.com/embed/{{ talk.video }}{{ talk.video_params }}" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</p>

{% endfor %}
