# 8 Week SQL Challenges by Danny | Case Study #3 - Foodie-Fi
### ERD for Database
![image](https://github.com/user-attachments/assets/9516db79-b7ef-4ef1-abc8-21ba4bc71242)

## Result of Questions
### Data Preparation
```sql
-- Data Preparation for "plans" table
UPDATE foodie_fi.plans
SET price = case when price = 'null' then null
				 else price
				 end;

ALTER TABLE foodie_fi.plans
ALTER COLUMN price type numeric
USING price::numeric;
```
### Understanding Customer Journey
```sql
WITH rank_subscriptions as
(
	SELECT customer_id
	,plan_id
	,start_date
	,row_number() over (partition by customer_id order by  start_date) as ranking
	FROM foodie_fi.subscriptions
	GROUP BY 1,2,3
)
SELECT rs1.customer_id
,rs1.plan_id as current_subscription
,rs2.plan_id as next_subscription
,age(rs2.start_date, rs1.start_date) as interval_days
FROM rank_subscriptions rs1
JOIN rank_subscriptions rs2 on rs1.customer_id = rs2.customer_id and rs1.ranking + 1 = rs2.ranking
ORDER BY 1,2,3
;
```

_Result:_

![image](https://github.com/user-attachments/assets/e2c07352-6889-4bc2-b7c8-8878ee868db2)

### Q1. How many customers has Foodie-Fi ever had?
```sql
SELECT count( distinct customer_id) as total_customers
FROM foodie_fi.subscriptions;
```

_Result:_

![image](https://github.com/user-attachments/assets/345bfd7e-586f-4f69-a2d7-087a7c8bd48c)

### Q2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value
```sql
SELECT to_char(start_date,'MM-YYYY') as month
,count(distinct customer_id) as total_count
FROM foodie_fi.subscriptions
WHERE plan_id ='0'
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/c1ed9ae3-4dee-41e9-991e-0e9a3cb3c4c2)

### Q3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name
```sql
SELECT p.plan_name
,count(distinct s.customer_id) as customer_count
FROM foodie_fi.subscriptions s
JOIN foodie_fi.plans p on p.plan_id = s.plan_id
WHERE extract(year from start_date) > 2020
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/ae0d1ab2-44ea-4902-b84e-45099ff73316)

### Q4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?
```sql
WITH caculation as
(
	SELECT count(distinct case when plan_id = 4 then customer_id end) as total_churned_customers
	,count(distinct customer_id) as total_customers
	FROM foodie_fi.subscriptions
)
SELECT total_churned_customers
,total_customers
,round((total_churned_customers *100 / total_customers)::int,1) || '%' as percentage
FROM caculation
;
```

_Result:_

![image](https://github.com/user-attachments/assets/3acb862e-c59c-49f3-acec-7352402956ac)

### Q5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?
```sql
WITH ranking as
(
	SELECT customer_id
	,plan_id
	,row_number() over (partition by customer_id order by start_date) as ranking
	FROM foodie_fi.subscriptions
)
, caculation as
(
	SELECT count(distinct case when plan_id = 4 and ranking = 2 then customer_id end) as churned_customer
	,count(distinct customer_id) as total_customers
	FROM ranking
)
SELECT churned_customer
,total_customers
,round((churned_customer*100/total_customers)::int,1) || '%' as percentage
FROM caculation;
```

_Result:_

![image](https://github.com/user-attachments/assets/97acebf2-76dd-424d-a857-aac1ff103bd3)


### Q6. What is the number and percentage of customer plans after their initial free trial?
```sql
WITH ranking as
(
	SELECT s.customer_id
	,s.plan_id
	,p.plan_name
	,row_number() over (partition by customer_id order by start_date) as ranking
	FROM foodie_fi.subscriptions s
	JOIN foodie_fi.plans p on p.plan_id = s.plan_id	
)
SELECT r.plan_id
,r.plan_name
,count(distinct case when r.ranking = 2 then r.customer_id end) as total_customers_after_initial_free_trial
,round((count(distinct case when r.ranking = 2 then r.customer_id end)*100 / count(distinct r.customer_id)) ::int,1) || '%' as percentage
FROM ranking r
JOIN foodie_fi.subscriptions s on s.plan_id = r.plan_id
WHERE r.plan_id != 0
GROUP BY 1,2
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/c04f62e4-afe2-477e-8241-585d397339ed)


### Q7. What is the customer count breakdown of all 5 plan_name values at 2020-12-31?
```sql
SELECT p.plan_name
,count(distinct s.customer_id) as total_customers
FROM foodie_fi.subscriptions s
JOIN foodie_fi.plans p on p.plan_id = s.plan_id
WHERE s.start_date = '2020-12-31'
GROUP BY 1
ORDER BY 1;
```

_Result: no value_

![image](https://github.com/user-attachments/assets/2741469d-1e0c-48d6-b8d7-5d2911516f39)


### Q8. How many customers have upgraded to an annual plan in 2020?
```sql
SELECT count(distinct customer_id) as total_customer_upgraded_annual_plan
FROM foodie_fi.subscriptions
WHERE plan_id = '3' and extract(year from start_date) = 2020
;
```

_Result:_

![image](https://github.com/user-attachments/assets/f4c66af8-1ae5-4616-a958-ba92817a03cf)

### Q9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?
```sql
SELECT round(avg(aud.start_date - td.start_date),2) as avg_days_to_upgrade_annual_plan
FROM foodie_fi.subscriptions aud
JOIN foodie_fi.subscriptions td on aud.customer_id = td.customer_id
WHERE aud.plan_id = '3' and td.plan_id = '0'
;
```

_Result:_

![image](https://github.com/user-attachments/assets/dcb9e105-56be-42a8-a730-ad9dc16e95d0)

### Q10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)
```sql
SELECT case when (uad.start_date - td.start_date) between 0 and 30 then '0-30 days'
			when (uad.start_date - td.start_date) between 31 and 60 then '31-60 days'
			else '60+ days'
			end as day_range
,count(distinct uad.customer_id) as customer_count
FROM foodie_fi.subscriptions uad -- uad: upgraded_annual_date
JOIN foodie_fi.subscriptions td on uad.customer_id = td.customer_id -- td: trial_date
WHERE uad.plan_id = '3' and td.plan_id = '0'
GROUP BY 1
ORDER BY 1
;
```

_Result:_

![image](https://github.com/user-attachments/assets/42d5e01a-299b-4265-8aed-a7be1f5ece44)


### Q11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?
```sql
SELECT count(distinct pro.customer_id) as customers_downgraded_pro_to_basic_monthly
FROM foodie_fi.subscriptions pro -- pro: pro monthly
JOIN foodie_fi.subscriptions bas on pro.customer_id = bas.customer_id -- bas: basic monthly
WHERE pro.plan_id = '2' and bas.plan_id = '1'
						and pro.start_date < bas.start_date 
						and extract(year from pro.start_date) = 2020
;
```

_Result:_

![image](https://github.com/user-attachments/assets/087012e7-1726-414f-b658-23a728087d57)


Thank you for stopping by, and I'm pleased to connect with you, my new friend!

Please do not forget to FOLLOW and star ⭐ the repository if you find it valuable.

I wish you a day filled with happiness and energy!

Warm regards,

Hien Moon
