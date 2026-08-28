---
title: "Blog"
layout: archive
permalink: /blog/
author_profile: true
---

{% include base_path %}

<style>
.post-filter-wrapper > input[type="radio"] {
  position: absolute;
  opacity: 0;
  width: 0;
  height: 0;
}

.post-filter-bar {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin: 1rem 0;
}

.post-filter-pill {
  display: inline-block;
  padding: 0.35rem 0.75rem;
  border-radius: 999px;
  border: 1px solid var(--global-border-color, #ccc);
  font-size: 0.9rem;
  cursor: pointer;
  background-color: var(--global-bg-color, #fff);
  color: var(--global-text-color, #222);
}

.post-filter-pill:hover {
  border-color: var(--global-link-color, #2a5885);
  color: var(--global-link-color, #2a5885);
}

#filter-all:checked ~ .post-filter-bar label[for="filter-all"],
#filter-research-science:checked ~ .post-filter-bar label[for="filter-research-science"],
#filter-workflows-code:checked ~ .post-filter-bar label[for="filter-workflows-code"],
#filter-thoughts-process:checked ~ .post-filter-bar label[for="filter-thoughts-process"],
#filter-homelab-systems:checked ~ .post-filter-bar label[for="filter-homelab-systems"],
#filter-notes-snippets:checked ~ .post-filter-bar label[for="filter-notes-snippets"] {
  background-color: var(--global-link-color, #2a5885);
  color: #fff;
  border-color: var(--global-link-color, #2a5885);
}

/* Category descriptions */
.category-desc {
  display: none;
  margin-bottom: 1rem;
  font-size: 0.95rem;
  color: var(--global-text-color-light, #666);
}

#filter-all:checked ~ .category-desc-all,
#filter-research-science:checked ~ .category-desc-research-science,
#filter-workflows-code:checked ~ .category-desc-workflows-code,
#filter-thoughts-process:checked ~ .category-desc-thoughts-process,
#filter-homelab-systems:checked ~ .category-desc-homelab-systems,
#filter-notes-snippets:checked ~ .category-desc-notes-snippets {
  display: block;
}

/* Filtering logic */
.filterable-posts .post-item {
  display: block;
}

#filter-research-science:checked ~ .filterable-posts .post-item:not(.cat-research-science),
#filter-workflows-code:checked ~ .filterable-posts .post-item:not(.cat-workflows-code),
#filter-thoughts-process:checked ~ .filterable-posts .post-item:not(.cat-thoughts-process),
#filter-homelab-systems:checked ~ .filterable-posts .post-item:not(.cat-homelab-systems),
#filter-notes-snippets:checked ~ .filterable-posts .post-item:not(.cat-notes-snippets) {
  display: none;
}
</style>

I write here roughly once a week (or not) not on a content calendar, but organically, based on problems I actually run into.

<div class="post-filter-wrapper">
  <input type="radio" id="filter-all" name="post-filter" checked><input type="radio" id="filter-research-science" name="post-filter"><input type="radio" id="filter-workflows-code" name="post-filter"><input type="radio" id="filter-thoughts-process" name="post-filter"><input type="radio" id="filter-homelab-systems" name="post-filter"><input type="radio" id="filter-notes-snippets" name="post-filter">

  <div class="post-filter-bar">
    <label for="filter-all" class="post-filter-pill">All</label>
    <label for="filter-research-science" class="post-filter-pill">Research & Science</label>
    <label for="filter-workflows-code" class="post-filter-pill">Workflows & Code</label>
    <label for="filter-thoughts-process" class="post-filter-pill">Thoughts & Process</label>
    <label for="filter-homelab-systems" class="post-filter-pill">Homelab & Systems</label>
    <label for="filter-notes-snippets" class="post-filter-pill">Notes & Snippets</label>
  </div>

  <div class="category-desc category-desc-all">
    Browsing all posts, across every topic: research, workflow debugging, homelab infrastructure, reflections, and quick reference notes.
  </div>
  <div class="category-desc category-desc-research-science">
    Research & Science. Spatial AI, climate data, and the models behind flash drought early warning. This is where research findings, methods, and technical rigor live. The posts most relevant to postdoc committees, collaborators, and fellow researchers.
  </div>
  <div class="category-desc category-desc-workflows-code">
    Workflows & Code. Debugging logs, data pipeline fixes, and the developer workflow decisions that support the research dev environment setup, tooling for research code, and technical problem-solving directly tied to getting real work done.
  </div>
  <div class="category-desc category-desc-thoughts-process">
    Thoughts & Process. Reflections on academic life, research process, and the occasional opinion. The authentic voice behind the CV.
  </div>
  <div class="category-desc category-desc-homelab-systems">
    Homelab & Systems. Self-hosted infrastructure, containers, and networking. The systems and architecture work behind my homelab, including Tailscale setups, OS-level configuration on immutable/atomic Linux distros, and self-hosted service deployments.
  </div>
  <div class="category-desc category-desc-notes-snippets">
    Notes & Snippets. The small stuff I look up again later so I don't have to re-ask an AI or re-search the same problem twice. Not curated for a specific audience; mostly here for me.
  </div>

  <div class="filterable-posts">
    {% capture written_year %}'None'{% endcapture %}
    {% for post in site.posts %}
      {% capture year %}{{ post.date | date: '%Y' }}{% endcapture %}
      {% if year != written_year %}
        <h2 id="{{ year | slugify }}" class="archive__subtitle">{{ year }}</h2>
        {% capture written_year %}{{ year }}{% endcapture %}
      {% endif %}

    {% assign post_category = post.categories[0] %}
    <div class="post-item cat-{{ post_category }}">
      <article class="archive__item" itemscope itemtype="http://schema.org/CreativeWork">
        <h2 class="archive__item-title" itemprop="headline">
          <a href="{{ base_path }}{{ post.url }}" rel="permalink">{{ post.title | markdownify | remove: "<p>" | remove: "</p>" }}</a>
        </h2>

        {% if post.date %}
          <p class="page__date"><strong><i class="fa fa-fw fa-calendar" aria-hidden="true"></i> Published:</strong> <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %d, %Y" }}</time></p>
        {% endif %}

        {% if post.header.teaser %}
          <div class="archive__item-teaser" style="max-width: 500px;">
            <img src="{% if post.header.teaser contains '://' %}{{ post.header.teaser }}{% else %}{{ post.header.teaser | prepend: "/images/" | prepend: base_path }}{% endif %}" style="width: 100%; height: auto; display: block;" alt="">
          </div>
        {% endif %}

        {% if post.excerpt %}
          <p class="archive__item-excerpt" itemprop="description">{{ post.excerpt | markdownify }}</p>
        {% endif %}
      </article>
    </div>
    {% endfor %}
  </div>
</div>