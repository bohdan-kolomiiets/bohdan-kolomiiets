---
layout: post
title: "Deferrable constratins in PostgreSQL"
permalink: /postgres/deferrable-constraints/
---

## How to update primary key with related foreign keys using deferred constraints feature of PostgreSQL ?

[https://www.postgresql.org/docs/current/sql-altertable.html](https://www.postgresql.org/docs/current/sql-altertable.html)

[https://www.postgresql.org/docs/current/sql-set-constraints.html](https://www.postgresql.org/docs/current/sql-set-constraints.html)

When constraint is created, it can be given with on of three possble characteristics:

1. `NOT DEFERRABLE` (by default, cannot be altered)
2. `DEFERRABLE INITIALLY DEFERRED` (can be altered, initially it checks at the end of transaction)
3. `DEFERRABLE INITIALLY IMMEDIATE` (can be altered, initially it checks each statement) 

But later, these characteristics could be altered for the during transaction scope. Except 1st class `NOT DEFERRABLE` (behavior of such a constraint cannot be altered). 

Currently, only `UNIQUE`, `PRIMARY KEY`, `REFERENCES` (foreign key), and `EXCLUDE` constraints are affected by this setting. `NOT NULL` and `CHECK` constraints are always checked immediately when a row is inserted or modified.

Lets take a look at few cases to understand how in works by examples.

**Default (not deferrable) case**

```
drop table if exists a cascade;
create table if not exists a (id int primary key);
drop table if exists b;
create table if not exists  b (
  id int primary key, a_id int,
  constraint b_a_id foreign key (a_id) references public.a (id) **-- not deferrable**
);

begin;
**set constraints b_a_id deferred;** -- error: constraint "b_a_id" is not deferrable
insert into b (id, a_id) values (1, 1); 
insert into a (id) values (1);
commit;

```

**`deferrable initially immediate` case**

```
drop table if exists a cascade;
create table if not exists a (id int primary key);
drop table if exists b;
create table if not exists  b (
  id int primary key, a_id int,
  constraint b_a_id foreign key (a_id) references public.a (id) **deferrable initially immediate**
);

begin;
insert into b (id, a_id) values (1, 1); -- error: insert or update on table "b" violates foreign key constraint "b_a_id" 
insert into a (id) values (1);
rollback;
```

**`set constraints {name} deferred` case**

```
drop table if exists a cascade;
create table if not exists a (id int primary key);
drop table if exists b;
create table if not exists  b (
  id int primary key, a_id int,
  constraint b_a_id foreign key (a_id) references public.a (id) **deferrable initially immediate**
);

begin;
**set constraints b_a_id deferred;**
insert into b (id, a_id) values (1, 1); -- success: despite the fact that a.id with value 1 is not present yet, this command will not fail inside transaction
insert into a (id) values (1);
commit;
```

`**deferrable initially deferred` case**

```
drop table if exists a cascade;
create table if not exists a (id int primary key);
drop table if exists b;
create table if not exists  b (
  id int primary key, a_id int,
  constraint b_a_id foreign key (a_id) references public.a (id) **deferrable initially deferred**
);

begin;
insert into b (id, a_id) values (1, 1); -- success: constraint is deferred by default, not need to alter it behavior
insert into a (id) values (1);
commit;
```

**We can temporary make constraint deferrable**

```
drop table if exists a cascade;
create table if not exists a (id int primary key);
drop table if exists b;
create table if not exists  b (
  id int primary key, a_id int,
  constraint b_a_id foreign key (a_id) references public.a (id) **-- not deferrable by default** 
);

**alter table b alter constraint b_a_id deferrable initially immediate;**

begin;
**set constraints b_a_id deferred;**
insert into b (id, a_id) values (1, 1); -- success
insert into a (id) values (1);
commit;

**alter table b alter constraint b_a_id not deferrable;**
```

**How to update primary key in origin table and tables that reference it as foreign keys ?**

```
drop table if exists a cascade;
create table if not exists a (id int primary key);
drop table if exists b;
create table if not exists  b (
  id int primary key, a_id int,
  constraint b_a_id foreign key (a_id) references public.a (id)
);

insert into a (id) values (1);
insert into b (id, a_id) values (1, 1); 

alter table b alter constraint b_a_id deferrable initially immediate;

begin;
set constraints b_a_id deferred;
update a set id=3 where id=1;
update b set a_id=3 where a_id=1;
commit;

select * from a; select * from b;
```