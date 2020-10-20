---
title: "Tip: validating configured URLs"
categories:
  - Configuration
tags:
  - java
  - configuration
  - uri
  - urL
---

> **TIP** Use java.net.URL to validate absolute URL string,  use java.net.URI to validate relative URI. And to construct your absolute URL using base URL(with the context path) and relative URL, there is still nothing better than concatenating strings :).

{% gist af19530c222df02205366d8f2603d0d5 %}
