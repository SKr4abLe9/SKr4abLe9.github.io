---
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
date: {{ .Date }}
slug: {{ substr .File.UniqueID 0 7 }}
draft: false
description:
categories: ["Markdown"]
tags: ["Markdown"]
cover:
---
