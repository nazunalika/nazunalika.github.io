---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

Welcome to my notes

{% assign date = '2021-03-19T06:10:00Z' %}

- Original date - {{ date }}
- With timeago filter - {{ date | timeago }}
