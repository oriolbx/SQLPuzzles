# SQL Puzzles

## 01.Dance partners

You are tasked with providing a list of dance partners from the following table.

```sql
CREATE TABLE dance_partners ( id integer, gender text  );
INSERT INTO dance_partners (id, gender) 
VALUES (1001, 'M'),
(2002, 'M'),
(3003, 'M'),
(4004, 'M'),
(5005, 'M'),
(6006, 'F'),
(7007, 'F'),
(8008, 'F'),
(9009, 'F');
```

Provide an SQL statement that matches each Student ID with an individual of the opposite gender.

Note there is a mismatch in the number of students, as one female student will be left without a dance
partner. Please include this individual in your list as well.

Solution: Use row_number() to set an artificial id to match males and females

```sql
WITH male as (
    SELECT *, row_number() OVER(ORDER BY id) as dance_id FROM dance_partners
    WHERE gender = 'M'
),
female as (
    SELECT *, row_number() OVER(ORDER BY id) as dance_id FROM dance_partners
    WHERE gender = 'F'
)

SELECT a.id as m_id, b.id as f_id
FROM male a
LEFT JOIN female b
ON a.dance_id = b.dance_id;
```
## 02.Managers and Employees


Given the following table, write an SQL statement that determines the level of depth each employee has
from the president.

```sql
CREATE TABLE managers_employees (employee_id integer, manager_id integer, title text, salary text);
INSERT INTO managers_employees (employee_id, manager_id, title, salary)
VALUES (1001, null ,'President' , '$185,000'),
(2002 ,1001, 'Director' , '$120,000'),
(3003 ,1001, 'Office Manager', '$97,000'),
(4004 ,2002, 'Engineer' , '$110,000'),
(5005 ,2002, 'Engineer' , '$142,000'),
(6006 ,2002, 'Engineer' , '$160,000');
```

Solution: Use recursive query to get the relation/depth:

```sql
WITH recursive subordinates as (
    SELECT *, 0 as depth FROM managers_employees
    WHERE manager_id is null
    UNION ALL
    SELECT a.*, b.depth + 1 as depth
    FROM managers_employees a
    INNER JOIN subordinates b
    ON a.manager_id = b.employee_id
)
SELECT * FROM subordinates;
```

## 03.Fiscal year pay rates

For each standard fiscal year, a record exists for each employee that states their current pay rate for the
specified year.

Can you determine all the constraints that can be applied to this table to ensure that it contains only
correct information? 

```sql
CREATE TABLE employee_pay_record
(
EmployeeID INTEGER,
FiscalYear INTEGER,
StartDate DATE,
EndDate DATE,
PayRate float
);
```

Assume that no pay raises are given mid-year. There are quite a few of them, so
think carefully!

```sql
BEGIN;
-- make sure that there are not null values
ALTER TABLE employee_pay_record ALTER COLUMN EmployeeID type INTEGER;
ALTER TABLE employee_pay_record ALTER COLUMN EmployeeID SET NOT NULL;

ALTER TABLE employee_pay_record ALTER COLUMN FiscalYear type INTEGER;
ALTER TABLE employee_pay_record ALTER COLUMN FiscalYear SET NOT NULL;

ALTER TABLE employee_pay_record ALTER COLUMN StartDate type date;
ALTER TABLE employee_pay_record ALTER COLUMN StartDate SET NOT NULL;

ALTER TABLE employee_pay_record ALTER COLUMN EndDate type date;
ALTER TABLE employee_pay_record ALTER COLUMN EndDate SET NOT NULL;

ALTER TABLE employee_pay_record ALTER COLUMN PayRate type float;
ALTER TABLE employee_pay_record ALTER COLUMN PayRate SET NOT NULL;

-- set fiscal year and employee_id as primary keys
ALTER TABLE employee_pay_record ADD PRIMARY KEY (EmployeeID, FiscalYear);

-- fiscal year must be year of startdate
ALTER TABLE employee_pay_record ADD CONSTRAINT Check_Year_StartDate
 CHECK (FiscalYear = EXTRACT('year' from StartDate));

ALTER TABLE employee_pay_record ADD CONSTRAINT Check_Month_StartDate
 CHECK (EXTRACT('month' FROM StartDate) = 1);

ALTER TABLE employee_pay_record ADD CONSTRAINT Check_Day_StartDate
 CHECK (EXTRACT('day' FROM StartDate) = 1);

ALTER TABLE employee_pay_record ADD CONSTRAINT Check_Year_EndDate
 CHECK (FiscalYear = EXTRACT('year' FROM EndDate));

ALTER TABLE employee_pay_record ADD CONSTRAINT Check_Month_EndDate
 CHECK (EXTRACT('month' from EndDate) = 12);

ALTER TABLE employee_pay_record ADD CONSTRAINT Check_Day_EndDate
 CHECK (EXTRACT('day' FROM EndDate) = 31);
 COMMIT;
```

## 04.Two predicates

Write an SQL statement given the following requirements.

For every customer that had a delivery to California, provide a result set of the customer orders that
were delivered to Texas.

Customer ID 1001 would be in the expected output as this customer had deliveries to both California
and Texas. Customer ID 3003 would not show in the result set as they did not have a delivery to Texas,
and Customer ID 4004 would not appear in the result set as they did not have a delivery to California.

```sql
CREATE TABLE deliveries (customer_id integer, order_id text, delivery_state text, amount text);
INSERT INTO deliveries (customer_id, order_id, delivery_state, amount)
VALUES (1001 ,'Ord936254' ,'CA' ,'$340'),
(1001 ,'Ord143876' ,'TX' ,'$950'),
(1001 ,'Ord654876' ,'TX' ,'$670'),
(1001 ,'Ord814356' ,'TX' ,'$860'),
(2002 ,'Ord342176' ,'WA' ,'$320'),
(3003 ,'Ord265789' ,'CA' ,'$650'),
(3003 ,'Ord387654' ,'CA' ,'$830'),
(4004 ,'Ord476126' ,'TX' ,'$120');
```

Solution: Use lateral join to iterate over each customer_id that deliver to CA and TX

```sql
WITH customer_deliver_california as (
    SELECT distinct customer_id FROM deliveries WHERE delivery_state = 'CA'
)
SELECT a.customer_id, b.order_id, b.delivery_state, b.amount
FROM customer_deliver_california a
CROSS JOIN LATERAL (
    SELECT customer_id, order_id, delivery_state, amount
    FROM deliveries
    WHERE a.customer_id = deliveries.customer_id
    AND deliveries.delivery_state = 'TX'
) b;
```

## 05.Phone directory

Your customer phone directory table allows individuals to setup a home, cellular, or a work phone
number.

```sql
CREATE TABLE phone_directory ( customer_id integer, type text, phone_number text);
INSERT INTO phone_directory(customer_id, type, phone_number)
VALUES (1001 ,'Cellular', '555-897-5421'),
(1001 ,'Work', '555-897-6542'),
(1001 ,'Home', '555-698-9874'),
(2002 ,'Cellular', '555-963-6544'),
(2002 ,'Work', '555-812-9856'),
(3003 ,'Cellular', '555-987-6541');
```

Write an SQL statement to transform the following table into the expected output.


| CustomerID | Cellular | Work | Home|
| 1001 | 555-897-5421 | 555-897-6542 | 555-698-9874 |
| 2002 | 555-963-6544 | 555-812-9856 | |
| 3003 | 555-987-6541| | |

Solution: Pivot table 

```sql
WITH cellular as (
    SELECT customer_id, phone_number
    FROM  phone_directory
    WHERE type = 'Cellular'
),
work as (
    SELECT customer_id, phone_number
    FROM  phone_directory
    WHERE type = 'Work'
),
home as (
    SELECT customer_id, phone_number
    FROM  phone_directory
    WHERE type = 'Home'
)
SELECT a.customer_id, a.phone_number as cellular, b.phone_number as work, c.phone_number as home
FROM cellular a
LEFT JOIN work b
ON a.customer_id = b.customer_id
LEFT JOIN home c
ON a.customer_id = c.customer_id;
```


## 06.Workflow Steps

Write an SQL statement that determines all workflows that have started but have not completed.

```sql
CREATE TABLE workflow (workflow text, step_number integer, completion_date date);
INSERT INTO workflow(workflow, step_number, completion_date)
VALUES ('Alpha', 1 ,'7/2/2018'),
('Alpha', 2 ,'7/2/2018'),
('Alpha', 3 ,'7/1/2018'),
('Bravo', 1 ,'6/25/2018'),
('Bravo', 2 ,null),
('Bravo', 3 ,'6/27/2018'),
('Charlie', 1 ,null),
('Charlie', 2 ,'7/1/2018');
```

Solution:

```sql
SELECT workflow FROM (
    SELECT workflow, count(*) FILTER(WHERE completion_date is null) FROM workflow GROUP BY 1
) sub WHERE count > 0;
```


## 07.Misstion to Mars

You are given the following tables that list the requirements for a space mission and a list of potential
candidates.

```sql
CREATE TABLE candidates(id integer, description text);
INSERT INTO candidates(id, description)
VALUES (1001, 'Geologist'),
(1001, 'Astrogator'),
(1001, 'Biochemist'),
(1001, 'Technician'),
(2002, 'Surgeon'),
(2002, 'Machinist'),
(3003, 'Cryologist'),
(4004, 'Selenologist');

-- requirements:
CREATE TABLE requirements(description text);
INSERT INTO requirements(description)
VALUES ('Geologist'), ('Astrogator'), ('Technician');
```

Write an SQL statement to determine which candidates meet the requirements of the mission.

Solution:

```sql
WITH number_requirements as (
    SELECT count(*) FROM requirements
)
SELECT id FROM (
SELECT c.* FROM candidates c 
INNER JOIN requirements r 
ON c.description = r.description) sub
GROUP BY id
HAVING count(*) = (SELECT count FROM number_requirements);
```

## 08. Workflow cases

You have a report of all workflows and their case results. 

A value of 0 signifies the workflow failed, and a value of 1 signifies the workflow passed.

Write an SQL statement that transforms the following table into the expected output.

```sql
CREATE TABLE workflow_cases(workflow text, case1 integer, case2 integer, case3 integer);
INSERT INTO workflow_cases(workflow, case1, case2, case3)
VALUES ('Alpha', 0, 0, 0),
('Bravo', 0, 1, 1),
('Charlie', 1, 0, 0),
('Delta', 0, 0, 0);
```

Solution: 

```sql
SELECT workflow, case1 + case2 + case3 as passed 
FROM workflow_cases;
```

## 09. Matching sets

Write an SQL statement that matches an employee to all other employees that carry the same licenses.

```sql
CREATE TABLE employee_license(id integer, license text);
INSERT INTO employee_license(id, license)
VALUES (1001, 'Class A'),
(1001, 'Class B'),
(1001, 'Class C'),
(2002, 'Class A'),
(2002, 'Class B'),
(2002, 'Class C'),
(3003, 'Class A'),
(3003, 'Class D');
```

Employee ID 1001 and 2002 would be in the expected output as they both carry a Class A, Class B, and a
Class C license.

Solution:

```sql
-- option 1
WITH aggregating as (
    SELECT array_agg(license ORDER BY license ASC) as licenses, id FROM employee_license GROUP BY id
)

SELECT a.id FROM aggregating a INNER JOIN aggregating b ON a.licenses = b.licenses AND a.id <> b.id;

-- option 2
WITH data as (
    SELECT a.id, count(a.id) FROM employee_license a 
    INNER JOIN employee_license b 
    ON a.license = b.license 
    AND a.id <> b.id
    GROUP BY a.id
)
SELECT distinct d.id
FROM data d
INNER JOIN data d2
ON d.count = d2.count
AND d.id <> d2.id;

-- option3
WITH employee_count as (
    SELECT id, count(*) as license_count
    FROM employee_license
    GROUP BY id
),
employee_count_combined as (
    SELECT
    a.id as id,
    b.id as id2,
    count(*)  as license_count_combo
    FROM employee_license a
    INNER JOIN employee_license b
    ON a.license = b.license
    WHERE a.id <> b.id
    GROUP BY a.id, b.id
)
SELECT a.id, a.id2, a.license_count_combo
FROM employee_count_combined a 
INNER JOIN employee_count b 
ON a.license_count_combo = b.license_count AND
 a.id <> b.id;
```


## 10.Mean, Median, Mode, and Range

The mean is the average of all numbers.
The median is the middle number in a sequence of numbers.
The mode is the number that occurs most often within a set of numbers.
The range is the difference between the highest and lowest values in a set of numbers.

Write an SQL statement to determine the mean, median, mode and range of the following set of
integers.

```sql
CREATE TABLE sample_data (integer_value integer);
INSERT INTO sample_data VALUES(5),(6),(10),(10),(13),(14),(17),(20),(81),(90),(76);
```

Solution:

```sql
WITH number_set as (
    SELECT 
        integer_value,
        row_number() OVER(ORDER BY integer_value) as row_id,
        (SELECT count(*) FROM sample_data) as ct
    FROM sample_data
), median as (
    select avg(integer_value) as median
    from number_set
    where row_id between ct::numeric/2.0 and ct::numeric/2.0 + 1
), mode as (
    select integer_value as mode
    from sample_data
    group by 1
    order by count(1) desc
    limit 1
), other as (
    SELECT 
    AVG(integer_value) as mean,
    MAX(integer_value) - MIN(integer_value) as range
    FROM sample_data
)

SELECT mean, median, mode, range FROM
median, mode, other;
```

## 11.Permutations

You are given the following list of test cases and must determine all possible permutations.
Write an SQL statement that produces the expected output.

```sql
CREATE TABLE permutations (test_case varchar(1));
INSERT INTO permutations VALUES('A'), ('B'), ('C');
```

Solution: cross joins

```sql
SELECT row_number() OVER(), concat_ws(',', a.test_case, b.test_case, c.test_case) as output
FROM permutations a, permutations b, permutations c
WHERE a.test_case <> b.test_case
AND a.test_case <> c.test_case
AND b.test_case <> c.test_case;
```

## 12. Average days

Write an SQL statement to determine the average number of days between executions for each
workflow.

```sql
CREATE TABLE workflow_days (workflow text, exectuion_date date);
INSERT INTO workflow_days (workflow, exectuion_date)
VALUES ('Alpha' ,'6/1/2018'),
('Alpha' ,'6/14/2018'),
('Alpha' ,'6/15/2018'),
('Bravo' ,'6/1/2018'),
('Bravo' ,'6/2/2018'),
('Bravo' ,'6/19/2018'),
('Charlie' ,'6/1/2018'),
('Charlie' ,'6/15/2018'),
('Charlie' ,'6/30/2018');
```

Solution:

```sql
WITH ordered_set as (
    SELECT workflow, exectuion_date
    FROM workflow_days
    ORDER BY workflow, exectuion_date
),
get_previous_date as (
    SELECT *, LAG(exectuion_date) OVER(PARTITION BY workflow ORDER BY exectuion_date) as previous_date
    FROM ordered_set
)
SELECT workflow, TRUNC(AVG(DATE_PART('day', AGE(exectuion_date,previous_date)))) as average_days
FROM get_previous_date
GROUP BY workflow;
```


## 13. Inventory tracking

You work for a manufacturing company and need to track inventory adjustments from the warehouse.
Some days the inventory increases, on other days the inventory decreases.
Write an SQL statement that will provide a running balance of the inventory.

```sql
CREATE TABLE inventory_tracking (date date, quantity numeric);
INSERT INTO inventory_tracking (date, quantity)
VALUES ('7/1/2018', 100),
('7/2/2018', 75),
('7/3/2018', -150),
('7/4/2018', 50),
('7/5/2018', -100);
```


Solution:

```sql
SELECT *, SUM(quantity) OVER(ORDER BY date)
FROM inventory_tracking ORDER BY date ASC;
```


## 14. Indeterminate process log

Your process log has several workflows broken down by step numbers with the possible status values of
Complete, Running, or Error.

Your task is to write an SQL statement that creates an overall status based upon the following
requirements.

* If all the workflow steps have a status of complete, set the overall status to complete. (ex.
Bravo).
* If all the workflow steps have a status of error, set the overall status to error (ex. Foxtrot).
* If the workflow steps have the combination of error and complete, or error and running, the
overall status should be indeterminate (ex. Alpha, Charlie, Echo).
* If the workflow steps have the combination of complete and running, the overall status should
be running (ex. Delta).