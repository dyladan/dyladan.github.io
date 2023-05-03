---
layout: page
title: "Post Archive"
permalink: /posts/
main_nav: true
---

<div>
  {% for desc in site.descriptions %}
    <h2 id="{{cat}}">{{ desc.cat | capitalize }}</h2>
    {% if desc.desc %}
      <p class="desc"><em>{{ desc.desc }}</em></p>
    {% endif %}
    <ul class="posts-list">
    {% for post in site.categories[desc.cat] %}
    <li>
      <strong>
        <a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
      </strong>
      <span class="post-date">- {{ post.date | date_to_long_string }}</span>
    </li>
    {% endfor %}
  </ul>
  {% if forloop.last == false %}
  <hr />
  {% endif %} {% endfor %}
  <br />
</div>

