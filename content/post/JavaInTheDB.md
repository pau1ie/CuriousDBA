---
title: "Java in the database"
date: 2020-10-02T14:20:49+01:00
tags: ["Oracle","Database","DBA","Cloud","Java","Language","PLSQL","SQL"]
---

## Why start using Java in the database?

I have had a couple of situations recently where using
Java in the database might come in handy. 

One is replacing a self-hosted database with one on the cloud. The provider
gives access to the database, which means remote procedure calls don't
work - the provider doesn't give access to the underlying OS, they provide
the database as a service. This means the remote procedure call will have
to be replaced in some way.

Another is a 19c upgrade. Oracle is de-supporting Multimedia in this version,
and we used it for a single case - to resize images. If we can find a way to
do this in Java, then it might be a good replacement.

## Is it really complicated?
The Java documentation talks about using `loadjava`, to load Java objects into
the database after they have been compiled with `javac`. This works, but you
still need to create the definition in the database, and the code needs to
be written in a way that is suitable for interfacing with the database.

I didn't see a reference to the fact that Java can be
created in exactly the same way as a stored procedure. As an example I will
take a stored procedure that creates some
random bytes. This was done in a C program called as an external procedure.
I think the original programmer
wanted something more random and less predictable than PL/SQL `DBMS_RANDOM`
which the documentation clearly says is pseudo random, and not suitable
for things like
cryptography which require real random numbers.

This code runs, but hasn't been reviewed. I am a DBA rather than a programmer,
so please don't use this in production before getting it reviewed!

```java
create or replace and compile
java source named Nonce as
import java.security.SecureRandom;
import java.util.Base64;
public class Nonce
{
  public static String token64 (int len)
  {
    SecureRandom random = new SecureRandom();
    byte[] bytes = new byte[len];
    random.nextBytes(bytes);
    String encoded = Base64.getEncoder().encodeToString(bytes);
    return encoded;
  }
};
/
```

This gets created in the database just like PL/SQL, you can view the source.
To be able to run it, a wrapper needs to be created:

```sql
create or replace function raven_auth.nonce (len in NUMBER) return VARCHAR2
AS LANGUAGE JAVA
NAME 'Nonce.tokenb64(int) return java.lang.String';
/
```

This does a translation between Java and SQL. Now I can call the function in SQL:

```console
SQL> select nonce(10) from dual;

NONCE(10)
-------------------
cap4zETRIs+PyA==

SQL> 
```

Or from PL/SQL:

```sql
set serveroutput on size 30000;
declare
 str varchar2(4000);
begin
 str := raven_auth.nonce(10);
 dbms_output.put_line(str);
end;
/
```
And Oracle replies:

```console
32Ilf+tebFi3fw==


PL/SQL procedure successfully completed.
```

For small pieces of code, or sites that are used to maintaining PL/SQL I think this is
the best way to introduce Java.

## Limitations

I had hoped that a Java object could be created in PL/SQL and used as an object,
but this doesn't seem to be possible. This has to be done in Java as I do above.

If you don't already have Java in the database (If you have Multimedia I believe you
do because it depends on Java) then you will need to install it, and keep the
patches up to date. This entails longer downtime
with critical patches. Java has a lot of libraries, so security bugs are found fairly
often. Also it is difficult to remove Java from the database once
it is installed (It is probably easier to create a new database and import all the data).
