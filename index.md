---
layout: default
title: Home
---

# keepConcentration

Personal blog powered by Jekyll and GitHub Pages.

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.date | date: "%Y-%m-%d" }} — {{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
