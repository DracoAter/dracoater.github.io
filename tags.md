---
layout: page
title: Tags

---

<div class="page-content wc-container">
	<div class="post">
		<h1>Tags</h1>
		{% assign sorted_tags = site.tags | sort %}
		{% for tag in sorted_tags %}
		<span class="tag">
			<i class="las la-tag"></i>
			<a href="/tags#{{ tag[0] | uri_escape }}">{{ tag[0] }}</a> ({{ tag[1].size }})
		</span>
		{% endfor %}
		<hr>
		{% for tag in sorted_tags %}
		<h3 id="{{ tag[0] }}" class="offset">{{ tag[0] }}</h3>
		<ul>
			{% for post in tag[1] %}
			<li>
				<a href="{{ post.url }}">{{ post.title }}</a>
				<small>{{ post.date | date_to_string }}</small>
			</li>
			{% endfor %}
		</ul>
		{% endfor %}
	</div>
</div>
