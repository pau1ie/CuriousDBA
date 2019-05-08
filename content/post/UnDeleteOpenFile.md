---
title: "Un-Delete Open file"
date: 2018-09-07T11:56:26+01:00
tags: ['Backup','Disaster Recovery','DR','Fail','Recovery','Linux' ]
---

There is a way to un-delete a file if a program has it open in Linux.

## How directory entries work in Unix

When you remove a file on Linux, what actually happens is that the file is unlinked. Lets have a look. I start with a clean directory:

~~~console
$ ls -la
total 4
drwxrwxr-x  2 me me    6 Sep  4 15:58 .
drwxrwxr-x 14 me me 4096 Sep  4 15:28 ..
~~~

The directory has two entries, the current directory and the parent directory. The second entry, the number (2 and 14 above), is the number of links to
the file. The current directory has 2 links, one for it's entry in itself '.' and one for its entry in the parent directory. If we 
create a file, it has one link:

~~~console
$ echo "Hello unlink" > a
$ ls -la
total 4
drwxrwxr-x  2 me me   15 Sep  4 15:58 .
drwxrwxr-x 14 me me 4096 Sep  4 15:28 ..
-rw-rw-r--  1 me me   13 Sep  4 15:58 a
~~~

I can create a hard link and a symbolic link to that file.

~~~console
$ ln a b
$ ln -s a c
$ ls -la
total 4
drwxrwxr-x  2 me me   33 Sep  4 15:58 .
drwxrwxr-x 14 me me 4096 Sep  4 15:28 ..
-rw-rw-r--  2 me me   13 Sep  4 15:58 a
-rw-rw-r--  2 me me   13 Sep  4 15:58 b
lrwxrwxrwx  1 me me    1 Sep  4 15:58 c -> a
~~~

The hard link is increases the number of links to 2. The symbolic link doesn't, it is a pointer to the file so it doesn't add to the number of links
If I remove file _b_, we can see the number of links to a is reduced. This is why removing a file is called unlinking it.

~~~console
$ ls -l
total 0
-rw-rw-r-- 1 me me 13 Sep  4 15:58 a
lrwxrwxrwx 1 me me  1 Sep  4 15:58 c -> a
~~~

## Opening a file

In Unix, people are fond of saying everything is a file. A really nice thing is that the internals of the operating system are
exposed to the user as files, and can be viewed as files. In Linux we can view information about processes using the information in
/proc. This contains a load of directories which are numbers. These are process ids and contain information about that process. In the fd
directory of each process is a list of the file descriptors.

So if I do:
~~~console
cat >> a
~~~

I have opened my file for writing. I can have a look in the */proc/nn/fd* directory, where *nn* is  the process id of the cat command, I see:

~~~console
lrwx------ 1 me me 64 Sep  4 16:22 0 -> /dev/pts/96
l-wx------ 1 me me 64 Sep  4 16:22 1 -> /home/me/tmp/unlinks/a
lrwx------ 1 me me 64 Sep  4 16:22 2 -> /dev/pts/96
~~~

The cat command has three file descriptors. 0 is stdin,, 1 is stdout and 2 is stderr. I redirected stdout to the file a.
The file is open in the process. This means if I delete the file from disc, there is still a link to the file in memory.


## Deleting the file

Let's see what happens:

~~~console
$ rm a
$ ls -l 
$ rm a
total 0
lrwxrwxrwx 1 me me 1 Sep  4 15:58 c -> a
~~~

My file is gone, and I don't have a backup. Oh no! To add insult to injury, the symbolic link has gone red and is flashing at me.

~~~console
$ cat c
cat: c: No such file or directory
~~~

## Getting the contents back

File *c* does exist, but file *a*, which it points to, is gone. There are no links to the file on disc, but the file is still on disc.
My *cat* command in the other session still has a link to the file in memory. If I could copy that back to the disc, everything
would be good.

Lets have a look at the */proc/nn/fd* directory again:

~~~console
$ ls -l
lrwx------ 1 me me 64 Sep  7 11:22 0 -> /dev/pts/96
l-wx------ 1 me me 64 Sep  7 11:22 1 -> /home/me/tmp/unlinks/a (deleted)
lrwx------ 1 me me 64 Sep  7 11:22 2 -> /dev/pts/96
$ cat 1
Hello unlink
$
~~~

It tells me the file is deleted. But I can still see it. I can copy the file back.

## Can the link be recreated?


What if the file is enormous and there isn't enough space on
the disc, is there a way to recreate the link? 

~~~console
$ ln 1 /home/me/tmp/unlinks/a
ln: failed to create hard link ‘/home/me/tmp/unlinks/a’ => ‘1’: Invalid cross-device link
~~~

Apparently not, or at least, not with the _ln_ command. I am sure if one knew the format of directory entries on disc
it would be possible to insert one. It would be interesting to do, but would never get past change control.
 It would be easier to buy a USB flash drive and get someone to plug it into
the server and copy the file that way. 

## Conclusion

If a deleted file is open in a program, it is possible to get the contents back pretty easily.
