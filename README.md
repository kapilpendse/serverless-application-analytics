# Serverless Application Analytics Pipeline on AWS

This is a guide to deploying a fully serverless data analytics pipeline on AWS. AWS is a very broad collection of services, and it can be a daunting task to just identify and configure the right set of services when you are just getting started. This guide keeps it simple and gets you from zero knowledge of AWS to visualization of your data in a few steps. You can bring your own data, but if you don't have it handy, this guide also contains instructions on how to generate some fake sample data as well.

To keep things simple, this guide uses only 5 AWS services:
- Amazon Kinesis Data Firehose for data ingestion
- Amazon S3 for data storage
- Amazon Glue to generate a schema
- Amazon Athena to query the data
- Amazon QuickSight to visualize the data

**Q. Wow, that's a lot of unfamiliar names. Doesn't it already feel daunting? How come this is "keeping it simple" you ask?**
A. Well, you will only interact with 2 of these i.e. QuickSight to create dashboards and Athena to write custom SQL queries.


## Architecture Diagram
![architecture diagram](/arch/diagram.png)


## Deploy Serverless Data Pipeline
1. Deploy the pipeline using the button below.

Region| Launch
------|-----
Asia Pacific (Singapore) | [![cfn](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/images/cloudformation-launch-stack-button.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-1#/stacks/new?stackName=serverless-app-analytics&templateURL=https://kapilpen.s3-ap-southeast-1.amazonaws.com/labs/serverless-app-analytics-pipeline/serverless-app-analytics-pipeline.yaml)

2. Scroll to the bottom and click **Next**.
	* *databaseName*: **serverless-app-analytics-db**
	* *s3BucketName*: **serverless-app-analytics-YOUR NAME OR INITIALS**
	* Scroll to the bottom and click **Next**, then **Next**
	* Acknowledge the checkbox and click **Create stack**


This deploys everything. The rest is just testing and getting familiar.


## Generate Test Data
In order to test this analytics pipeline, we need to have some data. Let's use a fake data generator.

3. Go to [Kinesis Data Generator (KDS)](https://awslabs.github.io/amazon-kinesis-data-generator/web/help.html) and follow the setup instructions.
	* During this setup, you will be launching a CloudFormation stack in the AWS Oregon region. This stack creates a user identity with the necessary permissions for the Kinesis Data Generator to send data to your analytics pipeline.
	* During setup, you will be asked to specify a username and password for this identity. Specify values that you will remember. Then launch the stack and wait for it to complete.
	* After the stack status becomes *CREATE_COMPLETE*, click on the **Outputs** tab. You will see a *KinesisDataGeneratorUrl* link. Click on it, and log in using the username and password that you specified in the previous step.
4. After logging in, select the region in which you deployed the serverless-app-analytics-pipeline stack. Then select the Kinesis Data Firehose which starts with the prefix 'serverless-app-analytics-deliveryStream-'.
5. Use the following template. It generates fake application clickstream data in JSON format.
	```
	{
	    "customerId": "{{random.number(500)}}",
	    "ipAddress": "{{internet.ip}}",
	    "countryCode": "{{random.arrayElement(["US","UK","SG","ID","CN","AUS","TH","MY","PH","VN","IN"])}}",
	    "gender": "{{random.arrayElement(["M","F"])}}",
	    "trafficOrigin": "{{random.arrayElement(["webAd","mobileAd","search","direct","email"])}}",
	    "event": {
	        "type": "{{random.arrayElement(["search","viewDetail","addToCart","checkout","addToWishlist","removeFromCart"])}}",
	        "product": "{{random.arrayElement(["","shoes","bags","cosmetics","apparel"])}}",
	        "amount": {{random.number({"min":50,"max":300})}}
	    }
	}
	```
6. Click 'Test template'. Ensure that you can see what looks like valid JSON objects.
7. If all looks good, click 'Send data'. KDS will start sending data to your Kinesis Firehose. **Leave this running**. Wait for a minute or so, and then open Amazon Kinesis in your AWS console.
8. Click on 'Data Firehose' from left hand menu, then click on the firehose delivery stream which has prefix 'serverless-app-analytics-deliveryStream-'. Scroll down, and under the 'Amazon S3 destination' section, click on the name of the S3 bucket. Once the Amazon S3 console opens, click on the folder 'raw', then click 'firehose'. You should see a folder tree that is organized as year -> month -> date -> hour (utc). Inside this, you should see raw data files. Download one of them and open in a text editor to confirm that the raw files contain JSON data similar to below:
	```
	{    "customerId": "26",    "ipAddress": "167.49.68.77",    "countryCode": "VN",    "gender": "M",    "trafficOrigin": "mobileAd",    "event": {        "type": "search",        "product": "apparel",        "amount": 127    }}
	{    "customerId": "328",    "ipAddress": "13.109.202.188",    "countryCode": "SG",    "gender": "M",    "trafficOrigin": "webAd",    "event": {        "type": "checkout",        "product": "bags",        "amount": 256    }}
	{    "customerId": "169",    "ipAddress": "37.166.245.52",    "countryCode": "UK",    "gender": "F",    "trafficOrigin": "search",    "event": {        "type": "addToWishlist",        "product": "apparel",        "amount": 125    }}
	```


## Verify Glue Crawler Result
The serverless application analytics pipeline deployed in step (1) contains an AWS Glue crawler, which is configured to crawl the raw data every 10 minutes. The crawler attempts to make sense of the raw data and creates a meta data table in the Glue data catalog.

9. After about 10 minutes of starting the KDS data generation, open AWS Glue in your AWS console.
10. In the left hand menu, click 'Crawlers'. Find the crawler named 'serverless-app-analytics-crawler', and observe the schedule column. It should say something like 'Every 10 minutes'.
11. In the left hand menu, click 'Databases', click 'serverless-app-analytics-database', then click the link 'Tables in serverless-app-analytics-database'. You should see a table named 'firehose', click on it.
12. Scroll down to the 'Schema' section. An automatically deduced schema will be displayed. Verify that the column names and data types have been correctly deduced by Glue crawler.
	- AWS Glue crawler usually deduces the data schema automatically for most common data formats like JSON, CSV, TSV etc.
	- If your application data is not automatically deduced by AWS Glue crawler, you can create a "Classifier" of your own. Learn more [here](https://docs.aws.amazon.com/glue/latest/dg/add-classifier.html).


## Interactively Query Your Data
Now that your data is in Amazon S3, let's run some SQL queries on it to explore the data.

13. Go to Athena in your AWS console.
14. In the left hand menu, under **Database**, click the dropdown and select *serverless-app-analytics-database*
15. Under **Tables**, click on *firehose (Partitioned)*. The table should expand to reveal all the column names.
16. On the right hand side, click **+** to create a new query. Click into the query editor and type the following:
	```
	select count(*) as total from firehose;
	```
17. Click **Run query**. Within a few seconds, you should see the query result at the bottom. This query counts the number of data points (rows) in your data set.

18. In the query editor, click **+** to create a new query. Click into the query editor and type the following:
	```
	SELECT count(customerid) as Customers,
         gender as Gender
	FROM firehose
	WHERE countrycode = 'US'
	GROUP BY  Gender
	```
19. Click **Run query**. Within a few seconds, you should see the result which shows the number of female and male customers in the US within the generated data set.
20. On the right hand side, click **+** to create a new query. Click into the query editor and type the following:
	```
	select count(customerid) as purchases, event.product as product from firehose where event.type = 'checkout' group by event.product
	```
21. Click **Run query**. Within a few seconds, you should see the result which shows the number of purchases for each product. This is also an example of querying on nested JSON objects.


## Visualize Your Data
Now that we have explored the data little bit, let's learn how to visualize it using Amazon QuickSight. Amazon QuickSight

22. Go to QuickSight in your AWS console. Sign up for the service if you haven't done so already.
23. After you are inside the QuickSight console, in the top-right corner switch to the region in which you have deployed this serverless analytics pipeline.
24. Connect QuickSight to your dataset using Amazon Athena by following [these instructions](https://docs.aws.amazon.com/quicksight/latest/user/create-a-data-set-athena.html)
	- name your dataset 'serverless-app-analytics'
	- select 'serverless-app-analytics-database' from the database dropdown
	- select 'firehose' from the list of tables
	- select 'Directly query your data'. Click here to learn more about [QuickSight SPICE](https://docs.aws.amazon.com/quicksight/latest/user/welcome.html#spice).
25. After you have connected to Athena as a data source, you will be able to create visualizations and publish them as dashboards.
26. You can add parameterized date filter to your visualization in order to restrict your visualization to data from a specific date range. For this, you first need to [set up start date and end date parameters](https://docs.aws.amazon.com/quicksight/latest/user/parameterize-a-filter.html) and then [add a date filter](https://docs.aws.amazon.com/quicksight/latest/user/add-a-date-filter.html) of the type 'time range''between'.


## Connect Your Application
You can send your application data such as clickstream, log files, live feed etc. straight to your Amazon Kinesis Data Firehose delivery stream. There are multiple ways to do this, but here's a couple that are popular:

- [Fluent Plugin for Amazon Kinesis](https://github.com/awslabs/aws-fluent-plugin-kinesis)
- [Kinesis Agent](https://docs.aws.amazon.com/firehose/latest/dev/writing-with-agents.html)

Once your application data starts streaming into Amazon Kinesis Data Firehose, within a few minutes you will be able to start querying and visualizing it.

You can start querying and visualizing your data only after the Glue crawler creates a table for it in the Glue data catalog. Remember that we've set up the Glue crawler to run once every 10 minutes. If you don't want to wait for its next run, you can run it manually via the AWS Glue console. 
