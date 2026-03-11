Can replace:

- elasticsearch
- redis
- mongodb
- rabbitMQ
- kafka
- REST API

# Install

```
sudo apt install -y postgresql
```

Check if it runs

```
sudo service postgresql status
```

Terminal syntax highlighting. Replace `psql` with `pgcli` in commands.

```
sudo apt install -y pgcli
```

# Login

Anyone with `sudo` access can enter PostgreSQL as the `postgres` superuser without a password.

```
sudo -u postgres psql

sudo -u postgres psql -d database_name
```

With user

```
psql -U username -h localhost -d database_name
```

# Commands

```bash
# Show databases
\l

# Use (connect) database
\c database_name

# Show tables
\dt

# Describe table
\d table_name

# List schemas
\dn

# Schema owner
\dn+

# List roles and database-level privileg
\du

# Current database
SELECT current_database();

# Quit
\q
```

# User

```sql
CREATE USER username WITH PASSWORD 'password';

CREATE DATABASE database_name;

GRANT ALL PRIVILEGES ON DATABASE database_name TO username;
GRANT ALL PRIVILEGES ON SCHEMA database_name TO username;
```

Full control

```sql
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA schema_name TO username;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA schema_name TO username;
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA schema_name TO username;
```

Create a superuser will all access automatically

```sql
CREATE ROLE username WITH LOGIN SUPERUSER PASSWORD 'password';
```

# Schema

A schema in PostgreSQL is a namespace inside a database that organizes database objects—tables, types, functions, sequences, etc.

Every PostgreSQL database includes a `public` schema by default.

All users can usually access it, but only the schema owner or users with CREATE privilege can add objects.

You can create additional schemas in the same database to organize objects or isolate access.

Schemas prevent name collisions: two tables in different schemas can have the same name.

Database privileges (GRANT ALL PRIVILEGES ON DATABASE testdb TO testuser;) allow connecting and creating temporary objects.

To create permanent objects (tables, types) you need CREATE privilege on a schema.

Each schema has exactly one owner.

Database owner vs. schema owner are separate.

CREATE DATABASE sets the database owner, but the public schema owner is often pg_database_owner for new databases.

when you create a database as superuser (postgres) without specifying an owner, the database is automatically assigned an internal role called pg_database_owner as the schema owner.

postgres is still a superuser and can do anything, but the public schema inside the database is owned by pg_database_owner, not postgres.

This prevents superusers from unintentionally owning every schema in every database they create.

# Composite types

```sql
-- Composite type
create type address as (street text, city text);

-- Table
create table customers (name text, ship_to address);

-- Row
insert into customers (name, ship_to)
values ('Homer', row('123 Maple str', 'Springfield'));

-- Shorter way
insert into customers (name, ship_to)
values ('Marge', ('123 Maple str', 'Springfield'));
```

Selecting

```sql
select * from customers;

/*
 name  |            ship_to
-------+-------------------------------
 Homer | ("123 Maple str",Springfield)
 Marge | ("123 Maple str",Springfield)
*/

select name, (ship_to).city from customers;

/*
 name  |    city
-------+-------------
 Homer | Springfield
 Marge | Springfield
*/
```

# JSONB (Binary)

https://neon.com/guides/document-store

Allows JSON parsing for querying unstructured nested fields.

The JSON data is stored in a decomposed binary format, slightly slower to input due to conversion, but makes it queryable and indexable.

```sql
create table users (
    id serial primary key,
    name text not null,
    profile jsonb
);

insert into users (name, profile)
values
    ('Homer Simpson', ('{"city": "Springfield", "age": 39, "occupation": "Safety Inspector"}')),
    ('Bart Simpson', ('{"city": "Springfield", "age": 13}')),
    ('Marge Simpson', ('{"city": "Springfield", "age": 36, "interests": ["Reading", "Cooking"]}')),
    ('Jane Doe', ('{"city": "Shelbyville", "age": 32, "occupation": "Developer"}'))
;

select * from users;

/*
 id |     name      |                                profile
----+---------------+-----------------------------------------------------------------------
  1 | Homer Simpson | {"age": 39, "city": "Springfield", "occupation": "Safety Inspector"}
  2 | Bart Simpson  | {"age": 13, "city": "Springfield"}
  3 | Marge Simpson | {"age": 36, "city": "Springfield", "interests": ["Reading", "Cooking"]}
  4 | Jane Doe      | {"age": 32, "city": "Shelbyville", "occupation": "Developer"}
*/
```

Return rows that have a specific JSON property value

```sql
select * from users where profile ->> 'city' = 'Springfield';

/*
 name  |            ship_to
-------+-------------------------------
 Homer | ("123 Maple str",Springfield)
 Marge | ("123 Maple str",Springfield)
*/
```

Return rows that have a specific JSON key value pair

```sql
select * from users where profile @> '{"age": 13}';

/*
 id |     name     |              profile
----+--------------+------------------------------------
  2 | Bart Simpson | {"age": 13, "city": "Springfield"}
*/
```

Return rows that have a specific JSON property. Extremely useful for filtering unstructured data.

```sql
select * from users where profile ? 'interests';

/*
id |     name      |                                profile
----+---------------+-----------------------------------------------------------------------
  3 | Marge Simpson | {"age": 36, "city": "Springfield", "interests": ["Reading", "Cooking"]}
*/
```

Updating

```sql
update users
set profile = jsonb_set(profile, '{age}', '20'::jsonb)
where id = 1;

/*
 id |     name      |                                profile
----+---------------+-----------------------------------------------------------------------
  2 | Bart Simpson  | {"age": 13, "city": "Springfield"}
  3 | Marge Simpson | {"age": 36, "city": "Springfield", "hobbies": ["Reading", "Cooking"]}
  4 | Jane Doe      | {"age": 32, "city": "Shelbyville", "occupation": "Developer"}
  1 | Homer Simpson | {"age": 20, "city": "Springfield", "occupation": "Safety Inspector"}
*/
```

Selecting nested data

```sql
CREATE TABLE documents (
  id SERIAL PRIMARY KEY,
  data JSONB
);

INSERT INTO documents (data)
VALUES (
  '{
    "title": "Neon and JSONB",
    "body": "Using JSONB to store flexible data structures in Postgres.",
    "tags": ["Postgres", "Neon", "JSONB"],
    "steps": ["Set up a table with a JSONB column", "Insert and retrieve JSONB data"]
  }'
),
(
  '{
    "title": "Scaling Neon with Postgres",
    "body": "Learn how to scale your Neon instances with PostgreSQL features.",
    "tags": ["Neon", "Postgres", "scaling"],
    "author": { "name": "John Smith", "age": 30 }
  }'
);

SELECT *
FROM documents
WHERE data->'author' ->> 'name' = 'John Smith';

-- INTs must be cast
SELECT *
FROM documents
WHERE (data -> 'author' ->> 'age')::int > 29;
```

# JSON_TABLE

https://www.youtube.com/watch?v=es4IGr701RA

Used for expanding JSON arrays.

> **WORKS ONLY WITH VERSION 17+**

Data:

```sql
-- First, let's create a table to store our JSON data
--drop table dealership_inventory
CREATE TABLE dealership_inventory (
    id SERIAL PRIMARY KEY,
    inventory_data JSONB
);

-- Insert a sample JSON object into the table
INSERT INTO dealership_inventory (inventory_data) VALUES (
    '{
        "dealership": "UsedCars",
        "cars": [
            {
                "make": "Toyota",
                "model": "Camry",
                "year": 2022,
                "price": 25000,
                "features": ["Bluetooth", "Backup Camera"]
            },
            {
                "make": "Honda",
                "model": "Civic",
                "year": 2023,
                "price": 22000,
                "features": ["Apple CarPlay", "Lane Assist"]
            },
            {
                "make": "Ford",
                "model": "F-150",
                "year": 2021,
                "price": 35000,
                "features": ["Tow Package", "4WD"]
            }
        ]
    }'
);


--step 3:  Select all the data
select * from dealership_inventory
```

Select:

```sql
--step 4: using the json_table function
SELECT
    jt.*
FROM
    dealership_inventory,
    json_table(
        inventory_data->'cars',
        '$[*]' COLUMNS(
            make TEXT PATH '$.make',
            model TEXTPATH '$.model',
            make_year INT PATH '$.year',
            price DECIMAL PATH '$.price'
        )
    ) AS jt
ORDER BY
    jt.price DESC;

--step 5: selecting specific columns
SELECT
    cars.make,
    cars.model,
    cars.make_year,
    cars.price
FROM
    dealership_inventory,
    json_table(
        inventory_data->'cars',
        '$[*]' COLUMNS (
            make TEXT PATH '$.make',
            model TEXT PATH '$.model',
            make_year INT PATH '$.year',
            price DECIMAL PATH '$.price'
        )
    ) AS cars
ORDER BY
    cars.price DESC;

-- Now, let's use json_table to query this data
SELECT
    jt.*,
    f.feature
FROM
    dealership_inventory,
    json_table(
        inventory_data->'cars',
        '$[*]' COLUMNS (
            make TEXT PATH '$.make',
            model TEXT PATH '$.model',
            year INT PATH '$.year',
            price DECIMAL PATH '$.price',
            features JSON PATH '$.features'
        )
    ) AS jt
CROSS JOIN LATERAL json_array_elements_text(jt.features) AS f(feature)
WHERE
    jt.price > 1000 AND f.feature LIKE '%Play';
```

Here are the **NON** json_table equivalents

```sql
-- Option 1
SELECT
    jt.*
FROM
    dealership_inventory,
    jsonb_array_elements(inventory_data->'cars') AS car,
    jsonb_to_record(car) AS jt(
        make TEXT,
        model TEXT,
        make_year INT,
        price DECIMAL
    )
ORDER BY
    jt.price DESC;

-- Option 2 - More direct
SELECT
    (car->>'make')::TEXT AS make,
    (car->>'model')::TEXT AS model,
    (car->>'year')::INT AS make_year,
    (car->>'price')::DECIMAL AS price
FROM
    dealership_inventory
    CROSS JOIN LATERAL jsonb_array_elements(inventory_data->'cars') AS car
ORDER BY
    price DESC;
```

# Multi version concurrency control (MVCC)

Every transaction gets a snapshot of the data, allowing extremely efficient concurrency while preventing deadlocks.

# Extensibility

https://postgresforeverything.com
