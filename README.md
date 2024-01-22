# data_portfolio
Data Analytics &amp; ML portfolio

#### Analysis was done by myself on the airline seat-booking system(TASS) with some questions taken from my BT2102 assignment and some being my own data exploration on the dataset. 

Q1) Write an SQL statement to find the number of customers who made their first order in each country, each day -> so I first found out when and where each customer made his first order then use it to find answer

````sql
    use tass;

WITH temptable AS (
  SELECT c.Country, MIN(b.bookingDate) AS first_order_date
  FROM customer c 
  RIGHT JOIN booking b
  ON c.customerID = b.CustomerID
  GROUP BY c.customerID
)

SELECT t.Country, t.first_order_date, COUNT(t.first_order_date)
FROM temptable t
GROUP BY t.Country, t.first_order_date
ORDER BY t.Country, t.first_order_date;
````
* Results (limited to first 5 rows)
  * Begonia	2022-10-19 00:00:00	1
  * Begonia	2022-10-20 00:00:00	1
  * Begonia	2022-11-14 00:00:00	1
  * Begonia	2022-11-15 00:00:00	1

 
Q2) Find the 5 flights with the lowest number of bookings & select the flights & airlines with the least number of bookings

```sql
CREATE VIEW temporarytable AS (
SELECT subquery.FlightNumber, airline.AirlineName,  subquery.OriginAirportCode, subquery.DestinationAirportCode
FROM
  (SELECT FlightNumber, AVG((BookedBusinessSeats + BookedEconomySeats) / 
  (TotalBusinessSeats + TotalEconomySeats)) AS percentage, OriginAirportCode, DestinationAirportCode
  FROM flightavailability
  GROUP BY FlightNumber
  ORDER BY percentage
  LIMIT 5) subquery
LEFT JOIN flight ON subquery.FlightNumber = flight.FlightNumber
LEFT JOIN airline ON flight.AirlineCode = airline.AirlineCode
);
````
* Results:
  * FN056	Gro	AP05	AP04
  * FN012	CarAir	AP09	AP10
  * FN052	RoseAir	AP09	AP02
  * FN022	GauAir	AP05	AP03
  * FN006	TulAir	AP02	AP05

Q3) Is there a chance that these routes are the same -> denormalise so that country & city can be easily obtained from airport Code

````sql
CREATE VIEW airport_city_country AS (
SELECT a.airportCode, c.CityCode, c.CityName, country.CountryCode, country.CountryName
FROM airport a
JOIN city c ON a.CityCode = c.CityCode
JOIN country ON c.CountryCode = country.CountryCode
);

SELECT t.FlightNumber, 
(SELECT airport_city_country.CountryName FROM airport_city_country WHERE airport_city_country.airportCode = t.OriginAirportCode) as ori,
(SELECT airport_city_country.CountryName FROM airport_city_country WHERE airport_city_country.airportCode = t.DestinationAirportCode) as dest
FROM temporarytable t;
````

* FN056	Gaura	Carnation
* FN012	Rose	Rose
* FN052	Rose	Begonia
* FN022	Gaura	Carnation
* FN006	Begonia	Gaura

## From this we are able to observe that routes departing from Gaura,Rose & flights towardrs Carnation,Rose & Begonia are least frequented. Hence, we can consider if we should shut these down as they have the highest vacanies in %.

Q4) Calculate the percentage of foreigners in the customer list. 
A Foreigner is identified as people whose country in mail address is not the same as his/her nationality. 
Display the percentage as ForeignerPercent. Round the percentage up to 2 decimal places, e.g. 50.99%.

````sql
# create a Common Table Expression (CTE) for easy usage
WITH num as
(SELECT COUNT(c.customerID) AS ftot
FROM customer c
WHERE c.Nationality != c.Country) ,
denom as 
(SELECT COUNT(c.customerID) AS tot
FROM customer c)
SELECT CONCAT(ROUND((ftot/tot * 100),2),"%") AS ForiegnerPercent
FROM num,denom;
````

* 93.33%

Q5) List all Bookings placed between "2022-10-16" and "2022-11-15" which have not gotten any payments yet. 
"placed" refers to the booking that has been made, however, it does not mean this booking is confirmed. 
You also do not have to consider booking status. Display BookingID only.

````mysql
SELECT b.BookingID
FROM booking b
WHERE b.BookingID NOT IN
(SELECT DISTINCT BookingID FROM payment)
AND b.BookingDate BETWEEN DATE('2022-10-16') AND DATE('2022-11-15')
GROUP BY b.BookingID
ORDER BY b.BookingID;
````
## Since money has not been receieved make sure to flag these emails and remind customers to pay

Q6) Find the full name of customers who do not have any email address but have phone number, sort them 
in ascending order
````mysql
SELECT c.CustomerID, CONCAT(c.FirstName, c.LastName) AS full_name
FROM customer c
INNER JOIN customerphone cp ON c.CustomerID = cp.CustomerID
WHERE c.customerID NOT IN (SELECT customerID from customeremail) AND 
c.customerID IN (SELECT customerID from customerphone GROUP BY customerID Having count(customerID) = 1)
ORDER BY c.customerID;
````
* Results (first 5 rows only)
  * CM019	VeraThomas
  * CM020	JeanThomas
  * CM023	LilithEvans
  * CM031	EricaSmith
  * CM058	NicoleWilson
 
## Make sure to inform these customers to give their email to send their booking copy
 
Q7) Rank airports based on total number of flights arrival and departure in weekdays. Display AirportCode and rank. 
(Airports with the same number of flights are treated as having the same rank.
````mysql
# get count for origin airports
WITH sub AS (
	SELECT fa.OriginAirportCode, COUNT(fa.OriginAirportCode) as ocount
	FROM flightavailability fa
	WHERE WEEKDAY(fa.DepartureDateTime) < 5
	GROUP BY fa.OriginAirportCode),
    
# count for destination airports
sub2 AS (
	SELECT fa.DestinationAirportCode, 
	COUNT(fa.DestinationAirportCode) as dcount
	FROM flightavailability fa
	WHERE WEEKDAY(fa.ArrivalDateTime) < 5
	GROUP BY fa.DestinationAirportCode)
    
# use dense_rank() to ensure that airports with same num of flights have the same rank
SELECT sub.OriginAirportCode, dense_rank() OVER ( order by(sub.ocount + sub2.dcount) desc) as total
FROM sub
INNER JOIN sub2 ON
sub2.DestinationAirportCode = sub.OriginAirportCode;
````
* AP05	GalAirport	1
* AP09	RadAirport	2
* AP11	ThoAirport	2
* AP03	CopAirport	3

## Can give galAirport a prize for being the best airport handling the most number of flights in weekdays

Q7) Which are the top countries with the highest average paying customers, 
as well as ranking them in their respective countries
We shall consider the full amount (paid amount + balance) here

````mysql
SELECT c.customerID, 
c.Country,
SUM(p.balance + p.PaidAmount) AS total,
row_number() OVER(PARTITION BY c.Country ORDER BY ROUND(SUM(p.balance + p.PaidAmount),2)) as in_country_ranking
FROM customer c
INNER JOIN payment p
ON c.customerID = p.CustomerID
GROUP BY c.CustomerID
ORDER BY c.Country;
````
Q8) How much does airport tax account for each airline's profits, order by this amount
first for each flight find the total amount earned & total airport tax paid then use this information to calculate the airport tax
Give the profit ranking and provide the ranking values for this in the same table as well

````sql
# need to alter this table to include the airportTax
ALTER VIEW airport_city_country AS (
SELECT a.airportCode, c.CityCode, c.CityName, country.CountryCode, country.CountryName, a.airportTax
FROM airport a
JOIN city c ON a.CityCode = c.CityCode
JOIN country ON c.CountryCode = country.CountryCode
);

# get all the required fields in a single table
WITH tempview AS (
SELECT b.BookingID, b.FlightNumber,  

(SELECT AirlineName 
FROM airline 
WHERE AirlineCode = (SELECT AirlineCode FROM flight WHERE flight.FlightNumber = f.FlightNumber)) as airline_name,

(SELECT airportTax as ori_tax FROM airport_City_country WHERE airportcode = f.OriginAirportCode) as ori_tax,
(SELECT airportTax as dest_tax FROM airport_City_country WHERE airportcode = f.DestinationAirportCode) as dest_tax,
b.TotalPrice
FROM booking b 
LEFT JOIN flightavailability f ON b.FlightNumber = f.FlightNumber
)

# this extra query is due to make created columns look nicer instead of messily doing it above
SELECT t.airline_name,SUM((t.ori_tax + t.dest_tax) / (t.TotalPrice)) as percentage, 
SUM(t.totalPrice - t.ori_tax - t.dest_tax) as profit,
RANK() OVER(ORDER BY SUM(t.totalPrice - t.ori_tax - t.dest_tax) DESC) AS profit_ranking
FROM tempview t
GROUP BY airline_name
ORDER BY percentage DESC;
````

* Results (airline_name, percentage, profit, profit_ranking)
* Gro	42.12509111013744	1320767.9609375	1
* GauAir	39.124484659454716	618098.5869216919	4
* CarAir	35.94569103702192	819599.5782165527	3
* IpoAir	23.454337102429104	1066484.2144470215	2
* TulAir	13.139882533086572	395186.9242553711	5
* RoseAir	9.94487556692947	281584.36334228516	6

### This tells me that almost 42% of flight revenue goes into airport tax hence while it may seem that Gro is losing a lot of money, it is infact making a lot more in profit. While GauAir is paying 39% of its profits to airport tax but is only 4th in profitability, this suggests that they should instead reconsider if their route is worth continuting or not
