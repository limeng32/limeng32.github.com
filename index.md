---
layout: default
---

<body>
  <div class="index-wrapper">
    <div class="aside">
        <div class="info-card">
        <h1>my bat is flying</h1>
        <a href="https://github.com/limeng32/mybatis.flying/" target="_blank"><img src="https://cdn2.iconfinder.com/data/icons/social-icons-33/128/Github-32.png" alt="" width="24"/></a>
        <a href="https://www.zhihu.com/people/li-meng-48/" target="_blank"><img src="https://cdn4.iconfinder.com/data/icons/chinas-social-share-icons/256/cssi_zhihu-32.png" alt="" width="24"/></a>
        <a href="/wangwang/" target="_blank"><img src="https://cdn4.iconfinder.com/data/icons/chinas-social-share-icons/256/cssi_wangwang-32.png" alt="" width="24"/></a>
        <a href="https://user.qzone.qq.com/540906853/" target="_blank"><img src="https://cdn4.iconfinder.com/data/icons/flat-brand-logo-2/512/qq-32.png" alt="" width="24"/></a>
      </div>
      <div id="particles-js"></div>
    </div>

    <div class="index-content">
    <img src="https://travis-ci.org/limeng32/mybatis.flying.svg?branch=master" alt=""/>
        <img src="https://codecov.io/gh/limeng32/mybatis.flying/branch/master/graph/badge.svg" alt="" />
        <img src="https://maven-badges.herokuapp.com/maven-central/com.github.limeng32/mybatis.flying/badge.svg" alt="" />
        <img src="https://img.shields.io/badge/license-Apache%202-4EB1BA.svg" alt="" />
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
