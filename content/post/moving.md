---
title: "Moving"
date: 2019-09-17T16:24:21+01:00
tags: [ "Automation","GitHub","Netlify","Blog"]
---

# Moving

My hosting at the University of Cambridge is being discontinued. I am therefore moving [this blog](https://curiousdba.netlify.com/) to Netlify:


Please update your bookmarks or feed reader to: https://curiousdba.netlify.com/

# Netlify
This wasn't as hard as it might have been. I took the opportunity to upgrade the theme, because GitHub complained about the security of
some of the components. This caused the site not to build. The new version of the theme requires Hugo pipeline, which means it needs a recent version of 
Hugo extended. This won't run (easily) on RHEL7 due to glibcxx being too old. However, Netlify does supply Hugo extended, and allows the user
to pick a version. I needed to upgrade because of a conflict with the theme and the default Hugo version. I picked the latest and my
blog built!

I like the Continuous deployment, I just have to commit a post and the site is rebuilt. If I am working on a draft, Hugo allows me to
use draft:true to ensure the draft won't get posted.

I have done this in a bit of a rush, I would have liked to take some more time to understand the technology, but I needed to move before the
old hosting is discontinued. So I probably haven't done everything in the best way possible, but I am pleased that it all seems to work!

# RSS Feed

The one issue I did have was with the RSS Feed. It only contained one entry, posts. Eventually I realised that this is because posts
are actually all under the post directory. If I pointedt at post/index.xml the rss was what I wanted. I don't understand enough of 
Hugo to fix this, but Netlify does allow redirects. So I just created file static/_redirects with the following content:

```
/index.xml    /post/index.xml                                                                                                                                                                        ```

and any request is redirected. A bit of a hack, but hopefully it works!


