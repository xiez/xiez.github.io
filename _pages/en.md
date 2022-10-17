---
title: "English Posts"
permalink: /en/
author_profile: true
---


<div class="">
  {% for post in site.posts %}
    {% if post.type == 'en' %}
    <div class="list__item">
        <article class="archive__item" itemscope="" itemtype="https://schema.org/CreativeWork">

          <h2 class="archive__item-title" itemprop="headline"><a href="{{ site.baseurl }}{{ post.url }}" rel="permalink">{{ post.title }}</a></h2>
          
          <p class="page__meta">
            {{ post.date | date: '%B %d, %Y' }}
          </p>
          
          <div class="archive__item-excerpt" itemprop="description">
            {{ post.excerpt }}
            <a href="{{ site.baseurl }}{{ post.url }}" class="read-more">Read More</a>
          </div>
        </article>
    </div>
    {% endif %}
  {% endfor %}
</div>
