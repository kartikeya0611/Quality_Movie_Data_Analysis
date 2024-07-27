# Quality_Movie_Data_Analysis
Process and Ingest only the quality movie data in Redshift Data Warehouse

# Tech Stack
**Extract:**
1. S3
2. Glue Crawler
3. Glue Catalog

**Transform:**

4. Glue Catalog Table Data Quality
5. Glue Low Code ETL

**Load:**

6. Redshift

**Alerting:**

7. Event Bridge
8. SNS


# S3
Directory’s structure: 

![image](https://github.com/user-attachments/assets/a1df0291-ae00-4bb6-918c-6e541ebaf062)


Create a crawler for the S3 input file:

![image](https://github.com/user-attachments/assets/75049835-d141-4ea1-a475-061c72d17d6e)

Run the above crawler and check the catalog table created for the S3 input file - movies_table_1input:
![image](https://github.com/user-attachments/assets/13887410-bd9e-44d9-8115-57592a971558)
![image](https://github.com/user-attachments/assets/c5a111ee-b137-40f3-aa9a-19e6fc9518e7)


# Glue Catalog Table Data Quality
Create a ruleset for Historical Data Quality checks on files present in the input path:
![image](https://github.com/user-attachments/assets/6d783001-aa07-4d10-a15e-748affd35f75)


Run the DQ check and save the output of the data quality results to historical_data_rule_outcome directory:
![image](https://github.com/user-attachments/assets/549f9b74-62c6-46d2-aeca-c83a04fa975c)


DQ results will be either Passed or Fail:
![image](https://github.com/user-attachments/assets/c74b6058-1508-4026-9460-8eb28a1afdf6)

It won’t give results per line -> For per line results, use similar functionality available in Evaluate Data Quality transformation in AWS Glue job

# Redshift
Now let’s create the destination Table in Redshift: 

CREATE SCHEMA movies;

CREATE TABLE movies.imdb_movies_rating (
    Poster_Link VARCHAR(MAX),
    Series_Title VARCHAR(MAX),
    Released_Year VARCHAR(10),
    Certificate VARCHAR(50),
    Runtime VARCHAR(50),
    Genre VARCHAR(200),
    IMDB_Rating DECIMAL(10,2),
    Overview VARCHAR(MAX),
    Meta_score INT,
    Director VARCHAR(200),
    Star1 VARCHAR(200),
    Star2 VARCHAR(200),
    Star3 VARCHAR(200),
    Star4 VARCHAR(200),
    No_of_Votes INT,
    Gross VARCHAR(20)
);


Create JDBC connection for redshift cluster:
![image](https://github.com/user-attachments/assets/3a4ae55e-21f3-41c2-baa4-e28b8c42492e)

Now create a crawler using this JDBC connection to create a Catalog table inside our project_2_movies_db
![image](https://github.com/user-attachments/assets/fe0ebd03-e61a-42a6-8b5a-f7e6a3e4c6fa)

Run the above crawler and check the catalog table created : project_2dev_movies_imdb_movies_rating:
![image](https://github.com/user-attachments/assets/fa6dbbbe-4b04-465d-8d8d-fee98e9adff1)
![image](https://github.com/user-attachments/assets/168c6f13-8088-4db0-8570-6a716c1d7afd)


# Glue Low Code ETL

![image](https://github.com/user-attachments/assets/996ed4b2-3e66-4516-bc1b-a6259629444b)

**S3 file as source:**

![image](https://github.com/user-attachments/assets/d42905a1-e1b2-4375-bc1f-0d299c0e5e04)

**Transform - Evaluate Data Quality:**

![image](https://github.com/user-attachments/assets/626b1455-db98-4246-8a5e-cefbecceb729)

We’ll continue the job even if rules are failing for any record and publish the results in CloudWatch to let EventBridge match the failure pattern and later send SNS notification accordingly:

![image](https://github.com/user-attachments/assets/81c8c2af-6cc1-4be0-be1d-e7da28c3c59f)

We need to create VPC endpoints for Glue and Cloudwatch (to keep cloudwatch and Glue both in the same VPC so that the data transfer will happen)

Data quality outputs:

![image](https://github.com/user-attachments/assets/abbfbd3f-3d95-4497-a7bd-3e1b04e61f78)

[Original Data + below 4 DQ columns] will go in rowLevelOutcomes node:

![image](https://github.com/user-attachments/assets/dbe65f0f-f3d3-41a8-9fd1-1b391a055a7d)

Data quality results will go in ruleOutcomes node:

![image](https://github.com/user-attachments/assets/cea53dd2-2993-47ee-a3fb-250b0a85d2a4)

This is how the visual ETL would look:

![image](https://github.com/user-attachments/assets/48028f41-7558-4c1c-90d4-e5cabbdd904b)

Store this ruleOutcomes result in rule_outcome directory in S3:

![image](https://github.com/user-attachments/assets/be567ee9-8c26-4c21-a8aa-6949525aa78a)

**Conditional Router:**
Now the row level outcomes can be further divided as Failed_Records and Passed_Records(i.e., default_group)

![image](https://github.com/user-attachments/assets/8656723c-78b7-42b2-b5dc-010ef7ee777b)

Conditions for Failed_Records in the conditional router:

![image](https://github.com/user-attachments/assets/85cc04f5-a165-40ef-99d4-5eb489e0c967)

Keep these Failed_Records in bad_records directory:

![image](https://github.com/user-attachments/assets/e11bc215-100c-4dbb-9664-4047adff8469)

Now from the Passed_Records (i.e., default_group), drop the Drop the DataQuality columns.
For other columns keep the Data type as expected in the destination redshift table(imdb_movies_rating).
![image](https://github.com/user-attachments/assets/cc1fee11-0fc1-4900-920e-13ba842792ae)

**Redshift table load**

![image](https://github.com/user-attachments/assets/f0836a7c-7081-4820-82f8-282f4cd3ccfe)
![image](https://github.com/user-attachments/assets/c865574c-d021-4fc1-b38b-878cf57fb55b)

Note - S3 VPC endpoint is needed to be created to use this S3 directory for Glue

Now run the Glue job and check the destination table contents in redshift:

![image](https://github.com/user-attachments/assets/17a69191-cb65-4b27-bf35-9165a83c9bf8)

# Eventbridge Rule

Event Pattern:

![image](https://github.com/user-attachments/assets/dc3f97e3-8704-431c-a2d8-08dd14794919)

EventBridge Rule will look like:

![image](https://github.com/user-attachments/assets/de1290cb-03ac-41c0-b7d4-9219188414ba)
![image](https://github.com/user-attachments/assets/311a26bb-6f66-4a26-ad46-4abd861cc41d)
