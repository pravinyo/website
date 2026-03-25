---
title: Podcasts
icon: fas fa-podcast
order: 1
---

{% include lang.html %}

{% assign sorted_podcasts = site.podcasts | sort: 'date' | reverse %}

{% if sorted_podcasts.size > 0 %}
  <div id="podcast-list">
    {% for podcast in sorted_podcasts %}

      {% assign src = '' %}
      {% if podcast.image %}
        {% assign src = podcast.image.path | default: podcast.image %}
        {% unless src contains '//' %}
          {% assign src = podcast.media_subpath | append: '/' | append: src | replace: '//', '/' %}
        {% endunless %}
      {% endif %}

      <article class="pc-card">
        <!-- Invisible full-card link via ::after on this anchor -->
        <a href="{{ podcast.url | relative_url }}" class="pc-card-anchor" aria-label="{{ podcast.title }}"></a>

        {% if src != '' %}
          <div class="pc-img">
            <img src="{{ src }}" alt="{{ podcast.image.alt | xml_escape | default: 'Cover' }}">
          </div>
        {% else %}
          <div class="pc-img pc-img-placeholder">
            <i class="fas fa-podcast"></i>
          </div>
        {% endif %}

        <div class="pc-body">
          <h2 class="pc-title">{{ podcast.title }}</h2>
          <p class="pc-date">
            <i class="far fa-calendar fa-fw me-1"></i>
            {% include datetime.html date=podcast.date lang=lang %}
          </p>
          <p class="pc-desc">
            {{ podcast.description | default: podcast.excerpt | strip_html | truncate: 100 }}
          </p>
          <p class="pc-meta">
            {% if podcast.audio %}<i class="fas fa-headphones-alt me-1"></i>Audio
            {% elsif podcast.video %}<i class="fas fa-video me-1"></i>Video
            {% endif %}
            <span class="ms-2">Listen →</span>
          </p>
        </div>
      </article>

    {% endfor %}
  </div>
{% else %}
  <div class="text-center py-5">
    <i class="fas fa-podcast fa-3x text-muted mb-3"></i>
    <h3 class="text-muted">No podcast episodes yet</h3>
    <p class="text-muted">Add episodes to the <code>_podcasts</code> folder to get started.</p>
  </div>
{% endif %}

<style>
  /* ── Grid ─────────────────────────────────────────────────────────── */
  #podcast-list {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 1rem;
    margin-top: 0.5rem;
  }

  /* ── Card ─────────────────────────────────────────────────────────── */
  .pc-card {
    position: relative;   /* contains the stretched pseudo-link */
    display: flex;
    flex-direction: column;
    border-radius: 8px;
    overflow: hidden;
    border: 1px solid rgba(128, 128, 128, 0.25);
    transition: box-shadow 0.2s, transform 0.2s;
  }

  .pc-card:hover {
    box-shadow: 0 4px 18px rgba(0, 0, 0, 0.25);
    transform: translateY(-2px);
  }

  /* ── Stretched link (covers whole card via ::after) ───────────────── */
  .pc-card-anchor {
    position: absolute;
    inset: 0;
    z-index: 1;
  }

  /* ── Image ────────────────────────────────────────────────────────── */
  .pc-img {
    width: 100%;
    height: 160px;
    flex-shrink: 0;
    overflow: hidden;
    line-height: 0;
  }

  /* Chirpy's lightbox JS injects <a class="popup img-link"> around the img —
     it's inline by default, creating a line-height gap at the bottom.
     Force it to be a full-size block so the image fills the container. */
  .pc-img a {
    display: block;
    width: 100%;
    height: 100%;
    line-height: 0;
  }

  .pc-img img {
    display: block;
    width: 100%;
    height: 100%;
    object-fit: cover;
    object-position: center;
    transform: scale(1.08);
    transition: transform 0.3s ease;
    margin: 0;
    padding: 0;
  }

  .pc-card:hover .pc-img img {
    transform: scale(1.14);
  }

  .pc-img-placeholder {
    display: flex;
    align-items: center;
    justify-content: center;
    background: rgba(128, 128, 128, 0.12);
    line-height: normal;
  }

  .pc-img-placeholder i {
    font-size: 2.5rem;
    opacity: 0.3;
  }

  /* ── Body — all text uses CSS vars for light/dark ─────────────────── */
  .pc-body {
    flex: 1;
    display: flex;
    flex-direction: column;
    padding: 0.75rem;
    background: var(--card-bg, var(--main-bg));
  }

  .pc-title {
    font-size: 0.9rem;
    font-weight: 600;
    line-height: 1.35;
    margin: 0 0 0.3rem;
    color: var(--heading-color);
  }

  .pc-card:hover .pc-title {
    color: var(--link-color);
  }

  .pc-date {
    font-size: 0.72rem;
    margin-bottom: 0.3rem;
    color: var(--text-muted-color);
  }

  .pc-desc {
    font-size: 0.8rem;
    flex: 1;
    margin-bottom: 0.5rem;
    color: var(--text-color);
  }

  .pc-meta {
    font-size: 0.75rem;
    margin: 0;
    color: var(--text-muted-color);
  }

  @media (max-width: 480px) {
    #podcast-list {
      grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
    }
    .pc-img { height: 130px; }
  }
</style>
