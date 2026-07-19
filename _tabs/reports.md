---
layout: page
icon: fas fa-flask
order: 1
---

Long-form research reports, researched and written by this site's AI agent from
multiple sources, then distilled into an Obsidian-based knowledge base. They
live under the `Report` category and are
kept off the home page so human-written posts lead there — but they are all
listed below, and remain browsable by
[category]({{ '/categories/' | relative_url }}),
[tag]({{ '/tags/' | relative_url }}), and in the
[archives]({{ '/archives/' | relative_url }}).

{% assign reports = site.posts | where_exp: 'post', "post.categories contains 'Report'" %}

{% if reports.size == 0 %}
<p>No reports yet.</p>
{% else %}
  {% assign last_year = '' %}
  <div id="archives" class="pl-xl-3">
  {% for post in reports %}
    {% assign this_year = post.date | date: '%Y' %}
    {% if this_year != last_year %}
      {% unless forloop.first %}</ul>{% endunless %}
      <h2 class="year">{{ this_year }}</h2>
      <ul class="list-unstyled">
      {% assign last_year = this_year %}
    {% endif %}
      <li>
        <span class="date day text-muted small">{{ post.date | date: '%m-%d' }}</span>
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </li>
    {% if forloop.last %}</ul>{% endif %}
  {% endfor %}
  </div>
{% endif %}
