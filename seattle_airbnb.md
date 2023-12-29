## Seattle Airbnb Data Analysis
#### R, SQL and Tableau

### R Data Cleaning

This project is designed to do a deep dive on the Q4 Seattle Airbnb data to figure out the which neighborhoods are best for operating an Airbnb. 

We will start by downloading the most recent dataset from [Inside Airbnb](http://insideairbnb.com/get-the-data/).

I am starting this data analysis exploration with a quick R Overview.
This allows me to get a look at the structure of the data, the column names and the data types. 
I can then use this information to build my SQL database. 

Install tidyverse

```r
install.packages("tidyverse")
library(tidyverse)
```

Next I will import the csv and get a feel for the data structure

```r
airbnb_data <- read_csv("/Users/gunnarconley/Documents/Software Engineering/VSCodePractice/seattle_airbnb_project/seattle_airbnb_listings.csv")
head(airbnb_data)
View(airbnb_data)
colnames(airbnb_data)
```

I would like to separate the important data from the 'name' column for future analysis. Let's create a new dataframe using the separate function and separate the unit name, unit rating, unit type, bed count and bath count into their own columns. 

```r
airbnb_data_cleaned <- separate(airbnb_data,name,into=c('unit_name','unit_rating','unit_type','bed_count','bath_count'), sep = ' · ', fill = 'right')

view(airbnb_data_cleaned)
```

Next I will rename the id field to unit_id for better SQL usability.

```r
airbnb_data_cleaned <- airbnb_data_cleaned %>%
    rename(unit_id = id)
```

After reviewing the output of our separate function, it is clear that not every name separation was properly formatted. I will do some light data cleaning in excel before importing into SQL. Let's export this now first.

```r
write.csv(airbnb_data_cleaned, "/Users/gunnarconley/Documents/Software Engineering/VSCodePractice/seattle_airbnb_project/seattle_airbnb_listings_cleaned.csv", row.names=FALSE)
```

After successfully fixing an error in the separate function in Excel, I will now import the data to SQL for some window function analysis. 

### SQL (Window functions, rankings and data cleanup)

Now we will create a table inside our PostGre SQL database and upload the .csv data into our new table for analysis. 

```sql
CREATE TABLE airbnb_data (
	unit_id bigint NOT NULL,
    unit_name VARCHAR(150),
    unit_rating VARCHAR(5),
    unit_type VARCHAR(15),
    bed_count VARCHAR(20),
    bath_count VARCHAR(25),
	host_id INT,
	host_name VARCHAR (30),                     
	neighbourhood_group VARCHAR (25),
	neighbourhood VARCHAR (40),
	latitude DECIMAL(11,8),
	longitude DECIMAL (11,8),
	room_type VARCHAR (20),
	price INT,
	minimum_nights INT,
	number_of_reviews INT,
	last_review DATE,
	reviews_per_month DECIMAL,
	calculated_host_listings_count INT,
	availability_365 INT,
	number_of_reviews_ltm INT,
	license VARCHAR (40)
)
```
And now we will upload the .csv via the PostGre SQL UI. 

*Notice the `BIGINT` tag for `id` When I uploaded the .csv the first time there was an error that `id` was too large. 

After creating the table and importing the data, I realize that column names neighbourhood_group and neighbourhood are not easy to use, given that I don't typically spell "neighbourhood" with an added u. 

```sql
ALTER TABLE airbnb_data RENAME COLUMN neighbourhood_group TO neighborhood_group

ALTER TABLE airbnb_data RENAME COLUMN neighbourhood TO neighborhood
```

Let's move on to exploring the data..

### Data Exploration with SQL

I want to spend some time getting to know and understand our data so that my future analysis and visualization has a specific direction.  It will be good to understand how things can be segmented. 

```sql
SELECT *
FROM airbnb_data
```

I know the colnames are:

- unit_id
- unit_name
- unit_rating
- unit_type 
- bed_count
- bath_count
- host_id
- neighborhood_group
- neighborhood
- latitude
- longitude
- room_type
- price
- minimum_nights
- number_of_reviews
- last_review
- reviews_per_month
- calculated_host_listings_count
- availability_365
- number_of_reviews_ltm
- license

I know right off the bat that neighborhoods, room types, minimum nights, number of reviews, reviews per month, availability_365, number of reviews and license will be great columns for analysis. 

I also specifically went through the process of separating the unit rating, the unit type, bed and bath counts to create a better analysis. 

Also, it's important to note that we don't have the number of times booked for each unit so we will assume number_of_reviews as our best indicator of how 'successful' a unit is. 

### Data Analysis with SQL

The first set of analysis we will do is finding the average, minimum and maximum prices of airbnb listings in this data set using a simple OVER() window clause. 

```sql
SELECT 
	unit_name,
	unit_rating,
	number_of_reviews,
	neighborhood_group,
	price, 
	ROUND(AVG(price) OVER(),2),
	MIN(price) OVER(),
	MAX(price) OVER()
from airbnb_data
ORDER BY number_of_reviews desc;
```

We find that the average is $193.96, the minimum is $13 and the maximum is $10,000. 

This tells me that there will be some outliers in our dataset as $13 and $10,000 are not as reliable datapoints for the average airbnb. 

Next, I am curious which units are under $100 per night so let's take a look at that. 

```sql
SELECT 
	price,
	room_type,
	unit_name,
	neighborhood_group,
	minimum_nights,
	number_of_reviews,
	ROUND(AVG(price) OVER(),2)
from airbnb_data
WHERE price <= 100
ORDER BY number_of_reviews desc;
```

Based on a hunch, I ordered by number_of_reviews and specifically selected room_type, name, minimum_nights and number_of_reviews. 

Via a quick scroll of the 1400 or so units that were returned, it seems that small units, whether it's a guest suite, tiny home or detached dwelling unit seem to do quite well being priced just below $100 and have a minimum stay of 1-2 nights. 

Let's do the same calculation only this time we will segment by price over 400 per night. 

```sql
SELECT 
	room_type,
	price,
	unit_rating,
	number_of_reviews,
	neighborhood_group,
	minimum_nights,
	number_of_reviews,
	ROUND(AVG(price) OVER(),2)
from airbnb_data
WHERE price >= 400
ORDER BY number_of_reviews desc
```
Looks like of the 400 or so units that are returned, many of them have a minimum night stay of 2-3+. 

Let's do the same calculation only this time we will segment by number_of_reviews over 100. 

```sql
SELECT 
	price,
	room_type,
	unit_rating,
	number_of_reviews,
	unit_type,
	bed_count,
	bath_count,
	neighborhood_group,
	minimum_nights,
	number_of_reviews,
	ROUND(AVG(price) OVER(),2) AS avg_price
from airbnb_data
WHERE number_of_reviews >= 500	
ORDER BY number_of_reviews desc
```
Interestingly, the data shows all types and sizes of units. From just looking at the tables returned, it would be difficult to infer any meaningful conclusions. 

Since we have established that number of reviews is likely our best bet for understanding what a successful airbnb listing looks like (as it correlates to number of lifetime bookings) lets find the average number of reviews and zoom in to that segment of the data. We will bracket the data between 10-500 reviews to remove those outliers. 

```sql
SELECT 
	ROUND(AVG(number_of_reviews) OVER()) AS avg_num_reviews
from airbnb_data
WHERE number_of_reviews BETWEEN 10 AND 500;
```

We get an average number of reviews at around 94. 




Let's find the difference from the average price: 

```sql
SELECT 
	unit_id,
	unit_name,
	neighborhood_group,
	price, 
	ROUND(AVG(price) OVER(),2),
	ROUND((price-AVG(price) OVER()),2) AS diff_from_avg
from airbnb_data;
```
How about the percent of average price? 
```sql
SELECT 
	unit_id,
	unit_name,
	neighborhood_group,
	price, 
	ROUND(AVG(price) OVER(),2),
	ROUND((price / AVG(price) OVER() * 100),2) AS percent_of_avg
from airbnb_data;
```

Let's find the percent difference from average price

```sql
SELECT 
	unit_id,
	unit_name,
	neighborhood_group,
	price, 
	ROUND(AVG(price) OVER(),2),
	ROUND((price / AVG(price) OVER() - 1) * 100,2) AS percent_diff_of_avg
from airbnb_data;
```

Ok that's great. Window functions allow us to see more info tho. Let's use them to partition by neighborhood_group and find the average price of each neighborhood_group. 

```sql
SELECT 
	unit_id,
	unit_name,
	neighborhood,
	neighborhood_group,
	price, 
	AVG(price) OVER(PARTITION BY neighborhood_group) AS avg_price_neighborhood_group
FROM airbnb_data;
```
Awesome, that created a column that shows the average price for each neighborhood group (i.e. Ballard has an average price of $186.41 per night whereas Beacon Hill has an average of $158.68 per night).

Let's quickly add neighborhoods and round by 2 to see more data. 

```sql
SELECT 
	unit_id,
	unit_name,
	neighborhood,
	neighborhood_group,
	price, 
	ROUND(AVG(price) OVER(PARTITION BY neighborhood_group),2) AS avg_price_neighborhood_group,
	ROUND(AVG(price) OVER(PARTITION BY neighborhood),2) AS avg_price_neighborhood
FROM airbnb_data;
```

Create a breakdown by neighborhood and neighborhood group of each average price. 

```sql
SELECT 
	DISTINCT neighborhood,
	neighborhood_group,
	ROUND(AVG(price) OVER(PARTITION BY neighborhood),2) AS avg_price_neighborhood,
	ROUND(AVG(price) OVER(PARTITION BY neighborhood_group),2) AS avg_price_neighborhood_group
FROM airbnb_data
```


Now looking ahead, we could spend all day crafting queries to satisfy our curiosity. At this point for us to really understand the data it's time to step into Tableau for a visualization. 

### Tableau Visualization 

The final step in analysis is visualizing the information I have been cleaning and analyzing with R and SQL. 

I was interested in finding out the correlation between neighborhoods, review performance and unit size. I created four visualizations, all with the ability to filter by unit type: 

- Entire Home/Apt
- Private Room
- Shared Room
- All

The first visualizaiton I created was a map of Seattle with the neighborhoods outlined by average price. At first glance it doesn't tell us much, but as a casual user, you can zoom into a specific neighborhood and find the average price. 


![viz1](https://github.com/gunnarconley/seattle-airbnb-2023/blob/main/viz1.png?raw=true)

Perhaps most important is understanding the chepaest and most expensive neighborhoods by average price. 

I created two separate visualizations to showcase this: 

![viz2](https://github.com/gunnarconley/seattle-airbnb-2023/blob/main/viz2.png?raw=true)
![viz3](https://github.com/gunnarconley/seattle-airbnb-2023/blob/main/viz3.png?raw=true)

Lastly I created a chart that showcases the reviews by month (AKA booking volume). This shows the Summer months are the best months for Airbnb bookings, which shouldn't be a huge shock. 

![viz4](https://github.com/gunnarconley/seattle-airbnb-2023/blob/main/viz4.png?raw=true)

[View the full visualization on Tableau's website here](https://public.tableau.com/views/SeattleAirbnbMarketAnalysisSept_2023/SeattleAirbnbDashboard?:language=en-US&:display_count=n&:origin=viz_share_link) 

Thanks for viewing this analysis! 