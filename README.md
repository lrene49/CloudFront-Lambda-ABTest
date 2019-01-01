# CloudFront-Lambda@Edge-A/BTest
See the following sections for examples of using Lambda functions with CloudFront.

# Scenario

You have a website, static content served by an S3 bucket. Content is cacehed by AWS CloudFront CDN.

You can use the following example if you want to test two different versions of your home page, but you don't want to create redirects or change the URL. This example sets cookies when CloudFront receives a request, randomly assigns the user to version A or B, and then returns the corresponding version to the viewer.

Each version of the website is served by a separate URL(e.g. separate S3 Bucket). They are configured as different Origins in the CloudFront.

# How it works
We want to keep the served version stable with a cookie,  Origin Vaule = B. As content is served by S3 bucket the cookie can't be added by the server. So, on the first request by a new client, we randomly decide a version.

let's recap the scenario:
1. Client access on the websit. The brower request is directed to the closest CloudFront Edge location. This request does't    contain the cookie
2. Because doesn't contain the cookie, the request is forwarded to the Origin.
3. The function looks for cookie. Not finding it, it will randomly decide which version to send the client.
4. 


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
