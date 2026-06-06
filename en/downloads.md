---
layout: page
title: Downloads
subtitle: Presentations, documents and other resources available for download
lang: en
lang-ref: downloads
---

<style>
  .download-list {
    list-style: none;
    padding: 0;
  }
  .download-item {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 14px 18px;
    margin-bottom: 10px;
    background: rgba(255, 255, 255, 0.07);
    border-radius: 8px;
    transition: background 0.2s;
  }
  .download-item:hover {
    background: rgba(255, 255, 255, 0.13);
  }
  .download-info {
    display: flex;
    align-items: center;
    gap: 12px;
    min-width: 0;
  }
  .download-icon {
    font-size: 1.5rem;
    flex-shrink: 0;
  }
  .download-name {
    font-weight: 600;
    word-break: break-word;
  }
  .download-ext {
    font-size: 0.8rem;
    text-transform: uppercase;
    opacity: 0.7;
  }
  .download-btn {
    flex-shrink: 0;
    margin-left: 16px;
    padding: 6px 18px;
    border-radius: 5px;
    background: #FCB021;
    color: #333;
    font-weight: 600;
    text-decoration: none;
    transition: opacity 0.2s;
  }
  .download-btn:hover {
    opacity: 0.85;
    text-decoration: none;
    color: #333;
  }
  .no-files {
    text-align: center;
    opacity: 0.6;
    margin-top: 40px;
  }
</style>

{% assign downloads = site.static_files | where_exp: "file", "file.path contains '/assets/downloads/'" | where_exp: "file", "file.name != '.gitkeep'" %}

{% if downloads.size > 0 %}
<ul class="download-list">
  {% for file in downloads %}
    {% assign ext = file.extname | remove: "." | downcase %}
    {% if ext == "pdf" %}
      {% assign icon = "📄" %}
    {% elsif ext == "pptx" or ext == "ppt" %}
      {% assign icon = "📊" %}
    {% elsif ext == "docx" or ext == "doc" %}
      {% assign icon = "📝" %}
    {% elsif ext == "xlsx" or ext == "xls" %}
      {% assign icon = "📈" %}
    {% elsif ext == "zip" or ext == "tar" or ext == "gz" %}
      {% assign icon = "📦" %}
    {% else %}
      {% assign icon = "📎" %}
    {% endif %}
    {% assign display_name = file.name | split: "." | first | replace: "-", " " | replace: "_", " " %}
    <li class="download-item">
      <div class="download-info">
        <span class="download-icon">{{ icon }}</span>
        <div>
          <div class="download-name">{{ display_name }}</div>
          <div class="download-ext">{{ ext }}</div>
        </div>
      </div>
      <a class="download-btn" href="{{ file.path }}" download>Download</a>
    </li>
  {% endfor %}
</ul>
{% else %}
<p class="no-files">No files available for download yet.</p>
{% endif %}
