---
layout: page
title: Categories
permalink: /categories/
---

<div class="categories">
  {% for category in site.categories %}
    <div class="category">
      <h2 id="{{ category[0] | slugify }}">{{ category[0] }}</h2>
      <ul>
        {% for post in category[1] %}
          <li>
            <a href="{{ post.url }}">{{ post.title }}</a>
            <span class="post-date">{{ post.date | date: "%B %d, %Y" }}</span>
          </li>
        {% endfor %}
      </ul>
    </div>
  {% endfor %}
</div>

<style>
.categories {
  margin: 2em 0;
}

.category {
  margin-bottom: 2em;
}

.category h2 {
  border-bottom: 1px solid #ccc;
  padding-bottom: 0.5em;
}

.category ul {
  list-style: none;
  padding-left: 1em;
}

.category li {
  margin: 0.5em 0;
}

.post-date {
  color: #666;
  font-size: 0.9em;
  margin-left: 1em;
}
</style> 