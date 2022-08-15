---
permalink: /
title: "Homepage"
excerpt: "About me"
author_profile: true
redirect_from:
  - /about/
  - /about.html
---

I am a fifth year PhD candidate in The Ohio State University's CSE department, where I help developers understand challenges faced by emerging technology & I create tools to help developers build reliable software systems. Currently, I study reliability challenges in autonomous vehicle firmware. My advisors are Drs. Feng Qin and Christopher Stewart.


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
