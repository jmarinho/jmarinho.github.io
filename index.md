# Z Blg

<ul>
{% for entry in site.entries %}
  <li>
    {{ entry.date | date: "%Y-%m-%d" }} —
    <a href="{{ entry.url }}">{{ entry.title }}</a>
  </li>
{% endfor %}
</ul>
