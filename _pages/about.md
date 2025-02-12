---
permalink: /
title: "Homepage"
excerpt: "About me"
author_profile: true
redirect_from:
  - /about/
  - /about.html
---

I am currently a formal methods scientist at Idaho National Laboratory (INL). I like to call my
research interests *gradual formal methods*: Observe real software
in situ to construct a principled model that drives decision-making. 
I've applied this technique to autonomous vehicle firmware, and more recently
to deep learning frameworks. 

Prior to joining INL, I was a student at The Ohio State University.  My advisors are Drs. Feng Qin and Christopher Stewart.


## Recent Blog Posts
{% include base_path %}
{% capture written_year %}'None'{% endcapture %}
{% assign count = 0 %}
{% for post in site.posts %}
  {% if count == 3 %}
    {% break %}
  {% endif %}
  {% include archive-single.html %}
{% endfor %}
