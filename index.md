---
layout: default
title: Home
---

<div class="hero-wrap">
  <div class="hero">
    <div class="label">docs-style · jekyll · github pages</div>
    <h1>{{ site.description }}</h1>
    <p class="lede">{{ site.tagline }}</p>
    <div class="actions">
      {% if site.posts.size > 0 %}
      <a href="{{ site.posts.first.url | relative_url }}" class="btn primary">최신 글 보기</a>
      {% endif %}
      <a href="#posts" class="btn">글 목록 보기</a>
    </div>
  </div>
</div>

<div class="shell" id="posts">
  {% assign all_tags = site.posts | map: 'tags' | join: ',' | split: ',' | compact | uniq | sort %}
  {% if all_tags.size > 0 %}
  <aside class="side">
    <h3>Categories</h3>
    <a href="#" class="cat-link on" data-tag="all">All posts</a>
    {% for tag in all_tags %}
    <a href="#" class="cat-link" data-tag="{{ tag }}">{{ tag }}</a>
    {% endfor %}
  </aside>
  {% endif %}

  <main class="content">
    <div class="toolbar">
      <h2>Posts</h2>
    </div>
    <div class="grid">
      {% if site.posts.size == 0 %}
      <div class="empty-state">
        <h3>No posts yet</h3>
        <p>아직 발행된 글이 없습니다. <code>_posts/YYYY-MM-DD-title.md</code> 를 추가하면 여기에 표시됩니다.</p>
      </div>
      {% else %}
      {% for post in site.posts %}
      <a href="{{ post.url | relative_url }}" class="card" data-tags="{{ post.tags | join: ',' }}">
        <div class="meta">
          <span>{{ post.date | date: "%Y-%m-%d" }}</span>
          {% if post.tags %}
            {% for tag in post.tags %}
            <span class="tag">{{ tag }}</span>
            {% endfor %}
          {% endif %}
        </div>
        <h3>{{ post.title }}</h3>
        {% if post.excerpt %}
        <p>{{ post.excerpt | strip_html | truncatewords: 20 }}</p>
        {% endif %}
      </a>
      {% endfor %}
      {% endif %}
    </div>
  </main>
</div>

{% if all_tags.size > 0 %}
<script>
  const catLinks = document.querySelectorAll('.cat-link');
  const cards = document.querySelectorAll('.card');

  catLinks.forEach(link => {
    link.addEventListener('click', e => {
      e.preventDefault();
      const tag = link.dataset.tag;

      // Update active state
      catLinks.forEach(l => l.classList.remove('on'));
      link.classList.add('on');

      // Filter cards
      cards.forEach(card => {
        const cardTags = card.dataset.tags;
        if (tag === 'all' || cardTags.includes(tag)) {
          card.style.display = '';
        } else {
          card.style.display = 'none';
        }
      });
    });
  });
</script>
{% endif %}
