# SQL Conventions

## Data layer

* use PostgreSQL (fallback on SQLite)

## Application layer

* use PostgREST
* utiliser une librairie de data-mapping (e.g. [doobie](https://github.com/tpolecat/doobie)) et non un ORM.

## Tables

### Name

* pas d'abbréviations des mots sauf pour des expressions bien connues et longue (e.g. "i18n")
* pas de mots-clés réservé (par exemple `user` sur PGSQL)
* table and view names should be singular and in camelCase, e.g. `team` not `teams` ([why](https://launchbylunch.com/posts/2014/Feb/16/sql-naming-conventions/#singular-relations))
* nom des champs/tables en **camelCase**, e.g. `createdAt`
* utiliser uniquement underscore pour les foreign-keys des tables, e.g. `user_id`
* utiliser des UUID en type de  PK & FK ([why](https://www.clever-cloud.com/blog/engineering/2015/05/20/why-auto-increment-is-a-terrible-idea/))
* chaque table doit avoir les champs `createdAt`, [`deletedAt`](http://stackoverflow.com/questions/8289100/create-unique-constraint-with-null-columns/8289253#8289253) et `updatedAt` si la table contient des données. 

## Functions

### Name

- notify[SchemaName][TableName][Event] e.g. notifyAuthenticationUserCreat**ed**(*user_id*): should only format the notification message underneath and use pg_notify. Beware of the [8000 characters limit](http://stackoverflow.com/a/41059797/745121), only send metadata (ids), data should be asked by workers through the API. If you really wish to send data then [pg_kafka](https://github.com/xstevens/pg_kafka) might be a better alternative.

## Triggers

### Name

(translation in progress)

## 

* utiliser BNCF (au dessus de la 3NF) (cf normal form)
* always set column to NOT NULL by default, use NULL only when necessary
* never use ON DELETE CASCADE, set `deletedAt` to `NOW()`
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
