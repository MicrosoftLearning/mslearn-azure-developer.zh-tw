---
title: Azure 開發人員練習
permalink: index.html
layout: home
---

## 概觀

下列練習旨在為您提供實作學習體驗，您將探索開發人員在 Microsoft Azure 上建置和部署解決方案時所執行的一般工作。

> **請注意**：若要完成練習，您需要有足夠全線和配額的 Azure 訂用帳戶，才能佈建必要的 Azure 資源。 如果您還沒有 Azure 帳戶，[可以註冊一個 Azure 帳戶](https://azure.microsoft.com/free)。 

有些練習可能有額外或不同需求。 這些將包含該練習的**在開始之前**區段。

## 主題區域
{% assign exercises = site.pages | where_exp:"page", "page.url contains '/instructions'" %} {% assign grouped_exercises = exercises | group_by: "lab.topic" %}

<ul>
{% for group in grouped_exercises %}
<li><a href="#{{ group.name | slugify }}">{{ group.name }}</a></li>
{% endfor %}
</ul>

{% for group in grouped_exercises %}

## <a id="{{ group.name | slugify }}"></a>{{ group.name }} 

{% for activity in group.items %} [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) <br/> {{ activity.lab.description }}

---

{% endfor %} <a href="#overview">返回頂端</a> {% endfor %}

