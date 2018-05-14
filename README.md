# SQL Conventions

## Data layer

* For SQL use [PostgreSQL](https://www.postgresql.org), it's the [most loved relational database (StackOverflow survey 2018)](https://insights.stackoverflow.com/survey/2018/#technology-most-loved-dreaded-and-wanted-databases) and it's a multi-model database (K/V store, FDW and much more). Any questions?

## Application layer

* If your API is only doing mainly data persistence use [Postgrest](https://postgrest.com) is the way to go and only implement the missing part in another process. You can then compose both API with the reverse-proxy.
* Otherwise, use a data-mapping library (e.g. [doobie](https://github.com/tpolecat/doobie)) not an ORM.

## Tables/Views

### Name

* singular ([why](https://launchbylunch.com/posts/2014/Feb/16/sql-naming-conventions/#singular-relations)), e.g. `team` not `teams`
* camelCase, even for `n-n` tables, never snake_case, it's reserved for PK and FK.

### Columns

* Column names in **camelCase**, e.g. `createdAt`
* Only use underscore for PK and FK columns e.g. (PK) `user_id`, (FK) `organization_id`
* NOT NULL by default, NULL is the exception (think of it as the [maybe Monad](https://github.com/chrissrogers/maybe#why))
* pas d'abbréviations des mots sauf pour des expressions bien connues et longue (e.g. "i18n")
(* pas de mots-clés réservé (par exemple `user` sur PGSQL).)
* utiliser des UUID en type de  PK & FK ([why](https://www.clever-cloud.com/blog/engineering/2015/05/20/why-auto-increment-is-a-terrible-idea/)).
* `createdAt TIMESTAMPTZ DEFAULT now()`
* `updatedAt TIMESTAMPTZ DEFAULT now()` unless you plan to leverage event-sourcing
* `deletedAt TIMESTAMPTZ DEFAULT NULL`:
  * unless you plan to leverage event-sourcing
  * don't forget to [`deletedAt`](http://stackoverflow.com/questions/8289100/create-unique-constraint-with-null-columns/8289253#8289253)
* Comment each column, explain your rational, explain your decisions, should be in plain english for internal use only
* Boolean columns must start with either `is` or `has`.

## Functions

### Name

They are 3 types of functions, `notiy` functions and `private` functions and `public` functions
- **notify**, format: notify[*SchemaName*][*TableName*][*Event*] (e.g. `notifyAuthenticationUserCreated(user_id)`): should only format the notification message underneath and use pg_notify. Beware of the [8000 characters limit](http://stackoverflow.com/a/41059797/745121), only send metadata (ids), data should be asked by workers through the API. If you really wish to send data then [pg_kafka](https://github.com/xstevens/pg_kafka) might be a better alternative.
- **private**, format: _[*functionName*] (e.g. `_resetFailedLogin`): must never be exposed through the public schema. Used mainly for consistency and business-rules
- **public**, format [*functionName*] (e.g. `logIn(email, password)`): must be exposed through the public schema.


## Types

Enum types should be in *singular*, in *camelCase**.

## Triggers

### Name

(translation in progress)

### Columns

* utiliser BNCF (au dessus de la 3NF) (cf normal form)

* leverage `using`, so instead of:
```
select <fields> from
  table_1
  inner join table_2
    using (table_1_id)
```

use:

```
select <fields> from
  table_1
  inner join table_2
    on table_1.table_1_id =
       table_2.table_1_id
```

* utiliser les enum PG qui sont des types
* use the right PostgreSQL types:

```
inet (IP address)
timestamp with time zone
point (2D point)
tstzrange (time range)
interval (duration)
```

* utiliser les tableaux si besoin (permet de gérer la notion "d'ordre" facilement)
* constraint should be inside your database as much as possible:

```sql
create table reservation(
    reservation_id uuid primary key,
    dates tstzrange not null,
    exclude using gist (dates with &&)
);
```

* use row-level-security to ensure R/U/D access on each table rows

* standard names for indexes in PostgreSQL are: `{tablename}_{columnname(s)}_{suffix}` (e.g. `item_a_b_pkey`) where the suffix is one of the following:
  * Primary Key constraint: `pk`
  * Foreign key: `fk`
  * Unique constraint: `key`
  * Check constraint: `chk`
  * Exclusion constraint: `exl`
  * Any other kind of index: `idx`

([source](http://stackoverflow.com/questions/4107915/postgresql-default-constraint-names/4108266#4108266))

## Policies

### Name


