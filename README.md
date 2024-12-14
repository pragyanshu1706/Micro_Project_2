-- Group : 1
-- Topic : Address Payment Issues for Delivery Partners
-- Name : PRAGYANSHU (2215001254)

-- Calculate Partner Payout

UPDATE payouts 
SET payout = kg_delivered * per_kg_rate;

SELECT bp_id, payout from payouts p;


-- Calculate Total Cost

SELECT 
        bp.bp_id,
        (((1600 / vm.vehicle_mileage) * 80) + ((1600 * 3) + (1600 * 2) + m.driver_expenses)) as total_cost
    FROM vehicle_details vd
    JOIN business_partners bp ON vd.vehicle_type_id = bp.vehicle_type_id
    JOIN vehicle_mileage vm ON vd.vehicle_type = vm.vehicle_type
    JOIN maintenance m ON  vd.vehicle_type= m.vehicle_type;


-- Calculate Profit

SELECT 
        bp.bp_id,
        (p.payout -  (((1600 / vm.vehicle_mileage) * 80) + ((1600 * 3) + (1600 * 2) + m.driver_expenses))) as profit
    FROM vehicle_details vd
    JOIN business_partners bp ON vd.vehicle_type_id = bp.vehicle_type_id
    JOIN vehicle_mileage vm ON vd.vehicle_type = vm.vehicle_type
    JOIN maintenance m ON  vd.vehicle_type= m.vehicle_type
    JOIN payouts p ON bp.bp_id = p.bp_id ;


-- Calculate Profit/Loss Percentage

with total_cost as (
SELECT 
        bp.bp_id,
        (((1600 / vm.vehicle_mileage) * 80) + ((1600 * 3) + (1600 * 2) + m.driver_expenses)) as total_cost
    FROM vehicle_details vd
    JOIN business_partners bp ON vd.vehicle_type_id = bp.vehicle_type_id
    JOIN vehicle_mileage vm ON vd.vehicle_type = vm.vehicle_type
    JOIN maintenance m ON  vd.vehicle_type= m.vehicle_type
),
profit as (
SELECT 
        bp.bp_id,
        (p.payout -  (((1600 / vm.vehicle_mileage) * 80) + ((1600 * 3) + (1600 * 2) + m.driver_expenses))) as profit
    FROM vehicle_details vd
    JOIN business_partners bp ON vd.vehicle_type_id = bp.vehicle_type_id
    JOIN vehicle_mileage vm ON vd.vehicle_type = vm.vehicle_type
    JOIN maintenance m ON  vd.vehicle_type= m.vehicle_type
    JOIN payouts p ON bp.bp_id = p.bp_id
)
select p.bp_id, ((cast(p.profit as decimal)/ cast(tc.total_cost as decimal)) * 100) as pl_percent
from total_cost tc 
join profit p on p.bp_id = tc.bp_id ;



-- Categorize Partners (Overpaid, Underpaid or Normal)

with pl_percent as(
    with total_cost as (
	SELECT 
        	bp.bp_id,
        	(((1600 / vm.vehicle_mileage) * 80) + ((1600 * 3) + (1600 * 2) + m.driver_expenses)) as total_cost
    	FROM vehicle_details vd
    	JOIN business_partners bp ON vd.vehicle_type_id = bp.vehicle_type_id
    	JOIN vehicle_mileage vm ON vd.vehicle_type = vm.vehicle_type
    	JOIN maintenance m ON  vd.vehicle_type= m.vehicle_type
	),
	profit as (
	SELECT 
        	bp.bp_id,
        	((p.payout/10) -  (((1600 / vm.vehicle_mileage) * 80) + ((1600 * 3) + (1600 * 2) + m.driver_expenses))) as profit
    	FROM vehicle_details vd
    	JOIN business_partners bp ON vd.vehicle_type_id = bp.vehicle_type_id
    	JOIN vehicle_mileage vm ON vd.vehicle_type = vm.vehicle_type
    	JOIN maintenance m ON  vd.vehicle_type= m.vehicle_type
    	JOIN payouts p ON bp.bp_id = p.bp_id
	)
	select p.bp_id, p.profit, tc.total_cost, ((cast(p.profit as decimal)/ cast(tc.total_cost as decimal)) * 100) as pl_percent
	from total_cost tc 
	join profit p on p.bp_id = tc.bp_id
	)
select pp.bp_id,
	 CASE 
        WHEN pp.pl_percent < 0 THEN 'Underpaid'
        WHEN pp.profit < (0.25 * pp.total_cost) THEN 'Normal'
        ELSE 'Overpaid'
    END AS payment_status
    from pl_percent pp;



-- Explore Utilization and Profitability by Branch

with total_cost as (
SELECT 
        bp.bp_id,
        bp.bp_name,
        (((1600 / vm.vehicle_mileage) * 80) + ((1600 * 3) + (1600 * 2) + m.driver_expenses)) as total_cost
    FROM vehicle_details vd
    JOIN business_partners bp ON vd.vehicle_type_id = bp.vehicle_type_id
    JOIN vehicle_mileage vm ON vd.vehicle_type = vm.vehicle_type
    JOIN maintenance m ON  vd.vehicle_type= m.vehicle_type
),
profit as (
SELECT 
        bp.bp_id,
        (p.payout -  (((1600 / vm.vehicle_mileage) * 80) + ((1600 * 3) + (1600 * 2) + m.driver_expenses))) as profit
    FROM vehicle_details vd
    JOIN business_partners bp ON vd.vehicle_type_id = bp.vehicle_type_id
    JOIN vehicle_mileage vm ON vd.vehicle_type = vm.vehicle_type
    JOIN maintenance m ON  vd.vehicle_type= m.vehicle_type
    JOIN payouts p ON bp.bp_id = p.bp_id
)
select tc.bp_name,
	sum(po.payout) AS total_payout,
	sum(po.kg_delivered) AS total_kg_delivered,
	sum(p.profit) AS total_profit,
	(cast(sum(p.profit) as decimal)/ cast(sum(tc.total_cost) as decimal) * 100) as branch_profit_percentage
from total_cost tc 
join profit p on p.bp_id = tc.bp_id 
join payouts po on p.bp_id=po.bp_id 
group by 1;



-- Most Underpaid Partners

with pl_percent as(
    with total_cost as (
	SELECT 
        	bp.bp_id,
        	(((1600 / vm.vehicle_mileage) * 80) + ((1600 * 3) + (1600 * 2) + m.driver_expenses)) as total_cost
    	FROM vehicle_details vd
    	JOIN business_partners bp ON vd.vehicle_type_id = bp.vehicle_type_id
    	JOIN vehicle_mileage vm ON vd.vehicle_type = vm.vehicle_type
    	JOIN maintenance m ON  vd.vehicle_type= m.vehicle_type
	),
	profit as (
	SELECT 
        	bp.bp_id,bp.bp_name,
        	(p.payout -  (((1600 / vm.vehicle_mileage) * 80) + ((1600 * 3) + (1600 * 2) + m.driver_expenses))) as profit
    	FROM vehicle_details vd
    	JOIN business_partners bp ON vd.vehicle_type_id = bp.vehicle_type_id
    	JOIN vehicle_mileage vm ON vd.vehicle_type = vm.vehicle_type
    	JOIN maintenance m ON  vd.vehicle_type= m.vehicle_type
    	JOIN payouts p ON bp.bp_id = p.bp_id
	)
	select p.bp_id,p.bp_name, p.profit, tc.total_cost, ((cast(p.profit as decimal)/ cast(tc.total_cost as decimal)) * 100) as pl_percent
	from total_cost tc 
	join profit p on p.bp_id = tc.bp_id
	)
select pp.bp_id, pp.bp_name,
	pp.pl_percent
	from pl_percent pp
	where pp.pl_percent < 0
	order by pp.pl_percent;


-- Partners with Highest Profit Percentages

with pl_percent as(
    with total_cost as (
	SELECT 
        	bp.bp_id,
        	(((1600 / vm.vehicle_mileage) * 80) + ((1600 * 3) + (1600 * 2) + m.driver_expenses)) as total_cost
    	FROM vehicle_details vd
    	JOIN business_partners bp ON vd.vehicle_type_id = bp.vehicle_type_id
    	JOIN vehicle_mileage vm ON vd.vehicle_type = vm.vehicle_type
    	JOIN maintenance m ON  vd.vehicle_type= m.vehicle_type
	),
	profit as (
	SELECT 
        	bp.bp_id,bp.bp_name,
        	(p.payout -  (((1600 / vm.vehicle_mileage) * 80) + ((1600 * 3) + (1600 * 2) + m.driver_expenses))) as profit
    	FROM vehicle_details vd
    	JOIN business_partners bp ON vd.vehicle_type_id = bp.vehicle_type_id
    	JOIN vehicle_mileage vm ON vd.vehicle_type = vm.vehicle_type
    	JOIN maintenance m ON  vd.vehicle_type= m.vehicle_type
    	JOIN payouts p ON bp.bp_id = p.bp_id
	)
	select p.bp_id,p.bp_name, p.profit, tc.total_cost, ((cast(p.profit as decimal)/ cast(tc.total_cost as decimal)) * 100) as pl_percent
	from total_cost tc 
	join profit p on p.bp_id = tc.bp_id
	)
select pp.bp_id, pp.bp_name,
	pp.pl_percent
	from pl_percent pp
	where pp.pl_percent > 25
	order by pp.pl_percent;

