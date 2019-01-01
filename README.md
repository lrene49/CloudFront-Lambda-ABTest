# CloudFront-Lambda@Edge-A/BTest
See the following sections for examples of using Lambda functions with CloudFront.



## Scenario

You have a website, static content served by an S3 bucket. Content is cacehed by AWS CloudFront CDN.

You can use the following example if you want to test two different versions of your home page, but you don't want to create redirects or change the URL. This example sets cookies when CloudFront receives a request, randomly assigns the user to version A or B, and then returns the corresponding version to the viewer.

Each version of the website is served by a separate URL(e.g. separate S3 bucket). They are configured as different Origins in the CloudFront.

## How it works
We want to keep the served version stable with a cookie,  Origin Vaule = B. As content is served by S3 bucket the cookie can't be added by the server. So, on the first request by a new client, we randomly decide a version.

Recap the scenario:
1. Client access on the websit. The brower request is directed to the closest CloudFront Edge location. 
2. CloudFront serves content from cache if available, otherwise it goes to origin.
3. Only after a CloudFront cache miss, the origin request trigger is fired for that behavior. (Modify origin lambda   
   functions to determine which origin to ourte the request to.)
4. CloudFront sents the request to the chosen origin.
5. The object is returned to CloudFront from S3, served to the viewer and caches, if applicable.


2. Because doesn't contain the cookie, the request is forwarded to the Origin.
3. The function looks for cookie. Not finding it, it will randomly decide which version to send the client.


## Configuration 
### S3
Create two S3 buckets as origins (e.g. cfbucket01 and cfbucket02) a CloudFront distribution and an AWS Lambda function that routes a user's request to one of the two S3 origins based on a cookie. 

Upload a basic page.html file that identifies each bucket when I navigate to it. 


### Granting the Origin Access Identity Permission to Read Files in S3 bucket

When you create or update a distribution, you can add an origin access identity and automatically update the bucket policy to give the origin access identity permission to access your bucket. Alternatively, you can choose to manually change the bucket policy or change ACLs, which control permissions on individual files in your bucket.

Whichever method you use, you should still review the bucket policy for your bucket and review the permissions on your files to ensure that:

CloudFront can access files in the bucket on behalf of users who are requesting your files through CloudFront.
Users can't use Amazon S3 URLs to access your files.

https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html#private-content-creating-oai-console

### CloudFront



### Lambda@Edge

Copy the following code into the Function code box and ensure Node.js 6.10 Runtime and index.handler are selected. The bucket names in the following code need to be replaced with your origin names.

    exports.handler = (event, context, callback) => {
    const request = event.Records[0].cf.request;
    const headers = request.headers;
    const origin = request.origin;
    
    console.log('origin:');
    console.log(origin);

    //Setup the two different origins
    const originA = "cfbucket01.s3.amazonaws.com";
    const originB = "cfbucket02.s3.amazonaws.com";
    
    console.log(request);

   
    //Determine whether the user has visited before based on a cookie value
    //Grab the 'origin' cookie if it's been set before
    if (headers.cookie) {
        for (let i = 0; i < headers.cookie.length; i++) {
            if (headers.cookie[i].value.indexOf('origin=A') >= 0) {
                console.log('Origin A cookie found');
                headers['host'] = [{key: 'host',          value: originA}];
                origin.s3.domainName = originA;
                break;
            } else if (headers.cookie[i].value.indexOf('origin=B') >= 0) {
                console.log('Origin B cookie found');
                headers['host'] = [{key: 'host',          value: originB}];
                origin.s3.domainName = originB;
                break;
            }
        }
    } else
    {
        //New visitor so no cookie set, roll the dice weight to origin A
        //Could also just choose to return here rather than modifying the request
        if (Math.random() < 0.75) {
            headers['host'] = [{key: 'host',          value: originA}];
            origin.s3.domainName = originA;
            // origin.domainName = "www.gooogle.com";
            console.log('Rolled the dice and origin A it is!');
        } else {
            headers['host'] = [{key: 'host',          value: originB}];
            origin.s3.domainName = originB;
            // origin.domainName = "www.gooogle.com";
            console.log('Rolled the dice and origin B it is!');
        }
    }

    

    callback(null, request);
		};
