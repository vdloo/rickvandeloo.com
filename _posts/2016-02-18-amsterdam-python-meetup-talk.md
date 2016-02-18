---
title: Amsterdam Python Meetup talk about TaskFlow
layout: post
---

Yesterday I did a talk about Openstack TaskFlow at the Amsterdam Python
Meetup.  TaskFlow is a cool Python library that can help you write code
that can deal with unreliable systems (like flaky APIs with calls that
need retrying and/or reverting). It also enables you to continue a flow
(collection of tasks) where it left off if the original worker died with
relative ease. With TaskFlow you can use a jobboard (either based on
Zookeeper or Redis) which makes it very scalable. 

Conductors atomically pick up jobs from the jobboard and record their
state in a persistence backend (Zookeeper, MySQL or some other options),
so at all times the system knows how far a flow has progressed and what
actions still need to happen.

You can find the hosted Jupyter nbviewer slides
[here](https://nbviewer.jupyter.org/format/slides/github/vdloo/python-meetup-taskflow/blob/master/presentation.ipynb#/https://nbviewer.jupyter.org/format/slides/github/vdloo/python-meetup-taskflow/blob/master/presentation.ipynb#/)
and the notebook on GitHub
[here](https://github.com/vdloo/python-meetup-taskflow).

The slides were created by converting a Jupyter (previously named IPython)
notebook to html with jupyter nbconvert:

{% highlight bash %}
jupyter nbconvert presentation.ipynb --to slides
{% endhighlight %}

And cloning the HTML presentation framework
[reveal.js](https://github.com/hakimel/reveal.js/) into the same directory as
the generated .html file.

{% highlight bash %}
git clone https://github.com/hakimel/reveal.js/
{% endhighlight %}
