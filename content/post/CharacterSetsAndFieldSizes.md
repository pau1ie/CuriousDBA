---
title: "Character Sets and Field Sizes"
date: 2022-06-14T16:20:07+01:00
tags: ["Database","Oracle","Fail","Scripting","SQL"]
---

Here is an interesting problem we had recently. A field which should be
length 1 was being reported in some cases to be length 4. How is this possible?


## The Problem

I use `dbms_sql.describe_columns` to discover the size of the field in a
remote database. I use this value to recreate a copy of the view in the
remote database as a table locally. This is used for testing.
We can mimic this in the local database as follows:

```sql
create view length_test as 
  select
    cast(1 as number) id,
	CAST('A' as char(1)) value
  from dual
/
```
Then if we create a file desc.sql containing the following, we can get the 
length of the value column on the view:

```ada
set serveroutput on size 30000
declare
  sql_stmt varchar2(30) := 'select value from length_test';
  c number;
  cntCols number;
  l_descTbl dbms_sql.desc_tab;
begin
  c := dbms_sql.open_cursor;
  dbms_sql.parse(c, sql_stmt, dbms_sql.native);
  dbms_sql.describe_columns(c, cntCols, l_descTbl);

  for i in 1 .. cntCols loop
    dbms_output.put_line('col_name: '||l_descTbl(i).col_name);
    dbms_output.put_line('col_max_len: '||l_descTbl(i).col_max_len);
end loop;
end;
/
```

Let's try it:

```console
sqlplus myuser

SQL> alter session set nls_length_semantics = byte;
SQL> @desc
col_name: VALUE
col_max_len: 1

PL/SQL procedure successfully completed.
```

The first test has `nls_length_semantics` set to byte which will be the case 
in most databases. This reports correctly that the field has a length as 1.
But look what happens if we switch `nls_length_semantics` to char,
which is the default for a PeopleSoft system:

```console
SQL> alter session set nls_length_semantics = char;

Session altered.

SQL> @desc
col_name: VALUE
col_max_len: 3

PL/SQL procedure successfully completed.
```

The length of the field becomes 3. What has happened?

The problem is that we are defining our view ambiguously. We didn't say
whether it was 1 byte or 1 character. So when we change the value of
`nls_length_semantics`, we end up changing the view definition. The database
I am using to test has it's character set  defined as UTF8. A UTF8 database
might require 3 bytes to store one character.

Note that the UTF8 character set is now deprecated. The recommendation these 
days is  to use AL32UTF8 as the database character
set. In that case up to 4 bytes are required to store a Unicode character.

## Solution

The solution is to define our view unambiguously.

```sql
create or replace view length_test as 
  select
    cast(1 as number) id,
	CAST('A' as char(1 byte)) value
  from dual
/
```

Now let's try the describe:

```console
SQL> alter session set nls_length_semantics = byte;

SQL> @desc
col_name: VALUE
col_max_len: 1

PL/SQL procedure successfully completed.

SQL> alter session set nls_length_semantics = char;

Session altered.

SQL>  @desc
col_name: VALUE
col_max_len: 1

PL/SQL procedure successfully completed.
```

That's better. In our case we didn't control the view so had to ensure we 
set `nls_length_semantics` to `byte` before using it.


## Conclusion

It is important to be specific when defining views, especially if they will be
used by interfaces where you don't know in advance what the various NLS 
settings. There are loads of posts on this topic. It's important when
using numeric, date, and now character fields to be specific about
what you mean. Unfortunately it's not always obvious when you are relying
on a default that might change. This is why many languages have linters.
Oracle SQL doesn't seem to have one, so we only tend to find these errors
in production.

