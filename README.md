# SQL-Project-answering-business-questions
This SQL project answers business questions ranging from marketing department helping them on marketing campaign offers, delivery team on making efficient deliveries by pulling customers that exist in the same address/location, explore the sales data deeper by finding the top selling product.

/*
Utilized Joins, operators, subqueries, Group and having to answer real business questions. The data source explored is for WOWI sales
*/

---UNDERSTAND THE DATABASE AND THE DATA 

---1. IS THE DATABASE CASE INSENSITIVE?
SELECT 
CONVERT(varchar(256),SERVERPROPERTY('Collation')) as collation

--2. How many rows does the factsale tables have? 228,265 rows 

select COUNT(*)
from factSale

---3. How would you best describe the customer we have?

SELECT DISTINCT DC.Category
FROM dimCustomer AS DC 

---4. How many rows does Dimcustomer table have? 403

Select COUNT(*) as Number_of_rows
from dimCustomer

---5. Does dimCustomer have entry for unkown? yes 

---6. How many customers do we have? 403
select COUNT(distinct [WWI Customer ID]) 
from dimCustomer

---7. How many FK does the dimStockItem table have? 1
---8.How many rows are in dimStockItem table? 229 rows
select COUNT(*)
from dimStockItem

---9.How many known products are we dealing with? 228
SELECT COUNT(*) 
FROM dimStockItem
WHERE [Stock Item]<>'UNKNOWN'

--10. What does IsChiller Stock column means? does the stock items need to be refrigrated.
select *
from dimStockItem

---MARKETING 
---11. What is the unit price of the cheapest product that we currently sell excluding unknown items?
SELECT MIN([Unit Price])
FROM factSale fs 
WHERE fs.[Customer Key] <> 0

---12. what is the name of the cheapest product that we currently sale? 3kg courier post bag(white) 300X190X95mm
SELECT 
[Stock Item] AS ProductName,
[Unit Price]
FROM dimStockItem
WHERE [Stock Item]<>'unknown'

Order by [Unit Price] ASC


--13. what is the name of the cheapest product that we currently sale? remove names like bag,carton and box as those are packaging materials. Ans: packing knife with metal insert blade(yellow) 9mm
SELECT 
[Stock Item] AS ProductName,
[Unit Price]
FROM dimStockItem
WHERE [Stock Item]<>'unknown'
AND [Stock Item] NOT LIKE '%Box%'
AND [Stock Item] NOT LIKE '%Carton%'
AND [Stock Item] NOT LIKE '%Bag%'
Order by [Unit Price] ASC


---Alternative solution to 12 is using subquery in the WHERE statement 

SELECT 
[Stock Item] AS ProductName,
[Unit Price] as UnitPrice 

FROM dimStockItem

WHERE [Unit Price]= (SELECT MIN([Unit Price]) FROM dimStockItem WHERE [Stock Item]<>'UNKNOWN')


---Alternative solution to number 13 above is also to use subqueries instead of order by.
SELECT 
[Stock Item] AS ProductName,
[Unit Price] as UnitPrice 

FROM dimStockItem

WHERE [Stock Item Key] != 0
AND [Stock Item] NOT LIKE '%Box%'
AND [Stock Item] NOT LIKE '%Carton%'
AND [Stock Item] NOT LIKE '%Bag%'

Order by [Unit Price] ASC

---The marketing team have gotten back.
/* since the winner of the social media campaign might be a child, they don't want to send a knife. 
Instead they have suggested that we send a mug or a shirt  to the chosen winner.
The product should also be black if possible 
*/
---1. Find list of product that contain mug or shirt in their name. How many are there? 68 
SELECT 
[Stock Item] as ProductName 

FROM dimStockItem

WHERE [Stock Item] LIKE '%mug%' 
OR [Stock Item] LIKE '%Shirt%'

---2 How many products meets this description and are also black? 34

SELECT 
COUNT(*)  As  totalproducts

FROM dimStockItem

WHERE ([Stock Item] LIKE '%mug%' 
OR [Stock Item] LIKE '%Shirt%')
AND Color='Black'


---3. what is WWI Stock Item Id of the cheapest item meeting the above conditions? If multiple products have the same price choose the one with the lowest stock ID. ANS:17

SELECT 
[WWI Stock Item ID],
[Stock Item],
[Unit Price] AS UnitPrice,
Color

FROM dimStockItem
WHERE ([Stock Item] LIKE '%mug%' 
OR [Stock Item] LIKE '%Shirt%')
AND Color ='BlACK'

ORDER BY [Unit Price], [WWI Stock Item ID]

---4. What is the markup  of WWI Stock Item ID 29? Markup= (retail price-unit price)/unit price. ANS: 0.495

SELECT 
([Recommended Retail Price]- [Unit Price])/[Unit Price] AS Markup
FROM dimStockItem
WHERE [WWI Stock Item ID]=29


/* DELIVERY GROUP:
The delivery group is exploring idea for delivery efficiency. instead of delivering to individual stores, the team would like to group deliveries by postcodes or
Buying group. Buying group purchase inventory on behalf of the stores within the group. 
*/

---How many customers are in each buying group? 201

SELECT 
[Buying Group],
COUNT(*) as NumberOfCustomers 
FROM dimCustomer

WHERE [Customer Key]<>0

GROUP BY [Buying Group]
ORDER BY [NumberOfCustomers]

---we are trailing new delivery with wingtip toys. we need to identify clusters of shop from this buying group that are near each other.

---Do any postcodes have more than 3 wingtip toys shop? if so, which postcodes? ANSWER: YES, 90683


SELECT 
[Postal Code]
FROM dimCustomer
WHERE [Buying Group]='Wingtip Toys'

GROUP BY [Postal Code]

HAVING COUNT([Customer Key])>3


---If postal code is identified, then which stores/customers will be included in the delivery trials? wrap the above query as subquery in the where clause and add extra condition for the buying group.

SELECT [Postal Code],
Customer
FROM dimCustomer

WHERE [Postal Code] IN (SELECT 
                        [Postal Code]
                        FROM dimCustomer
                        WHERE [Buying Group]='Wingtip Toys'

                        GROUP BY [Postal Code]

                        HAVING COUNT([Customer Key])>3)
                        AND [Buying Group]='Wingtip Toys'



/* OTHER DATA: Exploring other data*/

---what proportion of our employees work in sales?


SELECT COUNT(*)
FROM dimEmployee
WHERE [Employee Key]<>0

SELECT 
CAST(CAST(COUNT(*) AS decimal(8,2))/(SELECT COUNT(*)
            FROM dimEmployee
            WHERE [Employee Key]<>0) AS decimal(8,4)) Slespeople 

FROM dimEmployee

WHERE [Is Salesperson]=1


---CITY DIMESNISON
---Which sales territory has highest population? Southeast,
----What is the total number of cities in this region? 9063
---what is the highest population in the largest city in this region? 821784
---what is the Total population across the territory? 227338926

SELECT *
FROM dimCity

SELECT 
ISNULL([Sales Territory], 'Total') as SalesTerritory,
sum([Latest Recorded Population]) as TotalPop,
COUNT([City Key]) AS NumberOfCities,
Max([Latest Recorded Population]) As HighestPopCity
FROM dimCity

WHERE [City Key]<>0

GROUP BY ROLLUP([Sales Territory])

ORDER BY TotalPop DESC 





---ADVANCED QUERIES: LINKING MULTIPLE TABLES 

---what granulity is factsale table 
SELECT *
FROM factSale

---What type of relationship exist between factsale and dimDate table?


---what is the maximum fiscal year in a dimDate? 2017

SELECT 
MAX([Fiscal Year])
FROM dimDate

---How many fiscal years do we have sales data for? 4

SELECT 
dd.[Fiscal Year] AS FiscalYears

FROM factSale fs 
JOIN dimDate dd 
ON fs.[Invoice Date Key]=dd.[Date]
GROUP BY dd.[Fiscal Year]
ORDER BY FiscalYears


---calculate a report of sales excluding tax,profit, quantity sold. You may need to analyze data by fiscal month or year

--what were the sales excluding tax in fiscal year 2015?

SELECT 
    dd.[Fiscal Year],
    sum([Total Excluding Tax]) as TotalSales,
    sum(Quantity) as QuantitySold,
    SUM(Profit) as Profit
FROM factSale fs 
JOIN dimDate dd ON dd.[Date]=fs.[Invoice Date Key]

GROUP BY dd.[Fiscal Year]

ORDER BY Profit 
/*questions:
1. what were the sales excluding tax in fiscal year 2015? 53827320.95
2.Which fiscal year appears to have highest profit?  2015
3.What explanation can you offer as to why profit in 2016 is significantly lower than 2015?
         reasons: less quantities were sold in 2016 but to explore it further, we need to fetch the data for 2015 and 2016 by fiscal month and see the record of sales through the periods.


*/

SELECT 
dd.[Fiscal Month Number] as FiscalMonth,
dd.[Fiscal Month Label],
dd.[fiscal year],
SUM(fs.[Total Excluding Tax])as Sales,
SUM(Quantity) AS Quantity,
SUM(Profit) as Profit
FROM factSale fs
JOIN dimDate dd 
ON dd.[Date]=fs.[Invoice Date Key]

WHERE dd.[Fiscal Year] in (2015,2016)

GROUP BY dd.[fiscal year],dd.[Fiscal Month Label],dd.[Fiscal Month Number]

ORDER BY dd.[Fiscal Year] desc,FiscalMonth desc

---From running the above query it is evident that the reason for small profit in 2016 is because the sales record is only for 7 months and not the whole year.




---TOP SELLING PRODUCT QUERIES 
--1.What were the total sales  excluding tax in fiscal year 2016? 31M

SELECT 
    SUM([Total Excluding Tax]) AS TotalSales
FROM factSale fs 
    JOIN dimDate dd 
    ON dd.[Date]=fs.[Invoice Date Key]
WHERE dd.[Fiscal Year] IN(2016)


--2 What was the top selling product in fiscal 2016 so far?

SELECT TOP(1)
si.[Stock Item],
SUM([Total Excluding Tax]) as TotalSales
FROM factSale fs 
    JOIN dimDate dd
    ON dd.[Date]=fs.[Invoice Date Key]
    JOIN dimStockItem si 
    ON si.[Stock Item Key]=fs.[Stock Item Key]

WHERE dd.[Fiscal Year] IN (2016)

GROUP BY si.[Stock Item]

ORDER BY TotalSales desc 


---3 What was the top performing product /salesperson combination  in fiscal 2016?

SELECT 
de.Employee as Salesperon,
si.[Stock Item] as StockItem,
sum([Total Excluding Tax])as TotalSales
FROM factSale fs 
    JOIN dimDate dd 
    ON fs.[Invoice Date Key]=dd.[Date]
    JOIN dimStockItem si 
    ON si.[Stock Item Key]=fs.[Stock Item Key]
    JOIN dimEmployee de 
    ON de.[Employee Key]=fs.[Salesperson Key]

WHERE dd.[Fiscal Year] IN (2016)

GROUP BY de.Employee, si.[Stock Item]

ORDER BY TotalSales desc


--4. what proportion of the total sales in 2016 does the top performances represent?

SELECT TOP(10)
de.Employee as Salesperon,
si.[Stock Item] as StockItem,
FORMAT(CAST(sum([Total Excluding Tax])/(SELECT 
                        SUM([Total Excluding Tax]) AS TotalSales
                    FROM factSale fs 
                        JOIN dimDate dd 
                        ON dd.[Date]=fs.[Invoice Date Key]
                    WHERE dd.[Fiscal Year] IN(2016)) AS decimal(8,6)),'P4') AS PctOfSalesYTD
FROM factSale fs 
    JOIN dimDate dd 
    ON fs.[Invoice Date Key]=dd.[Date]
    JOIN dimStockItem si 
    ON si.[Stock Item Key]=fs.[Stock Item Key]
    JOIN dimEmployee de 
    ON de.[Employee Key]=fs.[Salesperson Key]

WHERE dd.[Fiscal Year] IN (2016)

GROUP BY si.[Stock Item],de.Employee

ORDER BY PctOfSalesYTD desc


---thinking about to make the query dynamic such that it always fetches the recent year sales
/*First let's return the latest year in the sales. watchout not to use only the date dimension to return the lates date as the year might be listed on dimension table but has no 
data in the sales table. as such we join the two tables to fetch the latest year with sale by using max function on the fiscal column in the combined table.*/

SELECT MAX(dd.[Fiscal Year])
FROM dimDate dd 
JOIN factSale fs 
ON fs.[Invoice Date Key]=dd.[Date]

--NOW THE QUERY WILL APPEAR AS BELOW:

SELECT TOP(10)
de.Employee as Salesperon,
si.[Stock Item] as StockItem,
FORMAT(CAST(sum([Total Excluding Tax])/(SELECT 
                                            SUM([Total Excluding Tax]) AS TotalSales
                                        FROM factSale fs 
                                            JOIN dimDate dd 
                                            ON dd.[Date]=fs.[Invoice Date Key]
                                        WHERE dd.[Fiscal Year] IN(SELECT MAX(dd.[Fiscal Year])
                                                                    FROM dimDate dd 
                                                                        JOIN factSale fs 
                                                                        ON fs.[Invoice Date Key]=dd.[Date])) 
 AS decimal(8,6)),'P4') AS PctOfSalesYTD 
FROM factSale fs 
    JOIN dimDate dd 
    ON fs.[Invoice Date Key]=dd.[Date]
    JOIN dimStockItem si 
    ON si.[Stock Item Key]=fs.[Stock Item Key]
    JOIN dimEmployee de 
    ON de.[Employee Key]=fs.[Salesperson Key]

WHERE dd.[Fiscal Year] IN (SELECT MAX(dd.[Fiscal Year])
                            FROM dimDate dd 
                            JOIN factSale fs 
                            ON fs.[Invoice Date Key]=dd.[Date])

GROUP BY de.Employee, si.[Stock Item]

ORDER BY PctOfSalesYTD desc

--CHILLER PRODUCTS 

SELECT 
si.[Stock Item] as Product,
SUM(fs.Quantity) AS QuantitySold
FROM factSale fs 
RIGHT JOIN dimStockItem si 
ON si.[Stock Item Key]=fs.[Stock Item Key]

WHERE si.[Is Chiller Stock]=1

GROUP BY si.[Stock Item]

ORDER BY QuantitySold DESC
