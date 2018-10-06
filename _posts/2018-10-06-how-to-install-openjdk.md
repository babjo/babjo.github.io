---
layout: post
title:  "OpenJDK11 설치하기"
date:   2018-10-06 14:50:00 +0900
categories: learning-tech
tags: [java, open-jdk11, OpenJDK11 설치하기, How to install openjdk11, oraclejdk]
---

```bash
curl https://download.java.net/java/ga/jdk11/openjdk-11_osx-x64_bin.tar.gz \
 | tar -xz \
 && sudo mv jdk-11.jdk /Library/Java/JavaVirtualMachines
```

root 권한은 java system folder 로 쉽게 옮기기 위함이니 걱정말고 Password 넣어주시면 됩니다.

#### 참고
- https://stackoverflow.com/questions/52524112/how-do-i-install-openjdk-java-11-on-mac-osx-allowing-version-switching