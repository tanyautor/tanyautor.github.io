---
layout: page
icon: fas fa-info-circle
order: 3
---

<section class="outlined-section about-intro">
  <p>
    Hi! I'm Tanya, a games programming student at Breda University of Applied Sciences.
    In most group projects I've been able to shine by taking on a broad range of tasks:
    design-heavy gameplay programming, DevOps (including CI/CD pipelines), QA processes,
    and even setting up itch.io and Steam pages. On the side I've been exploring graphics
    programming and wrote some <a href="{{ '/blogs/' | relative_url }}">blog posts</a> about it.
  </p>
  <p>
    Most of my experience is in C++, with low-level insight from custom tech projects as well as
    working within larger frameworks like Unreal Engine. I also use C#, mostly in Unity.
  </p>
  <p class="about-status">Currently looking for a programming internship.</p>

  <dl class="skill-list">
    <dt>Languages</dt>
    <dd><span class="tag desc-tag">C++</span><span class="tag desc-tag">C#</span><span class="tag desc-tag">GLSL / HLSL</span></dd>
    <dt>Engines &amp; tech</dt>
    <dd><span class="tag desc-tag">Unreal Engine 5</span><span class="tag desc-tag">Custom C++ engines</span><span class="tag desc-tag">OpenGL</span><span class="tag desc-tag">Unity</span></dd>
    <dt>Workflow</dt>
    <dd><span class="tag desc-tag">CI/CD</span><span class="tag desc-tag">QA &amp; bug tracking</span><span class="tag desc-tag">Steam &amp; itch.io releases</span></dd>
  </dl>

  <div class="intro-links">
    <a class="cv-button" href="{{ '/assets/files/Tanya_Helmers_CV.pdf' | relative_url }}" target="_blank" rel="noopener">
      <i class="fas fa-file-lines"></i> View my CV
    </a>
    <div class="intro-icons">
      <a class="intro-icon" href="https://github.com/{{ site.github.username }}" target="_blank" rel="noopener" aria-label="GitHub"><i class="fab fa-github"></i></a>
      <a class="intro-icon" href="https://www.linkedin.com/in/{{ site.LinkedIn.username }}" target="_blank" rel="noopener" aria-label="LinkedIn"><i class="fab fa-linkedin"></i></a>
    </div>
  </div>
</section>

<section class="outlined-section-wrapper" id="education">
  <section class="outlined-section">
    <h2 class="dynamic-title">Education</h2>

    <ol class="timeline">
      {% for edu in site.data.education %}
        <li class="timeline-entry">
          <a class="timeline-logo" href="{{ edu.link }}" target="_blank" rel="noopener noreferrer" aria-label="{{ edu.school }} website">
            <img src="{{ edu.image | relative_url }}" alt="{{ edu.school }} logo">
          </a>
          <div class="timeline-body">
            <h3>{{ edu.degree }}</h3>
            <p class="timeline-meta">
              <a href="{{ edu.link }}" target="_blank" rel="noopener noreferrer">{{ edu.school }}</a>
              {% if edu.location %} &middot; {{ edu.location }}{% endif %}
              &middot; {{ edu.period }}
            </p>
            {% if edu.highlights %}
              <ul>
                {% for h in edu.highlights %}<li>{{ h | markdownify | remove: '<p>' | remove: '</p>' | strip }}</li>{% endfor %}
              </ul>
            {% endif %}
            {% if edu.skills %}
              <div class="timeline-skills">
                {% for s in edu.skills %}<span class="tag desc-tag">{{ s }}</span>{% endfor %}
              </div>
            {% endif %}
          </div>
        </li>
      {% endfor %}
    </ol>
  </section>
</section>
