---
layout: page
icon: fas fa-stream
order: 1
---


<link rel="stylesheet" href="../assets/css/cards.css">
<link rel="stylesheet" href="../assets/css/links.css">
<link rel="stylesheet" href="../assets/css/highlighted_projects.css">
<link rel="stylesheet" href="../assets/css/section-highlight.css">
<link rel="stylesheet" href="../assets/css/tag-colors.css">

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
            <h3 style="display:inline-block;white-space:break-spaces;">{{ project.title }}</h3>
            <a href="{{ project.link }}" class="card-link"></a>

            <div class="tags">

              {% for tag in project.tags %}
                {% assign tag_slug = tag | slugify %}

                <span class="tag" style="background-color: var(--tag-{{ tag_slug }}-bg); color: var(--tag-{{ tag_slug }}-text);">
                  {{ tag }}
                </span>

              {% endfor %}

            </div>   

            <div class="desc">
              <ul class="desc" style="list-style-type:disc;">

              {% for desc in project.desc %}

                <li>
                  <div class="desc" style="float:left;"> {{ desc }} </div> <br>
                </li>

              {% endfor %}
              </ul>
            </div>

          </div>
        </div>

      {% endfor %}

    </div>
  </section>
</section>
