# postgres-sharding
postgres Foreign Data Wrapper


docker run --name main -p 5432:5432 -e POSTGRES_PASSWORD=passMain -d postgres

docker run --name main2022 -p 5433:5432 -e POSTGRES_PASSWORD=pass2022 -d postgres

docker run --name main2021 -p 5434:5432 -e POSTGRES_PASSWORD=pass2021 -d postgres

```sql

CREATE DATABASE mydb;


CREATE TABLE forex
(
    id                BIGINT,
    account           VARCHAR(24)        NOT NULL,
    currency          CHAR(3)            NOT NULL, 
    amount            NUMERIC(20,2)      NOT NULL,
    cats              VARCHAR(30)        NOT NULL, 
    regdate            DATE -- 2023-01-02
) PARTITION BY RANGE (regdate);

CREATE INDEX forex_acc_idx ON forex (account);

CREATE TABLE forex_y2023 PARTITION OF forex
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

ALTER TABLE forex_y2023 ADD CONSTRAINT forex_y2023_pk_id PRIMARY KEY (id);

-- add in main2022 server
CREATE TABLE forex_y2022 PARTITION OF forex
    FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');

ALTER TABLE forex_y2022 ADD CONSTRAINT forex_y2022_pk_id PRIMARY KEY (id);

-- add in main2021 server
CREATE TABLE forex_y2021 PARTITION OF forex
    FOR VALUES FROM ('2021-01-01') TO ('2022-01-01');

ALTER TABLE forex_y2021 ADD CONSTRAINT forex_y2021_pk_id PRIMARY KEY (id);


CREATE TABLE accounts
(
    account          VARCHAR(24)        NOT NULL,
    name             VARCHAR(255)       NOT NULL,
    PRIMARY KEY (account)
);

insert into accounts VALUES('123', 'super account');
insert into accounts VALUES('124', 'normal account');
insert into accounts VALUES('125', 'low account');
insert into accounts VALUES('120', 'prostoy account');
insert into accounts VALUES('121', 'special account');
insert into accounts VALUES('122', 'usd account');
insert into accounts VALUES('126', 'eur account');
insert into accounts VALUES('127', 'tjs account');


insert into forex VALUES(1, '123', 'TJS', 100.30, 'buy',  '2023-03-18');
insert into forex VALUES(2, '122', 'USD', 130.50, 'sell',  '2023-03-17');
insert into forex VALUES(3, '124', 'TJS', 200.00, 'buy',  '2023-03-16');

insert into forex VALUES(4, '120', 'TJS', 300.30, 'buy',  '2022-02-11');
insert into forex VALUES(5, '121', 'TJS', 400.40, 'sell',  '2022-04-12');
insert into forex VALUES(6, '126', 'USD', 930.50, 'sell',  '2022-03-17');

insert into forex VALUES(7, '125', 'TJS', 500.00, 'buy',  '2021-05-26');
insert into forex VALUES(8, '127', 'TJS', 600.30, 'buy',  '2021-06-21');
insert into forex VALUES(9, '121', 'TJS', 700.40, 'sell',  '2021-07-10');

-- to all in main server

CREATE EXTENSION postgres_fdw;
-- Create foreign server 1.
--host ip or hostname (main2022)
CREATE SERVER main2022_server
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host '172.17.0.3', port '5432', async_capable 'true', dbname 'mydb');

--  map user
CREATE USER MAPPING FOR postgres
SERVER main2022_server
OPTIONS (user 'postgres', password 'pass2022');

-- create table
CREATE FOREIGN TABLE forex_y2022 PARTITION OF forex
FOR VALUES FROM ('2022-01-01') TO ('2023-01-01')
SERVER main2022_server OPTIONS (table_name 'forex');
