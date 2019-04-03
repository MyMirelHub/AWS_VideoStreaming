#  Using an AWS elastic transcoder for video streaming. 

In this project we will configure the AWS account to accept video files via an S3 bucket, and convert them to gifs or use them for streaming. 

Inline-style: 
![alt text](https://github.com/MyMirelHub/AWS_VideoStreaming/blob/master/images/Elastic%20Transcode.png?raw=true "Logo Title Text 1")

## Process 

Summary of steps taken 

1. Create two S3 buckets, one for storing the input video files and the other for the output. 
2. Create a transcoder pipeline for converting the content we uploaded into S3.
3. Use a AWS Lambda function to automatically trigger a conversion job and push the file ready for playing on the S3 bucket.
4. Configure a cloud front distribution for low latency streaming through amazons (CDN
5. Configure an SNS email notification on successful completion or failure of the job

## Creating S3 input and output buckets

 Create two s3 Buckets named

- transcodeinputmirel
  - Default settings
- transcodeoutpoutmirel 
  - Give this a public permission
  
https://github.com/MyMirelHub/AWS_VideoStreaming/blob/master/images/screenshot-s3.console.aws.amazon.com-2019.04.03-21-07-41.png?raw=true

Paste permission file in the bucket policy editor. 

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::transcodeoutputmirel/*"
        }
    ]
}
```



## SNS Topic 

Create two Topics named ```Errror``` and ```Complete``` . 

Subscribe the topics to to send Gmail notifications. 



## Elastic Transcoder Pipeline

Amazon Elastic Transcoder lets you convert digital media stored in Amazon S3 into the audio and video codecs and the containers required by consumer playback devices. For example, you can convert large, high-quality digital media files into formats that users can play back on mobile devices, tablets, web browsers, and connected televisions.

Elastic Transcoder has three components:

- **Pipelines** are queues that manage your transcoding jobs. Elastic Transcoder begins to process jobs in the order in which you add them to a pipeline. Typically, you’ll create at least two pipelines, one for standard-priority jobs and one for high-priority jobs. Most jobs go into the standard-priority pipeline; you use the high-priority pipeline only when you need a file to be transcoded immediately.
- **Jobs** specify the settings that aren’t included in the preset, for example, the file to transcode and whether to create thumbnails. Each job converts one file into one different format. When you create a job, Elastic Transcoder adds it to the pipeline you specify. If there are already jobs in the pipeline, Elastic Transcoder begins processing the new job when resources are available.
- **Presets** are templates that specify most of the settings for the transcoded media file. Elastic Transcoder includes some default presets for common formats. You can also create your own presets. When you create a job, you specify which preset to use.



The following transcode configuration was selected. 

https://github.com/MyMirelHub/AWS_VideoStreaming/blob/master/images/screenshot-eu-west-1.console.aws.amazon.com-2019.04.03-21-10-01.png?raw=true

In this process only the pipeline is created, as the job and the pre-set is specified in the lambda. 

Created an IAM role for it. Since unsure about looking into the permission I attached the policy of giving it Full Lambda permissions. In a practical use case you would have just a job submission lambda policy to it. 

Specify output bucket. Storage class is this one is set to reduced redundancy, as data is easily reproducible, to save costs. 

 Image applies the same, but considering the tiny amount of space it doesn't matter. 

https://github.com/MyMirelHub/AWS_VideoStreaming/blob/master/images/screenshot-eu-west-1.console.aws.amazon.com-2019.04.03-21-10-31.png?raw=true


## Lambda 

If an object is uploaded to an S3. The Lambda is triggered to react to the S3. The functions creates the job and the pipeline and has the video sent to the output bucket. 

```js
//Node.js 6.10 
var AWS = require('aws-sdk');
var s3 = new AWS.S3({
  apiVersion: '2012–09–25'
});
var eltr = new AWS.ElasticTranscoder({
  apiVersion: '2012–09–25',
  region: 'eu-west-1'
});
// Expots handler 
exports.handler = function(event, context) {
  var bucket = event.Records[0].s3.bucket.name;
  var key = event.Records[0].s3.object.key;
  var pipelineId = '1554190625819-opoph3';

  if (bucket !== 'transcodeinputmirel') {
    context.fail('Incorrect input bucket');
    return;
  }
  
  var srcKey = decodeURIComponent(event.Records[0].s3.object.key.replace(/\+/g, " ")); // the object may have spaces  
  var newKey = key.split('.')[0];
  var params = {
    PipelineId: pipelineId,
    OutputKeyPrefix: newKey + '/',
    Input: {
      Key: srcKey,
      FrameRate: 'auto',
      Resolution: 'auto',
      AspectRatio: 'auto',
      Interlaced: 'auto',
      Container: 'auto'
    },
    Outputs: [{
      Key: newKey + '.gif',
      ThumbnailPattern: 'thumbs-' + newKey + '-{count}',
      PresetId: '1351620000001-100200',
      Watermarks: [],
    }]
  };
 
  console.log('Starting a job');
  eltr.createJob(params, function(err, data) {
    if (err){
      console.log(err);
    } else {
      console.log(data);
    }
    context.succeed('Job successfully completed');
  });
};
```

Full of presents can be found here: 

https://docs.aws.amazon.com/elastictranscoder/latest/developerguide/system-presets.html>

## Cloudfront Distribution 

Delivery method select HTTP, as we dont want to use flash, and using a web distibution. 

Origin name, output bucket thats what we are taking info from. 

- Restrict bucket access - yes.  The S3 bucket is adding an ACL to restrict access to the cloudfront url. 

  - For this lab it was no

  

#### Credits

<https://medium.com/@kout.petr/using-aws-for-video-streaming-7a46264fcfb6>

www.linuxacademy.com

<https://hackernoon.com/youtube-to-mp3-using-s3-lambda-elastic-transcoder-df2c5903dea8>

<https://medium.com/@ratulbasak93/video-to-audio-using-elastic-transcoder-lambda-with-sns-notification-cdc8ba8a6e78>

 
