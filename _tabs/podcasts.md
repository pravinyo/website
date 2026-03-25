---
title: Podcasts
icon: fas fa-podcast
order: 1
---

{% assign sorted_podcasts = site.podcasts | sort: 'date' | reverse %}

{% if sorted_podcasts.size > 0 %}
  <div id="podcast-list" class="row gx-3 gy-3">
    {% for podcast in sorted_podcasts %}
      <div class="col-12 col-md-6">
        <article class="card h-100 shadow-sm">
          <a href="{{ podcast.url | relative_url }}" class="text-decoration-none">
            {% if podcast.image %}
              {% assign src = podcast.image.path | default: podcast.image %}
              {% unless src contains '//' %}
                {% assign src = podcast.media_subpath | append: '/' | append: src | replace: '//', '/' %}
              {% endunless %}
              {% assign alt = podcast.image.alt | xml_escape | default: 'Preview Image' %}
              <img src="{{ src }}" class="card-img-top" alt="{{ alt }}" style="height: 200px; object-fit: cover;">
            {% endif %}
            <div class="card-body p-3">
              <h2 class="h5 card-title mb-1">{{ podcast.title }}</h2>
              <p class="text-muted small mb-1">{% include datetime.html date=podcast.date lang=site.lang %}</p>
              <p class="card-text mb-2">{{ podcast.description | default: podcast.excerpt | strip_html | truncate: 120 }}</p>
              <p class="small mb-0">
                {% if podcast.audio %}
                  <i class="fas fa-headphones-alt me-1"></i> Audio
                {% elsif podcast.video %}
                  <i class="fas fa-video me-1"></i> Video
                {% endif %}
                <strong class="ms-2">Listen</strong>
              </p>
            </div>
          </a>
        </article>
      </div>
    {% endfor %}
  </div>
{% else %}
  <div class="text-center py-5">
    <i class="fas fa-podcast fa-3x text-muted mb-3"></i>
    <h3 class="text-muted">No podcast episodes yet</h3>
    <p class="text-muted">Check back later for updates!</p>
  </div>
{% endif %}