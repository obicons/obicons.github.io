---
permalink: /
title: "Homepage"
excerpt: "About me"
author_profile: true
redirect_from:
  - /about/
  - /about.html
---

I am an assistant professor at Boise State University.
My research interests are at the intersection of cybersecurity, software engineering, and formal methods.
Society depends on physical systems (e.g., critical infrastructure like water treatment plants, robotic systems) that are increasingly integrated with cyber controls that contain vulnerabilities.
These vulnerabilities should be eliminated by improving the state-of-the-art of software engineering.
This can be accomplished by leveraging results from formal methods, a field that uses mathematically rigorous techniques to analyze systems, to improve our standards of practice.

Prior to joining Boise State, I was a formal methods scientist at Idaho National Laboratory, where I maintain a joint appointment.
Before Idaho National Laboratory, I was a student at The Ohio State University.
My advisors are Drs. Feng Qin and Christopher Stewart.


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
