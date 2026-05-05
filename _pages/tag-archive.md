---
title: "Posts and Projects by Tag"
permalink: /tags/
layout: archive
author_profile: true
---

{% assign posts = site.posts %}
{% assign projects = site.projects %}
{% assign documents = posts | concat: projects %}

{% assign raw_tags = "" %}

{% for document in documents %}
  {% if document.tags %}
    {% for tag in document.tags %}
      {% assign raw_tags = raw_tags | append: tag | append: "," %}
    {% endfor %}
  {% endif %}
{% endfor %}

{% assign tags = raw_tags | split: "," | uniq | sort %}

{% for tag in tags %}
  {% if tag != "" %}
   <h2 class="archive__subtitle">{{ tag }}</h2>

    {% assign tagged_documents = documents | where_exp: "document", "document.tags contains tag" | sort: "date" | reverse %}

    {% for post in tagged_documents %}
      {% include archive-single.html %}
    {% endfor %}
  {% endif %}
{% endfor %}