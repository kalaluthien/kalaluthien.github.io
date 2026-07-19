---
layout: page
icon: fas fa-book
order: 2
---

Notes from an Obsidian-based knowledge base, published as a browsable wiki. Each
entry lives under `/wiki/` and is maintained in the vault, then synced here. The
full set is listed below, alphabetically by title.

{% assign entries = site.wiki | sort_natural: 'title' %}

{% if entries.size == 0 %}
<p>No wiki pages yet.</p>
{% else %}
<ul class="list-unstyled">
  {% for entry in entries %}
  <li>
    <a href="{{ entry.url | relative_url }}">{{ entry.title | default: entry.name }}</a>
  </li>
  {% endfor %}
</ul>
{% endif %}
