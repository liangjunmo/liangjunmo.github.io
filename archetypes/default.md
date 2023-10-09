---
title: "{{ replace .Name "-" " " | title}}"
slug: "{{ .Name }}"
type: post
date: {{ dateFormat "2006-01-02" .Date }}
lastMod: {{ dateFormat "2006-01-02" .Date }}
showSummary: false
summary:
---

