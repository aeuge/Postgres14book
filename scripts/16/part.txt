# partitioning
CREATE DATABASE part;
\c part

CREATE TABLE logs (
    id              serial,
    logdate         date not null,
    message         text,
    user_id         int
) PARTITION BY RANGE (logdate);

-- create sections
CREATE TABLE logs_y2021m01 PARTITION OF logs FOR VALUES FROM ('2021-01-01') TO ('2021-02-01');
CREATE TABLE logs_y2021m02 PARTITION OF logs FOR VALUES FROM ('2021-02-01') TO ('2021-03-01');
CREATE TABLE logs_y2021m03 PARTITION OF logs FOR VALUES FROM ('2021-03-01') TO ('2021-04-01');
CREATE TABLE logs_y2021m04 PARTITION OF logs FOR VALUES FROM ('2021-04-01') TO ('2021-05-01');
CREATE TABLE logs_y2021m05 PARTITION OF logs FOR VALUES FROM ('2021-05-01') TO ('2021-06-01');
CREATE TABLE logs_y2021m06 PARTITION OF logs FOR VALUES FROM ('2021-06-01') TO ('2021-07-01');
CREATE TABLE logs_y2021m07 PARTITION OF logs FOR VALUES FROM ('2021-07-01') TO ('2021-08-01');
CREATE TABLE logs_y2021m08 PARTITION OF logs FOR VALUES FROM ('2021-08-01') TO ('2021-09-01');
CREATE TABLE logs_y2021m09 PARTITION OF logs FOR VALUES FROM ('2021-09-01') TO ('2021-10-01');
CREATE TABLE logs_y2021m10 PARTITION OF logs FOR VALUES FROM ('2021-10-01') TO ('2021-11-01');
CREATE TABLE logs_y2021m11 PARTITION OF logs FOR VALUES FROM ('2021-11-01') TO ('2021-12-01');
CREATE TABLE logs_y2021m12 PARTITION OF logs FOR VALUES FROM ('2021-12-01') TO ('2022-01-01');
CREATE TABLE logs_y2022m01 PARTITION OF logs FOR VALUES FROM ('2022-01-01') TO ('2022-02-01');

-- create index
CREATE INDEX ON logs (logdate);

\d+ logs

INSERT INTO logs(logdate, message, user_id) VALUES (now(), 'DELETE RECORD 111 ON TABLE accounts', 33);

SELECT COUNT(*) FROM logs;
SELECT COUNT(*) FROM logs_y2021m05;
SELECT COUNT(*) FROM logs_y2021m06;

EXPLAIN SELECT * FROM logs WHERE logdate = now()::date;
EXPLAIN SELECT * FROM logs WHERE user_id = 1;

-- partitioning large table 
-- 1 option
\c demo
\dt bookings.*
\d+ bookings.bookings

CREATE TABLE bookings.bookings2 (
	book_ref bpchar(6) NOT NULL,
	book_date timestamptz NOT NULL,
	total_amount numeric(10,2) NOT NULL
);
INSERT INTO bookings.bookings2 SELECT * FROM bookings.bookings;

BEGIN;
SET LOCAL statement_timeout to '1s';
ALTER TABLE bookings.bookings ADD CONSTRAINT bookings_book_date_check CHECK (book_date < '2021-06-01' and book_date is not null) not valid;
COMMIT;

ALTER TABLE bookings.bookings VALIDATE CONSTRAINT bookings_book_date_check;

ALTER TABLE bookings.bookings DROP CONSTRAINT bookings_book_date_check;

CREATE TABLE bookings.bookings_part (
	book_ref bpchar(6) NOT NULL,
	book_date timestamptz NOT NULL,
	total_amount numeric(10,2) NOT NULL
) PARTITION BY RANGE (book_date);

CREATE TABLE bookings.bookings_part_y2021m6avobe PARTITION OF bookings.bookings_part FOR VALUES FROM ('2021-06-01') TO (MAXVALUE);


BEGIN;
SET statement_timeout TO '1s';
ALTER TABLE bookings.bookings RENAME TO bookings_archive;
ALTER TABLE bookings.bookings_part RENAME TO bookings;
ALTER TABLE bookings.bookings ATTACH PARTITION bookings.bookings_archive FOR VALUES FROM (MINVALUE) to ('2021-06-01');
COMMIT;

EXPLAIN SELECT * FROM bookings.bookings WHERE book_date = now()::date;

-- 2 option
DROP TABLE bookings.bookings;
ALTER TABLE bookings.bookings2 RENAME TO bookings;

CREATE TABLE bookings.bookings_part (
	book_ref bpchar(6) NOT NULL,
	book_date timestamptz NOT NULL,
	total_amount numeric(10,2) NOT NULL
) PARTITION BY RANGE (book_date);


SELECT min(book_date), max(book_date) FROM bookings.bookings;


CREATE TABLE bookings.bookings_part_y2016m08 PARTITION OF bookings.bookings_part FOR VALUES FROM ('2016-08-01') TO ('2016-09-01');
CREATE TABLE bookings.bookings_part_y2016m09 PARTITION OF bookings.bookings_part FOR VALUES FROM ('2016-09-01') TO ('2016-10-01');
CREATE TABLE bookings.bookings_part_y2016m10 PARTITION OF bookings.bookings_part FOR VALUES FROM ('2016-10-01') TO ('2016-11-01');
CREATE TABLE bookings.bookings_part_y2016m11avobe PARTITION OF bookings.bookings_part FOR VALUES FROM ('2016-11-01') TO (MAXVALUE);

\d+ bookings.bookings_part

INSERT INTO bookings.bookings_part SELECT * FROM bookings.bookings;

-- drop old & rename new
ALTER TABLE bookings.bookings RENAME TO bookings2;

DROP TABLE bookings.bookings;
ALTER TABLE bookings.bookings_part RENAME TO bookings;

-- after sections + data load
CREATE INDEX ON bookings.bookings(book_date);

-- error
CREATE UNIQUE INDEX bookings_pkey2 ON bookings.bookings(book_ref);
ALTER TABLE bookings.bookings ADD CONSTRAINT bookings_pkey_uniq UNIQUE (book_ref);

CREATE UNIQUE INDEX bookings_pkey2 ON bookings.bookings(book_ref, book_date);

EXPLAIN SELECT * FROM bookings.bookings WHERE book_date = now()::date;

-- sliding window
\c part
\d+ logs

INSERT INTO logs(logdate, message, user_id) VALUES ('2021-01-01', 'DELETE RECORD 111 ON TABLE accounts', 33);

ALTER TABLE logs DETACH PARTITION logs_y2021m01;
\d+ logs

SELECT * FROM logs_y2021m01;

CREATE TABLE logs_archive (
    id              serial,
    logdate         date not null,
    message         text,
    user_id         int
) PARTITION BY RANGE (logdate);
ALTER TABLE logs_archive ATTACH PARTITION logs_y2021m01 FOR VALUES FROM ('2021-01-01') TO ('2021-02-01');

SELECT * FROM logs_archive;