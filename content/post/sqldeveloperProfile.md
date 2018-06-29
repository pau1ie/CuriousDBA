---
title: "SqldeveloperProfile"
date: 2018-04-10T16:13:53+01:00
draft: true
---

To stop sql developer filling up the windows profile, just set the variable:

IDE_USER_DIR=C:\Somewhere\else

Then, copy the .sqldeveloper directory from your profile to the location you picked. When SQLDeveloper
is restarted, everything should just work.

And everything is good. Thanks to 
[Gary Graham in the Oracle Community](/home/psh35/blog/CuriousDBA/content/post/sqldeveloperProfile.md)
