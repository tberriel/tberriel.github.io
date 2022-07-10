---
layout: page
permalink: /publications/
title: publications

description: Still no paper accepted for publication. I am working on it :)
years: [2022]
nav: false
nav_order: 1
---
<!-- _pages/publications.md -->
<div class="publications">

{%- for y in page.years %}
  <h2 class="year">{{y}}</h2>
  {% bibliography -f papers -q @*[year={{y}}]* %}
{% endfor %}

</div>
