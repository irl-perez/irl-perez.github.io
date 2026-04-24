---
title: Projects
icon: fa-solid fa-box-open
order: 4
layout: page
excerpt: ""
---

<div id="post-list" class="flex-grow-1 px-xl-1">
  {% for project in site.projects %}
    <article class="card-wrapper card">
      <a href="{{ project.url | relative_url }}" class="post-preview row g-0 flex-md-row-reverse">
        <div class="col-md-12">
          <div class="card-body d-flex flex-column">
            <h1 class="card-title my-2 mt-md-0">{{ project.title }}</h1>

            {% if project.description %}
              <div class="card-text content mt-0 mb-3">
                <p>{{ project.description }}</p>
              </div>
            {% endif %}

            {% if project.tech %}
              <div class="post-meta flex-grow-1 d-flex align-items-end">
                <div class="me-auto">
                  <i class="fas fa-code fa-fw me-1"></i>
                  <span>{{ project.tech | join: ", " }}</span>
                </div>
              </div>
            {% endif %}
          </div>
        </div>
      </a>
    </article>
  {% endfor %}
</div>