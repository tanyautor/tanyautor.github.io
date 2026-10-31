---
icon: fas fa-graduation-cap
order: 3
---


<section class="outlined-section-wrapper">
  <section class="outlined-section">
    <h2>My Background</h2>

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