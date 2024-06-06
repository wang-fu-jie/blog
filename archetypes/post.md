---
title:       {{ replace (substr .File.ContentBaseName 11) "-" " " | title }}
subtitle:    ""
description: ""
excerpt:     ""
date:        {{ .Date }}
author:      "王富杰"
image:       "/img/home-bg-jeep.jpg"
published:   true
tags:
    - tag1
    - tag2
slug:        "{{substr .File.ContentBaseName 11}}"
categories:  [ "PYTHON" ]
---