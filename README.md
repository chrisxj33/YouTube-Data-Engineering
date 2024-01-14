# YouTube-Data-Engineering

## Overview

![Blank diagram](https://github.com/chrisxj33/YouTube-Data-Engineering/assets/53899548/0bb4fd93-a832-4e06-9573-7e1d7d56782f)

#### Key Components and Their Roles

1. **Data Source and Ingestion:**
   - The raw data is sourced from Kaggle and ingested into AWS using an EC2 instance. This instance runs a Python script to fetch and store data in the AWS S3 'youtube-raw-data-s3' bucket. This step ensures that the latest data from YouTube is consistently available for analysis.

2. **Data Storage and Management:**
   - The architecture utilizes two S3 buckets: one for raw data and another for cleaned data. The cleaned data S3 bucket will act as our data lakehouse. Strict access controls and encryption are employed to ensure data security and integrity.

3. **Data Transformation:**
   - AWS Lambda is used to transform the raw data. The transformation includes normalizing nested JSON structures into a tabular format, making it suitable for querying and analysis. The cleaned data is then stored in the 'youtube-clean-data-s3' bucket.

4. **Data Cataloging and Scheduling:**
   - AWS Glue Crawlers are configured to catalog the cleaned data, facilitating the ability for querying. Amazon EventBridge is used to trigger the Glue Crawlers automatically whenever new data is uploaded to the S3 bucket, ensuring the data catalog is always up-to-date.

5. **Data Querying and Analysis:**
   - AWS Athena is employed to query the semi-structured data. Example queries include determining the number of unique video categories trending and finding the average number of videos per channel within each category. These queries are crucial for extracting meaningful insights from the dataset.

#### Overall Aim

The architecture aims to create a robust, scalable, and secure pipeline for data ingestion, storage, transformation, and analysis. By leveraging AWS services, it ensures efficient handling of large datasets, enabling deep insights into YouTube video trends and engagement patterns across different regions and categories. The setup is designed to be flexible and adaptive to changes in data structures or requirements, making it a powerful tool for data-driven decision making and trend analysis in the realm of digital content.

## Data Source

Our primary dataset, "*Trending YouTube Video Statistics*", is sourced from Kaggle. This dataset offers a comprehensive daily snapshot of the most popular YouTube videos across different regions, including the USA, Germany, and others. Each region's data is compiled into individual files. 

Key data points encompass a range of metrics such as…

- `video_title`
- `channel_title`
- `publish_time`

And various engagement indicators like…

- `tags`
- `views`
- `likes`

Moreover, the datasetcontains a `category_id` field, which is specific to each region. To gain insights into the video category, you can reference the corresponding `JSON` file that maps `category_id` to its respective category name. This dataset allows for an in-depth understanding of the content trends across various YouTube categories in different geographical locations.

**Data Source:** https://www.kaggle.com/datasets/datasnaek/youtube-new

## Creating an IAM User

When working on projects that don't require access to high-level features like cost and billing, it's a best practice to avoid using a root account (for security reasons). Instead, within AWS's IAM (Identity and Access Management) section, we can create a new IAM user. For example, we'll name this user “*cj-data-engineer*” to reflect both the initials of the person and their role.

During the creation process, we select the option “Provide user access to the AWS Management Console”. The console is essentially the web-based interface for AWS, allowing users to manage various services.

Next, we assign the “*AdministratorAccess*” policy to this new user. This policy grants full admin access, which is extensive, though not as broad as the permissions of a root user. This setup ensures the user has sufficient permissions for most tasks while maintaining security best practices. If more strict restrictions are required, the policy can be adjusted. Throughout this project, all tasks will be carried out within the new user account.

![image](https://github.com/chrisxj33/YouTube-Data-Engineering/assets/53899548/54cb04ab-3e98-4a0a-bba6-aa989752d562)

## Creating the S3 Buckets

For this data architecture we will be using two S3 buckets - one for storing the raw data and the other for storing the cleaned data. When creating the S3 buckets, we apply the following settings...

1. **Block Public Access**: It is ensured that the 'Block Public Access' setting is enabled. This is crucial for preventing unauthorised access to the data stored in the bucket.
2. **Bucket Versioning**: We keep the bucket versioning disabled. This helps in reducing the storage size and, consequently, the cost. It's important when you want to manage storage efficiently and do not require keeping multiple versions of each object.
3. **Enable Encryption**: We choose for server-side encryption for added security. This ensures that the data is encrypted while it's stored in the bucket.

The resulting buckets are shown below…

![image](https://github.com/chrisxj33/YouTube-Data-Engineering/assets/53899548/765a0f60-a967-4d7c-b57f-391a2d3276ba)

## ****Configuring an EC2 Instance for Data Ingestion****

To effectively run our data ingestion script, we will deploy an Amazon EC2 (Elastic Compute Cloud) instance. We have chosen the *t3.nano* instance type for this purpose, balancing sufficient computational capability with cost efficiency. This instance will be tasked with executing a Python script designed to ingest data into our raw data S3 bucket.

*Note: An alternative approach, such as using AWS Lambda, was considered. However, due to dependency challenges encountered with the Kaggle API integration, we opted for the EC2 instance.*

**EC2 Instance Role and Policy Creation**: We will start by creating a policy named *youtube-s3-write*. This policy will specifically grant write permissions to our designated S3 bucket, *youtube-raw-data-s3*. The IAM policy in `JSON` format is shown below…

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::youtube-raw-data-s3/*"
        }
    ]
}
```

We will then create an IAM role, *ec2-yt-raw-data-s3-write*, and attach the above policy to this role. This role will be associated with our EC2 instance, thereby granting it the specified S3 write permissions. With the IAM role in place, our script can leverage AWS SDK for Python (boto3) for authenticated interactions with AWS services, without needing to hardcode credentials.

**Finalizing the EC2 Setup for Data Ingestion:** The EC2 instance will be equipped with Python and all necessary libraries to ensure readiness for running our data ingestion script. Initially, we will manually execute the script on the EC2 instance. This action will populate the *youtube-raw-data-s3*  bucket with data from Kaggle. In cases where the data source is regularly updated (like every 24 hours) we can schedule the script to run automatically. This ensures that our S3 is consistently updated with the latest data.

![image](https://github.com/chrisxj33/YouTube-Data-Engineering/assets/53899548/6fde788c-8994-4cf9-8f4e-0a46391473f7)

## Transforming the Raw Data

As previously noted, we have two types of data - the `csv` files which are in a tabular format and the `json` files which are in a nested structure.

```json
{
    "kind": "youtube#videoCategoryListResponse",
    "etag": "\"ld9biNPKjAjgjV7EZ4EKeEGrhao/1v2mrzYSYG6onNLt2qTj13hkQZk\"",
    "items": [
        {
            "kind": "youtube#videoCategory",
            "etag": "\"ld9biNPKjAjgjV7EZ4EKeEGrhao/Xy1mB4_yLrHy_BmKmPBggty2mZQ\"",
            "id": "1",
            "snippet": {
                "channelId": "UCBR8-60-B28hp2BmDPdntcQ",
                "title": "Film & Animation",
                "assignable": true
            }
        }
    ]
}
```

Within the `json` data we want to extract “*******items*******” then transform the nested structure by normalising it into a tabular format. The cleaned format should look like the following…

| kind | etag | id | snippet.channelId | snippet.title | snippet.assignable |
| --- | --- | --- | --- | --- | --- |
| youtube#videoCategory | etag_value_1 | 1 | channel_id_1 | Film & Animation | True |
| youtube#videoCategory | etag_value_2 | 2 | channel_id_2 | Autos & Vehicles | False |
| ... | ... | ... | ... | ... | ... |

To acheive this transformation we will be using AWS lambda. We name our AWS Lambda function *transformYouTubeData* and use Python 3.8 as the run time. Again we will need to create a new role - this role must allow sufficient permissions for the script to operate correctly, namely allow writting to an S3 bucket. Once set up, we copy over the `transform_data.py` script.

******************************************Additional Tweaking:****************************************** We increase the memory size to 512 MB and the timeout limit to 5 minutes to avoid any errors. Our script has a funtional dependency (the `awswrangler` **library) which will need to be inlcuded within a layer. AWS already this library within a layer, therefor we simply select it and assign it to the function. Latly, we add some environment variables.

As for the Python script, lets discuss the event…

1. **File is Uploaded to S3**: Someone uploads a file to your specified S3 bucket.
2. **S3 Generates an Event**: This upload triggers S3 to create an event. This event is like a message that says, "Hey, a new file is here, and here are the details about it."
3. **Event is Sent to Lambda**: This event is then automatically sent to your Lambda function.
4. **Lambda Processes the Event**: Your Lambda function receives this event. It reads the event details to find out which file was uploaded and where it is stored.
5. **Lambda Executes Your Code**: Using this information, your Lambda function then executes its code. It reads the file, processes it (like cleaning or transforming the data), and then stores the result back in S3.

*******Note:******* `awswrangler` *provides a convenient method to write data to S3 and simultaneously create a data catalog in AWS Glue. An example snippet is shown below:*

```python
# write to s3 - this will also require permissions for glue services
wr_response = wr.s3.to_parquet(
    df=your_data,
    path=your_save_path,
    dataset=True,
    database=name_of_your_glue_database,
    table=name_of_your_glue_table,
    mode=os_input_write_data_operation
)
```

***********************************************************************************************************In this case we will be using the Glue Crawler service instead to schedual a crawler and generate a data catalog, which will be triggered once data is uploaded to the S3 bucket. Ultimately you can choose either method - however the crawler method means there is loser coupling and more flexibility to adapt to any alternative sources that may add new data to the bucket that is not through Lambda.***********************************************************************************************************

To verify that our Lambda function works as intended, wecan set up a test using a simulated S3 event. AWS offers a convenient way to create this test by providing a pre-configured template for an S3 'put' event (which is what our script needs). This type of event typically occurs when a file is uploaded to an S3 bucket. We simply input our S3 bucket name and the file name - this will be used to provide the function with a fake (test) event.

*youtube-raw-data-s3* is our test bucket and we can choose any file name for example `US_category_id.json`.

Once we have confirmed the script works accordingly, we set up a trigger to create the same JSON event when a new file is uploaded to our *youtube-raw-data-s3* bucket. This automates the process of cleaning the data once the raw data has been ingested.

## Glue Crawler

*AWS Glue* is a fully managed extract, transform, and load (ETL) service that makes it simple and to categorise your data, clean it, enrich it, and move it reliably between various data stores. Within AWS Glue, several components work together, including *Glue Crawlers*, *Glue Databases*, and *Glue Tables*. Our project will be using these Glue sub-services - which fall under the role of “exploring and categorising our data”.

In summary, the *Glue Crawler* is responsible for discovering and categorising data, creating schemas, and populating the *Glue Data Catalog* with *tables*. These ******tables****** are stored in *********databases********* for better management.

We will be setting up a *Glue Crawler* that automatically runs every time a new S3 file is uploaded to the *youtube-clean-data-s3* bucket.

Once the crawler is complete we will be schedualing it using **************Amazon EventBridge**************. This is a similar process to the Lambda event previously discussed. Within the S3 bucket we create a new event notification where “All object create events”. We apply the appropraite settings. Now when the Lambda function uploads cleaned data to the S3 bucket - our crawler is automatically update the data catalog for enhanced querrying.

## Query the S3 Data with Athena

Now we can query our semi-structured data with Athena. Some example querries are shown below.

*Get the number of unique video categories that were trending…*

```sql
SELECT "snippet.title" as Category, COUNT(*) as VideoCount
FROM "AwsDataCatalog"."cleaned-yt-db"."youtube_clean_data_s3"
GROUP BY "snippet.title"
ORDER BY VideoCount DESC;
```

![image](https://github.com/chrisxj33/YouTube-Data-Engineering/assets/53899548/ccc36a64-ed34-4f3b-8fbf-edaf9c98cd32)

*Find the average number of videos per channel within each category…*

```sql
SELECT "snippet.title" as Category, AVG(VideoCount) as AverageVideos
FROM (
  SELECT "snippet.channelid", "snippet.title", COUNT(*) as VideoCount
  FROM "AwsDataCatalog"."cleaned-yt-db"."youtube_clean_data_s3"
  GROUP BY "snippet.channelid", "snippet.title"
) as ChannelVideos
GROUP BY "snippet.title";
```

![image](https://github.com/chrisxj33/YouTube-Data-Engineering/assets/53899548/5fb7989e-fd77-44a9-8ebc-a4b20e6de6ea)

