---
layout: page
icon: fas fa-stream
order: 1
---


<link rel="stylesheet" href="../assets/css/cards.css">
<link rel="stylesheet" href="../assets/css/links.css">
<link rel="stylesheet" href="../assets/css/highlighted_projects.css">
<link rel="stylesheet" href="../assets/css/section-highlight.css">

<section class="outlined-section">
  These are mostly research topics I wrote blog posts about. Some of these were written for my time as a student at Breda University of applied sciences (BUas).
</section>

<section class="outlined-section-wrapper">
  <section class="outlined-section">

  <h1 class="dynamic-title">
    My Projects
  </h1>

    <div class="projects-container">
      {% for project in site.data.projects %}
        <div class="card-wrapper">
          <div class="project-card" data-tags="{{ project.tags | join: ',' }}">
            <img src="{{ project.image }}" alt="{{ project.title }}">
            <h3>{{ project.title }}</h3>
            <a href="{{ project.link }}" class="card-link"></a>

            <div class="tags">

              {% for tag in project.tags %}
                {% assign tag_data = site.data.tags[tag] %}

                <span class="tag" style="background-color: {{ tag_data.color }}; color: {{ tag_data.text_color }};">
                    {{ tag }}
                </span>
                
              {% endfor %}
            </div>   
          </div>
        </div>

      {% endfor %}
    
    </div>
  </section>
</section>
