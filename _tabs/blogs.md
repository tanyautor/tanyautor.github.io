---
title: Blog Posts
layout: page
icon: fas fa-archive
order: 2
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
    My Posts
  </h1>

    <div class="projects-container">
      {% for post in site.data.posts %}
        <div class="card-wrapper">
          <div class="project-card" data-tags="{{ post.tags | join: ',' }}">
            <img src="{{ post.image }}" alt="{{ post.title }}">
            <h3>{{ post.title }}</h3>
            <a href="{{ post.link }}" class="card-link"></a>

            <div class="tags">

              {% for tag in post.tags %}
                {% assign tag_slug = tag | slugify %}

                <span class="tag" style="background-color: var(--tag-{{ tag_slug }}-bg); color: var(--tag-{{ tag_slug }}-text);">
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