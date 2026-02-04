--how many quanity was sold in a month, total sales and total customers

select datetrunc(month,order_date) as order_date, sum(sales_amount) as total_sale,
count(distinct customer_key) as total_customers,
sum(quantity) as total_quantity
from fact_sales
where order_date is not null
group by datetrunc(month,order_date)
order by datetrunc(month,order_date)



--calculate the total sales per month 
--and the running total of sales over time
--running average of price
select order_date, total_sales,
sum(total_sales) over ( order by order_date) as running_sales ,
avg(avg_price) over( order by order_date) as moving_avg
from
(
select 
datetrunc(month, order_date) as order_date,
sum(sales_amount) as total_sales,
avg(price) as avg_price
from fact_sales
where order_date is not null
group by datetrunc(month, order_date)


) t

order by order_date


--analyze the yearly performance of products by comparing their sales to both average sales performance and the previous years sales


select order_year,current_sales,product_name,avg(current_sales) over (partition by product_name order by order_year) as avg_sales,
current_sales - avg(current_sales) over (partition by product_name order by order_year) as dif_sales,
case 
     when current_sales - avg(current_sales) over (partition by product_name order by order_year) >0 then 'above avg'
     when current_sales - avg(current_sales) over (partition by product_name order by order_year) <0 then 'below avg'
     else 'avg'
end     average_change,lag(current_sales) over(partition by product_name order by order_year) as py_sales,
current_sales - lag(current_sales) over(partition by product_name order by order_year) as diff_py,
case
     when current_sales -  lag(current_sales) over(partition by product_name order by order_year) >0 then 'increase'
     when current_sales -  lag(current_sales) over(partition by product_name order by order_year) <0  then 'decrease'
     else 'no change'
end py_change
from
(

select year(f.order_date) as order_year,p.product_name,sum(f.sales_amount) as current_sales
from fact_sales as f left join dim_products as p
on f.product_key = p.product_key
where year(f.order_date) is not null
group by year(f.order_date),p.product_name
) t

-- what category contribute the most to overall sales
select sum(sum_sales) over () as overall_sales,sum_sales, category,

concat(round((cast(sum_sales as float) /sum(sum_sales) over ())* 100,2), '%') as  per_total

from

(

select p.category ,sum(f.sales_amount)  as sum_sales  from fact_sales as f left join dim_products as p
on f.product_key = p.product_key
group by p.category
) t
order by sum_sales desc


--segment products into cost ranges and count how many products fall into each segment

select range_cost,count(product_key) as total_products
from(

select product_key,product_name,cost,
case
     when cost<100 then' below 100'
     when cost between 100 and 500 then '100-500'
     when cost between 500 and 1000 then '500-1000'
     else 'above 1000'
end range_cost
from dim_products) t

group by range_cost






--group customers into 3 segments based on spending behaviour
--vip   at least 12 months of history and more than 5000 spent
--regular at least 12 months of history and less than 5000 spent
--new  lifespan less than 12 months
--and find total no of customers by each group






with customer_spending as (


select c.customer_key, sum(f.sales_amount) as total_spending,max(order_date) as lastoder , min(order_Date) as first_order,
datediff(month,min(order_date), max(order_Date)) as life_span from fact_sales as f
left join dim_customers as c
on f.customer_key = c.customer_key
group by c.customer_key
 )

select
case 
    when life_span >=12 and total_spending >5000 then 'vip'
    when life_span >=12 and total_spending <5000 then 'regular'
    else 'new'
end customer_seg,
count(customer_key) as countof_customers
from customer_spending
group by (case 
    when life_span >=12 and total_spending >5000 then 'vip'
    when life_span >=12 and total_spending <5000 then 'regular'
    else 'new'
end)
order by countof_customers desc











