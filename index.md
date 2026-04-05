# Z Blg

{% assign sorted = site.entries | sort: "date" | reverse %}

<ul>
{% for entry in sorted %}
  <li>
    {{ entry.date | date: "%Y-%m-%d" }} —
    <a href="{{ entry.url }}">{{ entry.title }}</a>
  </li>
{% endfor %}
</ul>
