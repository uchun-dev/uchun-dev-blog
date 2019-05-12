---
layout: post
title:  "Hello world"
author: uchun
categories: [ android, kotlin ]
image: assets/images/covers/sms.jpg
---

### Hello World

```kotlin
data class Person (
  val name: String = "Bob"
)

interface Hello {
  fun hi(): Unit
}

class Situation {
  init {
    Person().hi()
  }
}
```
