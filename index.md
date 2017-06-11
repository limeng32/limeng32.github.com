---
layout: default
---

<body>
  <div class="index-wrapper">
    <div class="aside">
      <div class="info-card">
        <h1>my bat is flying</h1>
        <a href="https://github.com/limeng32/mybatis.flying/" target="_blank"><img src="https://github.com/favicon.ico" alt="" width="25"/></a>
        <a href="https://www.zhihu.com/people/li-meng-48/" target="_blank"><img src="https://www.zhihu.com/favicon.ico" alt="" width="22"/></a>
        <a href="https://user.qzone.qq.com/540906853/" target="_blank"><img src="https://cdn3.iconfinder.com/data/icons/social-media/512/QQ-24.png" alt="" width="24"/></a>
      </div>
      <div id="particles-js"></div>
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
