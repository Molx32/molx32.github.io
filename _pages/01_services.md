---
layout: page
title: titles.services
permalink: /services/
description: descriptions.services
nav: true
nav_order: 2
display_categories: [pentest, audit, conseil]
horizontal: false
---

<!-- pages/projects.md -->
<div class="projects">
{%- if site.enable_project_categories and page.display_categories %}
  <!-- Display categorized projects -->
  {%- for category in page.display_categories %}
  <h2 class="category">{{ category }}</h2>
  {%- assign categorized_services = site.services | where: "category", category -%}
  {%- assign sorted_services = categorized_services %}
  <!-- Generate cards for each project -->
  {% if page.horizontal -%}
  <div class="container">
    <div class="row row-cols-2">
    {%- for service in sorted_services -%}
      {% include services_horizontal.html %}
    {%- endfor %}
    </div>
  </div>
  {%- else -%}
  <div class="grid">
    {%- for service in sorted_services -%}
      {% include services.html %}
    {%- endfor %}
  </div>
  {%- endif -%}
  {% endfor %}

{%- else -%}
<!-- Display projects without categories -->
  {%- assign sorted_services = site.services -%}
  <!-- Generate cards for each project -->
  {% if page.horizontal -%}
  <div class="container">
    <div class="row row-cols-2">
    {%- for service in sorted_services -%}
      {% include services_horizontal.html %}
    {%- endfor %}
    </div>
  </div>
  {%- else -%}
  <div class="grid">
    {%- for service in sorted_services -%}
      {% include service.html %}
    {%- endfor %}
  </div>
  {%- endif -%}
{%- endif -%}
</div>
