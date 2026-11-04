---
layout: page
icon: fas fa-stream
order: 1
---


<section class="outlined-section-wrapper">
  <section class="outlined-section">

  <h1 class="dynamic-title">
    My Projects
  </h1>

  <div class="projects-container">
    {% for project in site.data.projects %}
      <div class="card-wrapper">
        <div class="project-card" data-tags="{{ project.tags | join: ',' }}">
          {% include media.html src=project.image alt=project.title %}
          {% include team-badge.html people=project.people %}
            {% include release-badge.html release=project.release %}
          <h3 style="display:inline-block;white-space:break-spaces;">{{ project.title }}</h3>
          <a href="{{ project.link }}" class="card-link"></a>

          <div class="tags">
            {% for desc in project.desc limit: 3 %}
              <span class="tag desc-tag">{{ desc }}</span>
            {% endfor %}
          </div>

        </div>
      </div>

    {% endfor %}

  </div>
  </section>
</section>
