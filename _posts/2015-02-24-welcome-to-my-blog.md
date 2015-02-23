---
layout: post
title:  "Welcome to my blog"
categories: Blogging
tags: Blogging Jekyll Lanyon Markdown
comments: true
analytics: true
---

Welcome to my blog on networking and more!

My name is Fredy Neeser (pronounced "Freddy"; my last name sounds like "Leser" in German) and I live with my family near Zurich, Switzerland.

In my blog, I will address various issues and findings regarding open-source and networking technologies, including network virtualization, SDN and OpenStack. I decided to start my own technical blog to facilitate interactions with open-source communities.

Motivated by Scott Lowe's recent [blog migration story][slowe-blog-migration], my new blogging infrastructure is based on [Jekyll][jekyll]/[Lanyon][lanyon] and [hosted on github pages][fneeser-github].  I greatly appreciate Scott's tips and helpful advice on how to get going with [Markdown][markdown] and Jekyll/Lanyon.

One area I had some difficulties with is how to create nicely displayed figures from my Markdown sources without resorting to in-line HTML.  Jekyll/Lanyon's support for embedding images is pretty basic. However, I wanted support for "responsive" figures with clickable URLs, captions and means to resize and center the images.  

By putting together some helpful tips from the Wild Web, I ended up with an easy-to-use solution for figures in Jekyll.  Here's a demo:

{% include image.html img="public/img/router-arrows.svg" width="50%" title="Router [SVG]" caption="Fig. 1: A nicely drawn router" url="/public/img/router-arrows.svg" %}


I plan to describe my support for figures in an upcoming post.

After playing around with Jekyll for some time, I'm about to get ready for my first technical post.

Stay tuned!


[slowe-blog-migration]: http://blog.scottlowe.org/2015/01/06/the-story-behind-the-migration/
[jekyll]: http://jekyllrb.com/
[lanyon]: http://lanyon.getpoole.com/
[markdown]: http://daringfireball.net/projects/markdown/
[fneeser-github]: https://github.com/fneeser/fneeser.github.io