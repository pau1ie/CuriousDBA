---
title: "ForeignKeys"
date: 2017-09-08T11:02:13+01:00
draft: true
---

Here is another one to remind myself. I tend to work with delivered
applications which don't use foreign keys, so when they are used by
developers I need to look up what is going on.

Here is a script to loate foreign keys where a table is the child or
parent

{{< highlight sql >}}
SELECT
    A.TABLE_NAME table_name,
    A.CONSTRAINT_NAME key_name,
    B.TABLE_NAME referencing_table,
    B.CONSTRAINT_NAME foreign_key_name,
    B.STATUS fk_status
  FROM dba_CONSTRAINTS A, dba_CONSTRAINTS B
  WHERE a.owner = '&1' and b.owner = a.owner and
    A.CONSTRAINT_NAME = B.R_CONSTRAINT_NAME and
    B.CONSTRAINT_TYPE = 'R'
    and ( b.table_name = '&&2'
     or b.table_name = '&&2')
  ORDER BY 1, 2, 3, 4
{{< /highlight >}}

And here is a script to locate foreign keys with missing indexes. It is
adapted from this [ask tom answer](https://asktom.oracle.com/pls/asktom/f?p=100:11:0%3a%3a%3a%3aP11_QUESTION_ID:4530093713805#26568859366976)

{{< highlight sql >}}
SELECT owner,
  table_name,
  constraint_name,
  cname1
  || nvl2(cname2,','
  ||cname2,NULL)
  || nvl2(cname3,','
  ||cname3,NULL)
  || nvl2(cname4,','
  ||cname4,NULL)
  || nvl2(cname5,','
  ||cname5,NULL)
  || nvl2(cname6,','
  ||cname6,NULL)
  || nvl2(cname7,','
  ||cname7,NULL)
  || nvl2(cname8,','
  ||cname8,NULL) columns
FROM
  (SELECT b.owner,
    b.table_name,
    b.constraint_name,
    MAX(DECODE( position, 1, column_name, NULL )) cname1,
    MAX(DECODE( position, 2, column_name, NULL )) cname2,
    MAX(DECODE( position, 3, column_name, NULL )) cname3,
    MAX(DECODE( position, 4, column_name, NULL )) cname4,
    MAX(DECODE( position, 5, column_name, NULL )) cname5,
    MAX(DECODE( position, 6, column_name, NULL )) cname6,
    MAX(DECODE( position, 7, column_name, NULL )) cname7,
    MAX(DECODE( position, 8, column_name, NULL )) cname8,
    COUNT(*) col_cnt
  FROM
    (SELECT owner,
      SUBSTR(table_name,1,30) table_name,
      SUBSTR(constraint_name,1,30) constraint_name,
      SUBSTR(column_name,1,30) column_name,
      position
    FROM dba_cons_columns
    ) a,
    dba_constraints b
  WHERE a.constraint_name = b.constraint_name
  AND a.owner             = b.owner
  AND b.constraint_type   = 'R'
  GROUP BY b.owner,
    b.table_name,
    b.constraint_name
  ) cons
WHERE col_cnt > ALL
  (SELECT COUNT(*)
  FROM dba_ind_columns i
  WHERE i.table_name     = cons.table_name
  AND i.index_owner      = cons.owner
  AND i.column_name     IN (cname1, cname2, cname3, cname4, cname5, cname6, cname7, cname8 )
  AND i.column_position <= cons.col_cnt
  GROUP BY i.index_name
  ) 
{{< /highlight >}}

