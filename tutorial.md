# Tutorial for Scanning User-generated Content Using the Cloud Video Intelligence and Cloud Vision APIs


## Quick Note

### Reference

gloud beta functions deploy: <https://cloud.google.com/sdk/gcloud/reference/beta/functions/deploy>



### Objective

*   Deploy cloud functions.
*   Create or support Cloud Storage Buckets, Cloud Pub/Sub topics, and Cloud Storage Pub/Sub Notifications.
*   Create or support BigQuery dataset and table.



### Architecture

![](./static/video-image-api-arch.png)



## Google Cloud Shell



Export necessary information as the environment variables.

```sh
export PROJECT_ID=$(gcloud info --format='value(config.project)')
export IV_BUCKET_NAME=${PROJECT_ID}-upload
export FILTERED_BUCKET_NAME=${PROJECT_ID}-filtered
export FLAGGED_BUCKET_NAME=${PROJECT_ID}-flagged
export STAGING_BUCKET_NAME=${PROJECT_ID}-staging
```



## Creating Cloud Storage buckets

Create a bucket for storing uploaded image and video files.

```sh
gsutil mb gs://${IV_BUCKET_NAME}
```

Create a bucket for storing the filtered image and video files.

```sh
gsutil mb gs://${FILTERED_BUCKET_NAME}
```

Create a bucket for storing your flagged image and video files.

```sh
gsutil mb gs://${FLAGGED_BUCKET_NAME}
```

Create a bucket for cloud functions to use as a staging location.

```sh
gsutil mb gs://${STAGING_BUCKET_NAME}
```

Check there are four storage buckets that have been created.

```sh
gsutil ls
```



## Creating Cloud Pub/Sub topics

Cloud Pub/Sub topics is used for cloud storage notification messages and for messages between cloud functions.

```sh
export UPLOAD_NOTIFICATION_TOPIC=upload_notification
gcloud pubsub topics create ${UPLOAD_NOTIFICATION_TOPIC}
```

Create a topic to receive your messages from the Vision API. The default value in the `config.json` is `visionapiservice`.

```sh
gcloud pubsub topics create visionapiservice
```

Create a topic to receive your messages from the Video Intelligence API. The default value in the `config.json` is `videointelligenceservice`.

```sh
gcloud pubsub topics create videointelligenceservice
```

Create a topic to receive your messages to store in BigQuery.. The default value in the `config.json` is `bqinsert`.

```sh
gcloud pubsub topics create bqinsert
```

Check there are four topics that have been created.

```sh
gcloud pubsub topics list
```



## Creating Cloud Storage notifications



Create a notification that is triggered only when one of your new objects is placed in the Cloud Storage file upload bucket.

```sh
# -t : types
# -f : format
# -e : event types
gsutil notification create -t upload_notification -f json -e OBJECT_FINALIZE gs://${IV_BUCKET_NAME}
```

Confirm that your notification has been created for the bucket.

```sh
gsutil notification list gs://${IV_BUCKET_NAME}
```



## Preparing the Cloud Functions for Deployment



### Download the Source

Download the example code using cloud functions via nodejs.

```sh
git clone https://github.com/GoogleCloudPlatform/cloud-functions-intelligentcontent-nodejs
cd cloud-functions-intelligentcontent-nodejs
```



### Create the BigQuery dataset and table

The results of the Vision and Video Intelligence APIs are stored in BigQuery. Here we use dataset and table names set to `intelligentcontentfilter` and `filtered_content`.

Create a BigQuery dataset.

```sh
export DATASET_ID=intelligentcontentfilter
export TABLE_NAME=filtered_content

bq --project_id ${PROJECT_ID} mk ${DATASET_ID}
```

After created a dataset, we now create a BigQuery table from the schema file (`intelligent_content_bq_schema.json`) which is located on the directory we had cloned from GitHub.

```sh
bq --project_id ${PROJECT_ID} mk --schema intelligent_content_bq_schema.json -t ${DATASET_ID}.${TABLE_NAME}
```

Verify that the BigQuery table has been created.

```
bq --project_id ${PROJECT_ID} show ${DATASET_ID}.${TABLE_NAME}
```



### Edit your JSON configuration file

Before deploying the cloud functions defined in the source code, you must modify the `config.json` to use defined CLoud Storage buckets, Cloud Pub/Sub topic names, and BigQuery dataset ID and table name.

```sh
sed -i "s/\[PROJECT-ID\]/$PROJECT_ID/g" config.json
sed -i "s/\[FLAGGED_BUCKET_NAME\]/$FLAGGED_BUCKET_NAME/g" config.json
sed -i "s/\[FILTERED_BUCKET_NAME\]/$FILTERED_BUCKET_NAME/g" config.json
sed -i "s/\[DATASET_ID\]/$DATASET_ID/g" config.json
sed -i "s/\[TABLE_NAME\]/$TABLE_NAME/g" config.json
```

Notice [FLAGGED_BUCKET_NAME] and [FILTERED_BUCKET_NAME] must not include the leading protocol `gs://` index.



## Deploying the Cloud Functions



### Deploy the `GCStoPubsub` function

We will deploy the `GCStoPubsub` cloud Function, which contains the logic to receive a Cloud Storage notification message from CLoud Pub/Sub and forward the message to the appropriate function whith another Cloud Pub/Sub message.

```sh
gcloud beta functions deploy GCStoPubsub --allow-unauthenticated --stage-bucket gs://${STAGING_BUCKET_NAME} --trigger-topic ${UPLOAD_NOTIFICATION_TOPIC} --entry-point GCStoPubsub --runtime nodejs8
```

Verify that this cloud fiunction works.

```sh
gcloud beta functions logs read --filter "GCStoPubsub" --limit 100
```

Or you can delete the previous one created before.

```sh
gcloud beta functions delete GCStoPubsub
```

Notice that encountering error `finished with status: 'error'`.

![](./static/gcfunc-pubsub-error.png)

You need edit the `index.js` as the following script.

```javascript
exports.GCStoPubsub = function GCStoPubsub (event) {
  // event is already a json object
  const jsonData = Buffer.from(event.data, 'base64').toString();
  var jsonObj = event;
   
  return Promise.resolve()
    .then(() => {
	  
      // parse the string data into json format
      jsonObj = JSON.parse(jsonData);
       
      if ((typeof(jsonObj.bucket) === "undefined") || (!jsonObj.bucket)) {
      // ... 
}
```



### Deploy the `visionAPI` function

We will deploy `VisionAPI` CLoud Function, which contains the logic to receive a messgae with Cloud Pub/Sub, call the Vision API, and forward the message to the `insertIntoBigQuery` Cloud Function with another CLoud Pub/Sub message.

```sh
gcloud beta functions deploy visionAPI --allow-unauthenticated --stage-bucket gs://${STAGING_BUCKET_NAME} --trigger-topic visionapiservice --entry-point visionAPI --runtime nodejs8
```

Verify that this cloud fiunction works.

```sh
gcloud beta functions logs read --filter "VisionAPI" --limit 100
```

If there is error, you can delete the function first.

```sh
gcloud beta functions delete visionAPI
```

You need edit the `index.js` as the following script.

```javascript
exports.visionAPI = function visionAPI (event) {
  //const reqData = Buffer.from(event.data.data, 'base64').toString();
  const reqData = Buffer.from(event.data, 'base64').toString();
  
  const reqDataObj = JSON.parse(reqData);
    
  // ...
}
```



### Deploy the `videoIntelligenceAPI` function

We will deploy `videoIntelligenceAPI` CLoud Function, which contains the logic to receive a messgae with Cloud Pub/Sub, call the Video Intelligence API, and forward the message to the `insertIntoBigQuery` Cloud Function with another CLoud Pub/Sub message.

```sh
gcloud beta functions deploy videoIntelligenceAPI --allow-unauthenticated --stage-bucket gs://${STAGING_BUCKET_NAME} --trigger-topic videointelligenceservice --entry-point videoIntelligenceAPI --runtime nodejs8 --timeout 540
```

Verify that this cloud fiunction works.

```sh
gcloud beta functions logs read --filter "videoIntelligenceAPI" --limit 100
```

If there is error, you can delete the function first.

```sh
gcloud beta functions delete videoIntelligenceAPI
```

You need edit the `index.js` as the following script.

```javascript
exports.videoIntelligenceAPI = function videoIntelligenceAPI (event) 
{
  //const reqData = Buffer.from(event.data.data, 'base64').toString();
  const reqData = Buffer.from(event.data, 'base64').toString();
  
  const reqDataObj = JSON.parse(reqData);
    
  // ...
}
```

### Deploy the `insertIntoBigQuery` function

We will deploy `insertIntoBigQuery` CLoud Function, which contains the logic to receive a message with Cloud Pub/Sub and call the BigQuery API to insert the data into created BigQuery table.

```sh
gcloud beta functions deploy insertIntoBigQuery --allow-unauthenticated --stage-bucket gs://${STAGING_BUCKET_NAME} --trigger-topic bqinsert --entry-point insertIntoBigQuery --runtime nodejs8
```

Verify that this cloud fiunction works.

```sh
gcloud beta functions logs read --filter "insertIntoBigQuery" --limit 100
```

If there is error, you can delete the function first.

```sh
gcloud beta functions delete insertIntoBigQuery
```

You need edit the `index.js` as the following script.

```javascript
exports.insertIntoBigQuery = function insertIntoBigQuery(event){
  //const reqData = Buffer.from(event.data.data, 'base64').toString();
  const reqData = Buffer.from(event.data, 'base64').toString();

  const reqDataObj = JSON.parse(reqData);
    
  // ...
}
```


### Confirm that the Cloud Functions have been deployed

```sh
gcloud beta functions list
```

Verify that there are four cloud functions that have been created.



## Testing the Flow

![](./static/gcloud-func-flow.png)



### Upload an image and a video file to the upload storage bucket.

Step

*   Surf the service `Storage` and click `Browser` to open the Storage Browser.

*   Click the name of bucket with `-upload` suffix and click `Upload files`.

*   Upload some image or video files.



### Monitor Log Activity

Run the following to test `GCStoPubsub`.

```sh
gcloud beta functions logs read --filter "finished with status" "GCStoPubsub" --limit 100
```

Run the following to test `insertIntoBigQuery`.

```sh
gcloud beta functions logs read --filter "finished with status" "insertIntoBigQuery" --limit 100
```



### View Results in BigQuery

```sql
echo "
#standardSql

SELECT insertTimestamp,
  contentUrl,
  flattenedSafeSearch.flaggedType,
  flattenedSafeSearch.likelihood
FROM \`PROJECT_ID.DATASET_ID.TABLE_ID\`
CROSS JOIN UNNEST(safeSearch) AS flattenedSafeSearch
ORDER BY insertTimestamp DESC,
  contentUrl,
  flattenedSafeSearch.flaggedType
LIMIT 1000
" > sql.txt
```

Query the result.

```sh
bq --project_id ${PROJECT_ID} query < sql.txt
```


















