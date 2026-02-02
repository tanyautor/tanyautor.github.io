---
icon: fas fa-graduation-cap
order: 3
---

<link rel="stylesheet" href="/assets/css/cards.css">
<link rel="stylesheet" href="/assets/css/links.css">
<link rel="stylesheet" href="/assets/css/highlighted_projects.css">
<link rel="stylesheet" href="/assets/css/section-highlight.css">

<section class="outlined-section-wrapper">
  <section class="outlined-section">

  <div class="projects-container">
    {% for project in site.data.education %}
      <div class="card-wrapper">
        <div class="project-card">
          <img src="{{ project.image }}" alt="{{ project.title }}">
          <h3>{{ project.title }}</h3>
          <h3>{{ project.period}}</h3>
          <a href="{{ project.link }}"  target="_blank" rel="noopener noreferrer" class="card-link"></a>
        </div>
      </div>
    {% endfor %}
  </div>
  </section>
</section>