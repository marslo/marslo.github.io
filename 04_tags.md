---
layout: page
title: tags
permalink: /tags/
order: 4
---

<div class="page-content wc-container">
    <div class="post">
        <h1>Tags</h1>
        <ul>
            {% for tag in site.tags %}
            <li><a href="{{ '/tags/' | append:tag[0] | relative_url }}">{{ tag[0] }}</a></li>
            {% endfor %}
        </ul>
    </div>
</div>
