###Problem

Here we have a dataset of heartbleed vulnerable ip addresses. We'd like to count how many vulnerable ip addresses belong to each city
where the ip->city mapping is defined by the db-ip geolocating database. Then we'd like to calculate the vulnerable ips per capita for the top 10
most populous cities where population is provided by the geonames databse. 


###Approach

Our approach is to import all 3 datasets into postgres where they can be queryied with standard sql techniques.

###Aquire Data

	curl -O "https://scans.io/data/umich/heartbleed-public/https-full/https-full-20140424T1025.json.gz"
	curl -O http://download.geonames.org/export/dump/cities15000.zip
	curl -O http://download.db-ip.com/free/dbip-city-2015-02.csv.gz

###Decompress Data

	gzip -d https-full-20140424T1025.json.gz
	unzip cities15000.zip 
	gzip -d dbip-city-2015-02.csv.gz 


###INIT POSTGRES DB

We used an Amazon RDS postgres instance which can be connected to with:

    psql -Utrackif -h trackif.cfnx8ncoqghk.us-east-1.rds.amazonaws.com -p5432


###Import raw data tables

For each of 3 data files we downloaded, we imported the data into staging tables. We use varchar as the prefered datatype for all columns. 
We can munge data and convert to more specific types in a later step.


#####First raw_ip_city table

This dataset provides the mapping from ip address to a geographical name. This datafile is a simple csv file we can import with the copy command from psql

	CREATE TABLE raw_ip_city (start_ip VARCHAR, 
		   end_ip VARCHAR, 
		   country_code VARCHAR, 
		   region VARCHAR, 
		   city VARCHAR);
	\COPY raw_ip_city FROM 'dbip-city-2015-02.csv' WITH CSV;

#####Raw Heartbleed Data

This file is very large (88G) and contains a json document per line. We take advantage of postgresql's json support by importing into a table with a single column of type json.
We configure the quote and delimiter to be characters that cannot appear in json. Later on we can extract fields in the json document using Postgresql's ->> and #> json operators.

	CREATE TABLE raw_heartbleed_data (data json);
	\COPY raw_heartbleed_data FROM 'https-full-20140424T1025.json' WITH CSV QUOTE e'\x01' DELIMITER e'\x02';



#####Extracted heartbleed data

Now we can Use Postrgresql's json query feature to extract the relevant information from the json documents generated by the heartbleed scan.
We end up with a table of ip_addresses where presence in the table represents vulnerability to heartbleed

    CREATE TABLE heartbleed_data AS 
		SELECT 
			  data->>'host' AS ip_address, 
			  (data #>> '{tls_handshake, ServerHelloMsg, heartbleed_vulnerable}')::boolean AS vuln, 
			  data AS data
		FROM raw_heartbleed_data 
		WHERE data#>>'{tls_handshake, ServerHelloMsg, heartbleed_vulnerable}' = 'true';

######Raw city population data - cities15000.txt

This dataset provides the population value each city. This is a tab delimited file with many columns, we care about asciiname, admin1_code (state code) and population, but first we will import everything into a staging table.

	CREATE TABLE raw_cities_pop(
							   fgeonameid      INTEGER PRIMARY KEY,
							   name            VARCHAR,
							   asciiname       VARCHAR, 
							   alternatenames  VARCHAR,  
							   latitude        VARCHAR,
							   longitude       VARCHAR,
							   feature_class   VARCHAR,
							   feature_code    VARCHAR,    
							   country_code    VARCHAR,    
							   cc2             VARCHAR,
							   admin1_code     VARCHAR,
							   admin2_code     VARCHAR,   
							   admin3_code     VARCHAR,    
							   admin4_code     VARCHAR,     
							   population      VARCHAR,      
							   elevation       VARCHAR,       
							   dem             VARCHAR,       
							   timezone        VARCHAR,      
							   modification_date VARCHAR);
	\COPY raw_cities_pop FROM 'cities15000.txt' WITH CSV QUOTE e'\b' DELIMITER e'\t';





###Data Cleanup

##### Dedup heartbleed data

It turns out there are duplicate records for some ip addresses, here we create a deduped table to make it easier to only count each vulnerable ip address once

	   CREATE TABLE deduped_heartbleed_data AS 
	   	   (SELECT ip_address, bool_or(vuln), count(*) AS c FROM heartbleed_data GROUP BY ip_address);


##### ip_city data

There are a few cities in the ip_city dataset that do not map to anything in the popluation dataset. For instance ip_city calls New York 'New York' where the
population dataset calls New York 'New York City'. We can use a left join to figure out locations in the ip_city table that will not map to anything in the population table

	SELECT DISTINCT region, city, ip_city.country_code 
		   FROM  ip_city 
		   LEFT JOIN raw_cities_pop ON  ip_city.city = raw_cities_pop.asciiname 
	WHERE  raw_cities_pop.asciiname IS NULL AND ip_city.country_code = 'US'

This returns 436 results. Perusing the list we can see that the only significant mismatch is New York City. Since we are only after the top 10 cities anyway
we can cheat a little bit and correct just that one record. 

	UPDATE corrected_ip_city SET city = 'New York City' where region = 'New York' and city = 'New York';

By issuing the following query we also see there are a few records with an inconsistent region field.

    SELECT DISTINCT region, ip_city.country_code FROM ip_city;

We can fix that with a few simple updates to always use the full state name

	UPDATE corrected_ip_city SET region = 'Montana' where region = 'Mt';
	UPDATE corrected_ip_city SET region = 'New Jersey' where region = 'Nj';
	UPDATE corrected_ip_city SET region = 'Massachusetts' where region = 'Ma';
	UPDATE corrected_ip_city SET region = 'California' where region = 'Ca';
	UPDATE corrected_ip_city SET region = 'California' where region = 'California - L.a. Metro';


##### state_abbrev table

The ip_city table uses a states full name in the region column. The population table uses two letter state codes. 
We can allow joining with a simple mapping table 

    CREATE TABLE state_abbrev (id SERIAL PRIMARY KEY, state_name VARCHAR, state_abbrev VARCHAR(2));

    INSERT INTO state_abbrev (state_name, state_abbrev)
    VALUES ('Alabama', 'AL'),('Alaska', 'AK'),('Arizona', 'AZ'),('Arkansas', 'AR'),('California', 'CA'),('Colorado', 'CO'),('Connecticut', 'CT'),('Delaware', 'DE'),('District of Columbia', 'DC'),('Florida', 'FL'),('Georgia', 'GA'),('Hawaii', 'HI'),('Idaho', 'ID'),('Illinois', 'IL'),('Indiana', 'IN'),('Iowa', 'IA'),('Kansas', 'KS'),('Kentucky', 'KY'),('Louisiana', 'LA'),('Maine', 'ME'),('Maryland', 'MD'),('Massachusetts', 'MA'),('Michigan', 'MI'),('Minnesota', 'MN'),('Mississippi', 'MS'),('Missouri', 'MO'),('Montana', 'MT'),('Nebraska', 'NE'),('Nevada', 'NV'),('New Hampshire', 'NH'),('New Jersey', 'NJ'),('New Mexico', 'NM'),('New York', 'NY'),('North Carolina', 'NC'),('North Dakota', 'ND'),('Ohio', 'OH'),('Oklahoma', 'OK'),('Oregon', 'OR'),('Pennsylvania', 'PA'),('Rhode Island', 'RI'),('South Carolina', 'SC'),('South Dakota', 'SD'),('Tennessee', 'TN'),('Texas', 'TX'),('Utah', 'UT'),('Vermont', 'VT'),('Virginia', 'VA'),('Washington', 'WA'),('West Virginia', 'WV'),('Wisconsin', 'WI'),('Wyoming', 'WY');



###Final query

	SELECT (COUNT(*)::float / population::float) as vuln_per_capita, COUNT(*) as vuln_ip_count, 
		region,
		city,
		population 
	FROM deduped_heartbleed_data AS hb 
	 	JOIN corrected_ip_city ON hb.ip_address BETWEEN start_ip AND end_ip  
		JOIN state_abbrev ON corrected_ip_city.region = state_abbrev.state_name
		JOIN raw_cities_pop ON raw_cities_pop.asciiname = corrected_ip_city.city 
		    AND state_abbrev.state_abbrev = raw_cities_pop.admin1_code
	GROUP BY region, city, population order by population::integer DESC
	LIMIT 10;


vuln_per_capita       | vuln_ip_count |    region    |     city      | population 
----------------------|---------------|--------------|---------------|------------
   0.0049523353320368 |         40486 | New York     | New York City | 8175133
  0.00189736860076448 |          7196 | California   | Los Angeles   | 3792621
  0.00120418549056647 |          3246 | Illinois     | Chicago       | 2695598
 5.86787118849167e-05 |           135 | New York     | Brooklyn      | 2300664
 0.000832598617448085 |          1748 | Texas        | Houston       | 2099451
 0.000742461038816361 |          1133 | Pennsylvania | Philadelphia  | 1526006
 0.000582444218168939 |           842 | Arizona      | Phoenix       | 1445632
  0.00169503400238209 |          2250 | Texas        | San Antonio   | 1327407
   0.0005247047197419 |           686 | California   | San Diego     | 1307402
 0.000909154661483901 |          1089 | Texas        | Dallas        | 1197816






###Further Steps

These process presented here represents a good approach for a one off quick analysis, however many improvements 
can be made to make the process repeatable, automated and more efficient. One good tool for this is Drake https://github.com/Factual/drake.
Drake is essential make for data. You establish a series of steps and their dependencies. If you need to modify 
something at one step you can then easily recompute the rest of the process. 

With this approach you can use tools like 
* jq http://stedolan.github.io/jq/ 
* csvkit https://github.com/onyxfish/csvkit


to extract only the data you need before importing it into the database.

Another approach would be using some big data tools to speed up the process. I have found spark http://spark.apache.org/ to be a big help here. 
You get the benefit of hadoop style mapreduce, but with a much simplier api that can string together multiple complex steps. Amazon EMR has good
support for spark now so if you can get your data into s3 its simple to throw an arbitrary sized cluster at your data.


