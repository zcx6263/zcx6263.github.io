---
layout: post
title: "[Git] 깃허브 보안 취약점 알림 메일"
subtitle: "We found a potential security vulnerability in one of your dependencies."
comments: true
categories : Git
date: 2020-05-08
background: '/img/posts/mac.png'
---

<h2 class="section-heading">깃허브 보안 취약점 알림 메일 </h2>

<p>We found a potential security vulnerability in one of your dependencies.</p>

<p>위의 내용으로 계속해서 메일이 오고 Github 화면에서 계속해서 나타나서 확인 해보니 
package.json을 확인해보니 패키지 내용에도 없었기 때문에 내가 사용하고 있는 패키지 중에 
무언가 의존하고 있는 것 같다. 해결 방법은 아래와 같다.</p>

```
// npm 최신 버전으로 업데이트
$ npm i -g npm

// 보안 취약성 확인 (해당 폴더로 이동)
$ npm audit

// 업데이트 (해당 폴더에서)
$ npm audit fix
```

<p>Reference</p>
<a href="https://jiggag.github.io/github-dependency/">https://jiggag.github.io/github-dependency/</a>

{% highlight ruby linenos %}


{% endhighlight %}


{%- if site.disqus.shortname -%}
    {%- include disqus.html -%}
{%- endif -%}
