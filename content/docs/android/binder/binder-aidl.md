---
title: "Binder AIDL"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# 第三章 AIDL

在应用端写binder实现的时候，都要写aidl文件的定义，当然并不是必须的，比如native层通常自己实现bn、bp类。那这里解释aidl是如何被编译成Java文件的，以及编译成了什么Java文件，所以后面就可以基于编译后的Java文件再去理解Binder的原理。

