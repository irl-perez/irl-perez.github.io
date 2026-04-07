---
title: Projects
icon: fa-solid fa-box-open
order: 4
layout: page
excerpt: ""
---

<div class="projects-grid">
  {% for project in site.projects %}
    <article class="project-card">
      <h2 class="project-title">
        <a href="{{ project.url | relative_url }}">{{ project.title }}</a>
      </h2>

      {% if project.description %}
        <p class="project-description">{{ project.description }}</p>
      {% endif %}

      {% if project.tech %}
        <p class="project-tech">
          <strong>Tech:</strong> {{ project.tech | join: ", " }}
        </p>
      {% endif %}
    </article>
  {% endfor %}
</div>