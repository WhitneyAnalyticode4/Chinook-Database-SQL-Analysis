# Chinook-Database-SQL-Analysis

## Project Overview

This project analyzes the Chinook database using SQL Server and SQL Server Management Studio (SSMS).

The analysis covers database setup, core SQL queries, advanced SQL concepts, business-focused analysis, and query optimization.

The goal of the project was to use SQL to explore the database, connect related tables, identify patterns, answer business questions, and improve query performance.

## Tools Used
SQL Server
SQL Server Management Studio (SSMS)
SQL

## Database Overview

The Chinook database contains 11 core tables:

Artist
Album
Track
Genre
MediaType
Playlist
PlaylistTrack
Customer
Invoice
InvoiceLine
Employee

## Key Relationships
text
Artist
   ↓
Album
   ↓
Track

Track ↔ Playlist
       ↓
PlaylistTrack

Customer
   ↓
Invoice
   ↓
InvoiceLine
   ↓
Track

## Other relationships:
Track → Genre
Track → MediaType
Employee → Employee (through ReportsTo)
Employee → Customer (through SupportRepId)


## SQL Analysis
## 1. SELECT, WHERE and ORDER BY

Objective: Identify tracks longer than 5 minutes and arrange them from longest to shortest.
```sql
SELECT Name, Milliseconds
FROM Track
WHERE Milliseconds > 300000
ORDER BY Milliseconds DESC;
```
What I learned: The query uses SELECT to choose the required columns, WHERE to filter the tracks, and ORDER BY to sort the results.

## 2. GROUP BY and HAVING

Objective: Analyze revenue by billing country and focus on countries with more than 10 invoices.

```sql
SELECT BillingCountry,
       COUNT(*) AS InvoiceCount,
       SUM(Total) AS TotalRevenue
FROM Invoice
GROUP BY BillingCountry
HAVING COUNT(*) > 10
ORDER BY TotalRevenue DESC;
```
What I learned: GROUP BY allows the data to be grouped by country, while HAVING filters the grouped results.

## 3. Aggregate Functions

I used aggregate functions to examine track length across the catalogue.

```sql
SELECT
    AVG(Milliseconds) / 60000.0 AS AvgMinutes,
    MIN(Milliseconds) / 60000.0 AS MinMinutes,
    MAX(Milliseconds) / 60000.0 AS MaxMinutes
FROM Track;
```
The analysis calculated the average, minimum, and maximum track length, converted from milliseconds to minutes.

## ADVANCED SQL CONCEPTS
## INNER JOIN

Objective: Connect tracks with their album and artist information.

```sql
SELECT t.Name AS Track,
       al.Title AS Album,
       ar.Name AS Artist
FROM Track t
INNER JOIN Album al
    ON t.AlbumId = al.AlbumId
INNER JOIN Artist ar
    ON al.ArtistId = ar.ArtistId;
```
The INNER JOIN returns records where matching information exists across the related tables.

## LEFT JOIN

Objective: Identify customers who have no invoices.

```sql
SELECT c.CustomerId,
       c.FirstName,
       c.LastName,
       i.InvoiceId
FROM Customer c
LEFT JOIN Invoice i
    ON c.CustomerId = i.CustomerId
WHERE i.InvoiceId IS NULL;
```
The LEFT JOIN keeps every customer and then filters for customers without a matching invoice, identifying customers who have never made a purchase.

## RIGHT JOIN

Objective: Return customer information together with invoice details.

```sql
SELECT c.FirstName,
       c.LastName,
       i.InvoiceId,
       i.Total
FROM Invoice i
RIGHT JOIN Customer c
    ON i.CustomerId = c.CustomerId;
```
This keeps every customer and attaches invoice information where it exists.

## WINDOW FUNCTIONS
## RANK()

I used RANK() to rank customers by their total spending within their respective countries.

```sql
SELECT c.FirstName,
       c.LastName,
       c.Country,
       SUM(i.Total) AS TotalSpent,
       RANK() OVER (
           PARTITION BY c.Country
           ORDER BY SUM(i.Total) DESC
       ) AS SpendRank
FROM Customer c
JOIN Invoice i
    ON c.CustomerId = i.CustomerId
GROUP BY c.FirstName,
         c.LastName,
         c.Country;
```
This allows customers to be ranked within their own country rather than across the entire database.

## ROW_NUMBER()

I used ROW_NUMBER() to identify the most recent invoice for each customer.

```sql
SELECT *
FROM (
    SELECT i.*,
           ROW_NUMBER() OVER (
               PARTITION BY CustomerId
               ORDER BY InvoiceDate DESC
           ) AS rn
    FROM Invoice i
) sub
WHERE rn = 1;
```
The query numbers each customer's invoices from most recent to oldest and keeps the first row.

## BUSINESS ANALYSIS

## Top 10 Tracks by Revenue
```sql
SELECT t.Name AS Track,
       SUM(il.UnitPrice * il.Quantity) AS Revenue
FROM InvoiceLine il
JOIN Track t
    ON il.TrackId = t.TrackId
GROUP BY t.Name
ORDER BY Revenue DESC
OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;
```
This calculates revenue for each track and returns the 10 highest-earning tracks.

## Top 10 Artists by Revenue
```sql
SELECT ar.Name AS Artist,
       SUM(il.UnitPrice * il.Quantity) AS Revenue
FROM InvoiceLine il
JOIN Track t
    ON il.TrackId = t.TrackId
JOIN Album al
    ON t.AlbumId = al.AlbumId
JOIN Artist ar
    ON al.ArtistId = ar.ArtistId
GROUP BY ar.Name
ORDER BY Revenue DESC
OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;
```
This traces revenue through InvoiceLine → Track → Album → Artist and identifies the top 10 artists by revenue.

## Top 10 Customers by Spend
```sql
SELECT c.CustomerId,
       c.FirstName,
       c.LastName,
       SUM(i.Total) AS TotalSpent
FROM Customer c
JOIN Invoice i
    ON c.CustomerId = i.CustomerId
GROUP BY c.CustomerId,
         c.FirstName,
         c.LastName
ORDER BY TotalSpent DESC
OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;
```
This identifies the customers with the highest total spending.

## Monthly Revenue Trends
```sql
SELECT
    YEAR(InvoiceDate) AS Yr,
    MONTH(InvoiceDate) AS Mo,
    SUM(Total) AS MonthlyRevenue
FROM Invoice
GROUP BY YEAR(InvoiceDate),
         MONTH(InvoiceDate)
ORDER BY Yr, Mo;
```
This groups invoice revenue by year and month to show how revenue changes over time.

## Customer Purchasing Behaviour
```sql
SELECT c.CustomerId,
       c.FirstName,
       c.LastName,
       COUNT(i.InvoiceId) AS NumOrders,
       AVG(i.Total) AS AvgOrderValue
FROM Customer c
JOIN Invoice i
    ON c.CustomerId = i.CustomerId
GROUP BY c.CustomerId,
         c.FirstName,
         c.LastName
ORDER BY NumOrders DESC;
```
This analysis looks at the number of orders per customer and the average order value per customer, helping distinguish frequent buyers from customers with higher-value purchases.

## Query Optimization

Query performance was reviewed using the Actual Execution Plan in SSMS. The analysis focused on heavier multi-table queries, particularly the artist revenue query, to identify table scans and clustered index scans on larger tables such as Track and InvoiceLine.

## Indexes Added
```sql
CREATE INDEX IX_InvoiceLine_TrackId
ON InvoiceLine(TrackId);

CREATE INDEX IX_InvoiceLine_InvoiceId
ON InvoiceLine(InvoiceId);

CREATE INDEX IX_Invoice_CustomerId
ON Invoice(CustomerId);

CREATE INDEX IX_Track_AlbumId
ON Track(AlbumId);

CREATE INDEX IX_Album_ArtistId
ON Album(ArtistId);
```
These indexes were added to foreign key columns used in the join-heavy queries. The purpose was to allow SQL Server to use index seeks instead of scanning entire tables when resolving joins.

## Key SQL Skills Demonstrated

Database setup in SQL Server
Data retrieval with SELECT
Filtering with WHERE
Sorting with ORDER BY
Grouping with GROUP BY
Filtering grouped results with HAVING
Aggregate functions
INNER JOIN, LEFT JOIN, RIGHT JOIN
RANK(), ROW_NUMBER()
Revenue analysis
Customer analysis
Trend analysis
Query optimization
Index creation
Execution plan review
Project Outcome

The Chinook database project provided practical experience in using SQL Server to explore relational data, connect multiple tables, perform business-focused analysis, and optimize queries. It also strengthened my understanding of how SQL can be used to move from individual database tables to meaningful analysis and business insights.
