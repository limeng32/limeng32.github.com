---
layout: default
---

<body>
  <div class="index-wrapper">
    <div class="aside">
        <div class="info-card">
        <h1>my bat is flying</h1>
        <h1>
          <img src="https://img.shields.io/badge/license-Apache%202-4EB1BA.svg" alt="" />
          <img src="https://maven-badges.herokuapp.com/maven-central/com.github.limeng32/mybatis.flying/badge.svg" alt="" />
         </h1>
        <a href="https://github.com/limeng32/mybatis.flying/" target="_blank"><img src="https://cdn2.iconfinder.com/data/icons/social-icons-33/128/Github-32.png" alt="" width="24"/></a>
      </div>
      <div id="particles-js">
      </div>
    </div>
    <div class="index-content">
      <ul class="artical-list">
        {% for post in site.categories.blog %}
        <li>
          <a href="{{ site.url }}{{ post.url }}" class="title">{{ post.title }}</a>
          <div class="title-desc">{{ post.description }}</div>
        </li>
        {% endfor %}
      </ul>
    </div>
  </div>
</body>
