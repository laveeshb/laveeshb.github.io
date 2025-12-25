---
layout: default
title: Blog
permalink: /blog/
---

<div class="blog-container">
    <h1>Blog</h1>
    
    {% if site.posts.size > 0 %}
    <div class="posts-list">
        {% assign sorted_posts = site.posts | sort: "order" | reverse %}
        {% for post in sorted_posts %}
        <article class="post-preview">
            <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
            <div class="post-meta">
                <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %-d, %Y" }}</time>
                {% if post.categories.size > 0 %}
                <span class="categories">
                    {% for category in post.categories %}
                    <span class="category">{{ category }}</span>
                    {% endfor %}
                </span>
                {% endif %}
            </div>
            <p class="excerpt">{{ post.excerpt | strip_html | truncate: 200 }}</p>
            <a href="{{ post.url | relative_url }}" class="read-more">Read more &rarr;</a>
        </article>
        {% endfor %}
    </div>

    {% if paginator.total_pages > 1 %}
    <nav class="pagination">
        {% if paginator.previous_page %}
        <a href="{{ paginator.previous_page_path | relative_url }}" class="prev">&larr; Newer</a>
        {% endif %}
        <span class="page-number">Page {{ paginator.page }} of {{ paginator.total_pages }}</span>
        {% if paginator.next_page %}
        <a href="{{ paginator.next_page_path | relative_url }}" class="next">Older &rarr;</a>
        {% endif %}
    </nav>
    {% endif %}

    {% else %}
    <div class="no-posts">
        <p>No posts yet. Check back soon!</p>
    </div>
    {% endif %}
</div>
