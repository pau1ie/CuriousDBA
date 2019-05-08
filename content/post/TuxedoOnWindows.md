---
title: "Tuxedo On Windows"
date: 2018-07-09T14:47:59+01:00
draft: true
---

Playing with installs in slightly different ways means that eventually I will get the VM in a state where it can't
uninstall itself. In these cases I tend just to remove everything from the peoplesoft homes and run the installer
again. This works well on Unix.

On Windows, the tuxedo installer keeps its inventory in C:\Program Files\Oracle\Inventory. So if the home is removed
by hand, it also needs to be removed from the inventory.



