---
title: "Creating Test Data"
date: 2019-05-13T10:45:54+01:00
tags: ['SQL','Automation','plsql','language','oracle','parsing','scripting','testing']
---

Here are some notes as to how to create test data from real data with SQL.

I created a package to do this. It generates SQL statements to mask columns which have personal data, so
we don't end up testing with real peoples data. Here are some things I made it do.

The procedure loops through a list of tables which is in a local table. In this case we are selecting across a
database link. I use a SQL statement to discover the format of the remote table using dbms_sql.parse.

{{< highlight "ada" >}}
    dbms_sql.parse(l_cursor,'Select * from table',dbms_sql.native);
    dbms_sql.describe_columns3(l_cursor, l_colcnt, l_desctab);
{{</highlight>}}

Then I loop through the columns, and build up a select statement in two halves.

For the first half I write _insert into_ followed by the list the columns. I could use *, but it makes it explicit which
column we are talking about. The second half, after the _values_ keyword is more interesting.
If it is not a column I want to mask,
I just append the column name to select the data from the source table. If I want to mask it I have a number
of options.

## Static text

This is the easiest one. Just use the static string instead of the column name ```'string'```

## Text appended with a number

This is pretty simple as well, there are two variants of this. One is to use the sequence number i.e.
`'text' || lpad(rownum,99,'0')`. In this case the sequence number is padded so the row is always
the same length, where 99 is the length of the number. I allow the user to specify the whole field
length then subtract the length of the static text.

Another option is that there is an id in the table. In this case the ID can be appended:
`text' || lpad(id,99,'0')`
This helps the testers to find the test data. We use this for email addresses, so it is easy to find the
test data who the email is about.

## Free text

I defined a string in the package definition to be a chunk of text. This is superior to using random
characters as it will flow properly, though every record will have the same text. It is customary to use
[Lorem Ipsum](https://en.wikipedia.org/wiki/Lorem_ipsum) for this, but my colleague preferred to use the
opening chapters of [Pride and Prejudice](https://www.gutenberg.org/files/1342/1342-h/1342-h.htm#link2HCH0001).
The free text is defined as a sub-string of this text depending on the length of the field. A disadvantage
is that the SQL text can only be 4000 characters long, so it isn't possible to have a 4000 character
text string. However I later discovered a possible solution to this.

## Array

If the field is one of a number of possible values (e.g. codes for sex or sexual orientation) these can
be picked from a list. The SQL becomes something like:

{{< highlight sql >}}
  case trunc(dbms_random.value*4)
    when 0 then 'A',
    when 1 then 'B'
    when 2 then 'C'
    when 3 then 'D'
  end
{{</highlight>}}

This does mean each option will have the same distribution, unlike many real world things (Like sexual
orientation for example)


## Array from reference

Sometimes the above types of data are a list of values, sometimes they are from a table. Still I can
select all possible values from a table and create a case statement as above. Again this will end 
up with an even distribution which I might not want.


## Random section of text

This is where I had a bit of an epiphany. We wanted to model peoples names, and discovered that searching
for a particular person when everyone has the same name is slow. Even if everyone has the same name
with a number appended, the application creates a search record by converting all alphabetical text to
upper case, and stripping everything else including numbers. Again, everyone had the same name.

We decided we would use a random chunk of the text. I didn't want to include the whole text in every
column, and realised I could call a function to return me a chunk of text. So I included a public
function in my PL/SQL code to return a random chunk of text:

{{< highlight "ada" >}}
  function randomstring(p_len number) return varchar2 is
    l_textlen number;
  begin
    l_textlen := lengthb(g_text);
    return substrb(g_text,
                   trunc(dbms_random.value*(l_textlen - p_len)),
                   p_len);
  end randomstring;
{{</highlight>}}

Then I can call this in my SQL once per row, and create generate random strings:

{{< highlight sql >}}
select trim(regexp_replace(owner.package.randomstring(30,'[^[:alnum:]. -]')) from table
{{</highlight>}}

For each row the procedure is called and returns a random chunk of text. The way I wrote it,
it will not always be 30 characters, it might be shorter if any of the invalid characters are
stripped. That doesn't matter in my case and could always be fixed if it does.

## Other possibilities

The idea of calling a procedure means that we have the power of a procedural programming
language, so we could:

- return rows with more realistic distributions.
- return free text columns to avoid having multiple copies of the whole free text in the SQL statement.
- return one of a number of free texts (For example, we could pick a random chapter
from Pride and Prejudice).

If I remember correctly, PL/SQL allows you to remember data in a session,
so the first row could select the values and counts from the remote table, then return values in the
same distribution.

Similarly for names, we could return a random name from a list.

The possibilities are endless!
