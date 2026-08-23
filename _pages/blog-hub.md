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
  pointer-events: none;
}
.post-filter-bar {
  margin: 1.5em 0;
}
.post-filter-bar label {
  display: inline-block;
  padding: 0.4em 0.9em;
  margin: 0 0.4em 0.5em 0;
  border: 1px solid var(--global-border-color, #ccc);
  border-radius: 999px;
  font-size: 0.85em;
  color: var(--global-text-color, #333);
  background-color: var(--global-bg-color, #fff);
  cursor: pointer;
  transition: background-color 0.15s ease, color 0.15s ease, border-color 0.15s ease;
}
.post-filter-bar label:hover {
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
  color: var(--global-bg-color, #fff);
  border-color: var(--global-link-color, #2a5885);
}

/* Category description reveal: hidden by default, shown when its filter is active */
.category-desc {
  display: none;
  margin: 0.8em 0 1.5em 0;
  padding: 0.8em 1em;
  border-left: 3px solid var(--global-border-color, #ccc);
  color: var(--global-text-color-light, #555);
  font-size: 0.95em;
}
#filter-all:checked ~ .category-desc[data-cat="all"],
#filter-research-science:checked ~ .category-desc[data-cat="research-science"],
#filter-workflows-code:checked ~ .category-desc[data-cat="workflows-code"],
#filter-thoughts-process:checked ~ .category-desc[data-cat="thoughts-process"],
#filter-homelab-systems:checked ~ .category-desc[data-cat="homelab-systems"],
#filter-notes-snippets:checked ~ .category-desc[data-cat="notes-snippets"] {
  display: block;
}

/* Post filtering by category */
#filter-research-science:checked ~ .filterable-posts .post-item:not(.cat-research-science) { display: none; }
#filter-workflows-code:checked ~ .filterable-posts .post-item:not(.cat-workflows-code) { display: none; }
#filter-thoughts-process:checked ~ .filterable-posts .post-item:not(.cat-thoughts-process) { display: none; }
#filter-homelab-systems:checked ~ .filterable-posts .post-item:not(.cat-homelab-systems) { display: none; }
#filter-notes-snippets:checked ~ .filterable-posts .post-item:not(.cat-notes-snippets) { display: none; }

@media (max-width: 600px) {
  .post-filter-bar label {
    padding: 0.5em 0.8em;
    font-size: 0.8em;
  }
}
.post-tag {
  display: inline-block;
  padding: 0.15em 0.6em;
  border-radius: 4px;
  font-size: 0.78em;
  font-weight: 600;
  letter-spacing: 0.02em;
  margin-right: 0.5em;
}
.post-tag--pillar {
  background-color: var(--global-link-color, #2a5885);
  color: var(--global-bg-color, #fff);
}
.post-tag--reference {
  background-color: var(--global-border-color, #e8e8e8);
  color: var(--global-text-color-light, #666);
  font-weight: 500;
}
</style>

I write here roughly once a week (or not) not on a content calendar, but organically, based on problems I actually run into.

<div class="post-filter-wrapper">
<input type="radio" name="post-filter" id="filter-all" checked><input type="radio" name="post-filter" id="filter-research-science"><input type="radio" name="post-filter" id="filter-workflows-code"><input type="radio" name="post-filter" id="filter-thoughts-process"><input type="radio" name="post-filter" id="filter-homelab-systems"><input type="radio" name="post-filter" id="filter-notes-snippets"><div class="post-filter-bar"><label for="filter-all">All</label><label for="filter-research-science">Research & Science</label><label for="filter-workflows-code">Workflows & Code</label><label for="filter-thoughts-process">Thoughts & Process</label><label for="filter-homelab-systems">Homelab & Systems</label><label for="filter-notes-snippets">Notes & Snippets</label></div><div class="category-desc" data-cat="all">Browsing all posts, across every topic: research, workflow debugging, homelab infrastructure, reflections, and quick reference notes.</div><div class="category-desc" data-cat="research-science">Spatial AI, climate data, and the models behind flash drought early warning. This is where research findings, methods, and technical rigor live. The posts most relevant to postdoc committees, collaborators, and fellow researchers.</div><div class="category-desc" data-cat="workflows-code">Debugging logs, data pipeline fixes, and the developer workflow decisions that support the research dev environment setup, tooling for research code, and technical problem-solving directly tied to getting real work done.</div><div class="category-desc" data-cat="thoughts-process">Reflections on academic life, research process, and the occasional opinion. The authentic voice behind the CV.</div><div class="category-desc" data-cat="homelab-systems">Self-hosted infrastructure, containers, and networking. The systems and architecture work behind my homelab, including Tailscale setups, OS-level configuration on immutable/atomic Linux distros, and self-hosted service deployments.</div><div class="category-desc" data-cat="notes-snippets">Quick personal fixes and reference notes. The small stuff I look up again later so I don't have to re-ask an AI or re-search the same problem twice. Not curated for a specific audience; mostly here for me.</div><div class="filterable-posts">
{% assign category_labels = "research-science:Research & Science|workflows-code:Workflows & Code|thoughts-process:Thoughts & Process|homelab-systems:Homelab & Systems|notes-snippets:Notes & Snippets" | split: "|" %}
{% assign pillar_categories = "research-science,workflows-code,thoughts-process" | split: "," %}
{% assign sorted_posts = site.posts | sort: 'date' | reverse %}
{% for post in sorted_posts %}
  {% assign post_category = post.categories[0] %}
  {% assign label_display = post_category %}
  {% for pair in category_labels %}
    {% assign parts = pair | split: ":" %}
    {% if parts[0] == post_category %}
      {% assign label_display = parts[1] %}
    {% endif %}
  {% endfor %}
  {% assign is_pillar = false %}
  {% for pc in pillar_categories %}
    {% if pc == post_category %}
      {% assign is_pillar = true %}
    {% endif %}
  {% endfor %}
  <div class="archive__item post-item cat-{{ post_category }}">
    <p class="page__meta" style="margin-bottom: 0.3em;">
      {% if is_pillar %}<span class="post-tag post-tag--pillar">{{ label_display }}</span>{% else %}<span class="post-tag post-tag--reference">{{ label_display }}</span>{% endif %}
      <span>{{ post.date | date: "%b %-d, %Y" }}</span>
    </p>
    <h2 class="archive__item-title" style="margin-top: 0;">
      <a href="{{ base_path }}{{ post.url }}">{{ post.title }}</a>
    </h2>
    {% if post.excerpt %}
      <p class="archive__item-excerpt">{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
    {% endif %}
  </div>
{% endfor %}
</div>
</div>
