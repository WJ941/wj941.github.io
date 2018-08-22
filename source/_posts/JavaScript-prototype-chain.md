---
title: JavaScript原型链
date: 2018-08-19 00:19:33
tags:
---

{% asset_img prototype_chain.png %}
构造函数的prototype属性指向一个对象，这个对象的constructor属性指向构造函数，__proto__属性指向对象；
new constructor()生成的对象的__proto__属性指向constructor的prototype属性。