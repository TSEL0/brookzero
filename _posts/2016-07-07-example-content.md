---
layout: post
title: 1
---

刚发布完Xcode的8.0果断更新了，发现用起来非常容易闪退，关键是我编辑项目时默认使用Xcode8打开，导致我用Xcode7打开Xib是报错：

-----

{% highlight c %}
This version does not support documents saved in the Xcode 8 format. Open this document with Xcode 8.0 or later.
{% endhighlight %}

导致用Xcode8打开的Xib全部打不开，只能用编辑器将Xib里面的下面一句话删除掉才能打开：

{% highlight c %}
<capability name="documents saved in the Xcode 8 format" minToolsVersion="8.0"/>
{% endhighlight %}

