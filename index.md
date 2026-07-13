---
title: 演習の手順
permalink: index.html
layout: home
---

<!--
Change the title above AND IN THE _config.yml FILE (so that the page header in the GitHub Pages site automatically shows the site name).

When you're ready for the labs to be live - make the repo public and enable GitHub Pages publishing from the main branch (in the repo Settings | Pages).
-->

このページには、[Microsoft Learn](https://learn.microsoft.com) の Microsoft スキルアップ コンテンツに関連付けられている演習が記載されています

<!-- You can edit the paragraph above to provide a more specific description and links to content on Learn.

Include the following note if an Azure subscription is required (or add something similar for any other requirements, such as a Microsoft 365 account).

> **Note**: To complete these exercises, you will need a [Microsoft Azure subscription](https://azure.microsoft.com/free) in which you have sufficient permissions to create and configure the required resources.

If a more complex setup is required, create a separate markdown file with setup instructions at \Instructions\Labs\00-setup.md - being sure to include "lab.title"" metadata at the top so it shows up the list below
-->

<hr>

{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %}
{% for activity in labs  %}
{% if activity.lab.title %}
### [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }})


{% if activity.lab.level %}**レベル**: {{activity.lab.level}} \| {% endif %}{% if activity.lab.duration %}**期間**: {{activity.lab.duration}}{% endif %}

{% if activity.lab.description %}
*{{activity.lab.description}}*
{% endif %}
<hr>
{% endif %}
{% endfor %}
