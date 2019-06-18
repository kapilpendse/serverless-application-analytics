# Serverless Application Analytics Pipeline on AWS

## Deploy Serverless Data Pipeline
1. Deploy [this template](/cfn/serverless-app-analytics-pipeline.yaml) to your account using AWS CloudFormation.

## Generate Test Data
In order to test this analytics pipeline, we need to have some data. Let's use a fake data generator.

2. Go to [Kinesis Data Generator](https://awslabs.github.io/amazon-kinesis-data-generator/)(KDS) and follow the setup instructions. Deploy the CloudFormation stack, then log in.
3. After logging in, select the region in which you deployed the serverless-app-analytics-pipeline stack. Then select the Kinesis Data Firehose which starts with the prefix 'app-analytics-deliveryStream-'.
4. Use the following template. It generates fake application clickstream data in JSON format.
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
5. Click 'Test template'. Ensure that you can see what looks like valid JSON objects.
6. If all looks good, click 'Send data'. KDS will start sending data to your Kinesis Firehose. Wait for a minute or so, and then open Amazon Kinesis in your AWS console.
7. Click on 'Data Firehose' from left hand menu, then click on the firehose delivery stream which has prefix 'app-analytics-deliveryStream-'. Scroll down, and under the 'Amazon S3 destination' section, click on the name of the S3 bucket. Once the Amazon S3 console opens, click on the folder 'raw', then click 'firehose'. You should see a folder tree that is organized as year -> month -> date -> hour (utc). Inside this, you should see raw data files. Download one of them and open in a text editor to confirm that the raw files contain JSON data similar to below:
	```
	{    "customerId": "26",    "ipAddress": "167.49.68.77",    "countryCode": "VN",    "gender": "M",    "trafficOrigin": "mobileAd",    "event": {        "type": "search",        "product": "apparel",        "amount": 127    }}
	{    "customerId": "328",    "ipAddress": "13.109.202.188",    "countryCode": "SG",    "gender": "M",    "trafficOrigin": "webAd",    "event": {        "type": "checkout",        "product": "bags",        "amount": 256    }}
	{    "customerId": "169",    "ipAddress": "37.166.245.52",    "countryCode": "UK",    "gender": "F",    "trafficOrigin": "search",    "event": {        "type": "addToWishlist",        "product": "apparel",        "amount": 125    }}
	```

## Verify Glue Crawler Result
The serverless application analytics pipeline deployed in step (1) contains an AWS Glue crawler, which is configured to crawl the raw data every 10 minutes. The crawler attempts to make sense of the raw data and creates a meta data table in the Glue data catalog.

8. After about 10 minutes of starting the KDS data generation, open AWS Glue in your AWS console.
9. In the left hand menu, click 'Crawlers'. Find the crawler named 'app-analytics-crawler', and observe the schedule column. It should say something like 'Every 10 minutes'.
10. In the left hand menu, click 'Databases', click 'app-analytics-database', then click the link 'Tables in app-analytics-database'. You should see a table named 'firehose', click on it.
11. Scroll down to the 'Schema' section. An automatically deduced schema will be displayed. Verify that the column names and data types have been correctly deduced by Glue crawler.
	- AWS Glue crawler usually deduces the data schema automatically for most common data formats like JSON, CSV, TSV etc.
	- If your application data is not automatically deduced by AWS Glue crawler, you can create a "Classifier" of your own. Learn more [here](https://docs.aws.amazon.com/glue/latest/dg/add-classifier.html).

## Interactively Query Your Data
Now that your data is in Amazon S3, let's run some SQL queries on it to explore the data.

13. Go to Athena in your AWS console.
14. In the left hand menu, under **Database**, click the dropdown and select *app-analytics-database*
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
	- select 'app-analytics-database' from the database dropdown
	- select 'firehose' from the list of tables
	- select 'Directly query your data'
25. After you have connected to Athena as a data source, you will be able to create visualizations and publish them as dashboards.
26. You can add parameterized date filter to your visualization in order to restrict your visualization to data from a specific date range. For this, you first need to [set up start date and end date parameters](https://docs.aws.amazon.com/quicksight/latest/user/parameterize-a-filter.html) and then [add a date filter](https://docs.aws.amazon.com/quicksight/latest/user/add-a-date-filter.html) of the type 'time range''between',

## Connect Your Application
You can send your application data such as clickstream, log files, live feed etc. straight to your Amazon Kinesis Data Firehose delivery stream. There are multiple ways to do this, but here's a couple that are popular:
	a. [Fluent Plugin for Amazon Kinesis](https://github.com/awslabs/aws-fluent-plugin-kinesis)
	b. [Kinesis Agent](https://docs.aws.amazon.com/firehose/latest/dev/writing-with-agents.html)

Once your application data starts streaming into Amazon Kinesis Data Firehose, within a few minutes you will be able to start querying and visualizing it.

You can start querying and visualizing your data only after the Glue crawler creates a table for it in the Glue data catalog. Remember that we've set up the Glue crawler to run once every 10 minutes. If you don't want to wait for its next run, you can run it manually via the AWS Glue console. 