---
layout: page
title: Tags

---

<div class="page-content wc-container">
	<div class="post">
		<h1>Tags</h1>
		{% assign tags = (site.tags | sort:0) %}
		{% for tag in tags %}
		<span class="tag" >
			<a href="#{{ tag[0] | cgi_escape }}">{{tag[0]}}</a> ({{ tag[1].size }})
		</span>
		{% endfor %}
		<hr>
		{% for tag in tags %}
		<h3 id="{{ tag[0] | cgi_escape }}" class="offset">{{ tag[0] }}</h3>
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
