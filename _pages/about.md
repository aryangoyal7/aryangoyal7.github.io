---
permalink: /
title: ""
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---

Hi, I'm Aryan Goyal. 
I will graduate from IIT Bombay this summer.

I work in AI research.
Currently I am working with Mechansitic Interpretability of Vision Language Action Models.

My previous work was most with fine-grained medical imaging.

I write technical blogs about my scrappy experiments on [Substack](https://aryangoyal.substack.com).

{% include base_path %}

## Publications

{% if site.publication_category %}
  {% for category in site.publication_category  %}
    {% assign title_shown = false %}
    {% for post in site.publications reversed %}
      {% if post.category != category[0] %}
        {% continue %}
      {% endif %}
      {% unless title_shown %}
        <h3>{{ category[1].title }}</h3>
        {% assign title_shown = true %}
      {% endunless %}
      {% include archive-single.html %}
    {% endfor %}
  {% endfor %}
{% else %}
  {% for post in site.publications reversed %}
    {% include archive-single.html %}
  {% endfor %}
{% endif %}

## Blog Posts

{% for post in site.posts reversed %}
  {% include archive-single.html %}
{% endfor %}

## Talks

{% for post in site.talks reversed %}
  {% include archive-single-talk.html %}
{% endfor %}

## Tutorials

{% for post in site.tutorials reversed %}
  {% include archive-single.html %}
{% endfor %}