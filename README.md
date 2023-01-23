# Upload files to Amazon S3 from your browser, using Javascript

This memo, dated 2023-01-23, describes the steps I had to take
to setup a file uploader to an Amazon S3 bucket.  Bear in mind
that all links presented below may have changed when you read
this, but the text should hopefully guide you to the right page
(amongst Amazon's 32058242 pages).

## Register an account

Create a root user account and login
```
https://aws.amazon.com
```

## Create a bucket

After you're logged in, go to "S3" so you can see "Buckets" in the left-hand column:
```
https://s3.console.aws.amazon.com/s3/buckets
```
followed by an argument which describes your region - in my case `eu-north-1`.

However,
I want my bucket to be in `eu-west-2` (closer to my clients).

Now click "Create bucket" and give it a name (lowercase),
choose the region which is closest to you or your clients
and disable ACL (access control lists).
Under "Block all public access" I found that I had to block everything except
"Block public access to buckets and objects granted through new access control lists".

You can choose versioning and alternative encryption methods
but I went with the default settings.

There are a few (to put it mildly) settings for the bucket which we have
to configure before we get to the actual program.
If the program doesn't work it is most likely because of one of these settings.

## Bucket permissions

Go to the bucket's "Permissions" tab and edit the "Bucket policy":
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::BUCKETNAME/*"
        },
        {
            "Sid": "PolicyForAllowUploadWithACL",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:PutObjectACL",
            "Resource": "arn:aws:s3:::BUCKETNAME/*"
        }
    ]
}
```
Here you must of course replace "BUCKETNAME" with your chosen bucket name.

## Bucket policy

In the same tab, edit the "Bucket policy":
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::BUCKETNAME/*"
        },
        {
            "Sid": "PolicyForAllowUploadWithACL",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:PutObjectACL",
            "Resource": "arn:aws:s3:::BUCKETNAME/*"
        }
    ]
}
```
where "BUCKETNAME" again is the name of your bucket.

## Object ownership

In the same tab, click on "Object Ownership"
and set it to "ACLs enabled".


## CORS

In the same tab, enter the following CORS settings:
```
[
    {
        "AllowedHeaders": [
            "*"
        ],
        "AllowedMethods": [
            "GET",
            "HEAD",
            "PUT",
            "POST",
            "DELETE"
        ],
        "AllowedOrigins": [
            "https://example.com"
        ],
        "ExposeHeaders": [
            "ETag"
        ],
        "MaxAgeSeconds": 3000
    }
]
```
Here you should replace `https://example.com` with the domain from which
you will be hosting your script (shown further down).  You can enter more
than one domain here, or none at all.  Bear in mind that it's easy for
someone to spoof your address so this is not as secure as you may think.

## Cognito

Now we need to head over to the Cognito page, and "Manage Identity Pools".
Click on "Services" in the upper-left corner and then choose "Cognito" from
the list of all services, then choose "Manage Identity Pools".

Click on "Create new identity pool", choose a name and check
"Enable access to unauthenticated identities" and click on "Create pool".
On the next page, just click "Allow" and you will then see "Sample code"
where an important Identity Pool ID is shown.  You need to save this ID:
```
eu-west-2:2081718f-8d70-4232-bf1f-xxxxxxxxxxxx
```
(I have chosen to omit the last string of digits.)

Pay attention to the region prefix in the pool ID!
I had problems generating a pool ID for my desired region, `eu-west-2` and
found that Amazon S3 defaulted to `eu-north-1`.  I had to change the browser's
URL until I got it right.

Make a note of the name for the unauthenticated role's name.
It should be something like "Cognito_nameUnauth_Role" where name is
your identity pool name.  You will use it next.

## IAM

Yet another wonderful acronym.  (Apparently it stands for "Identity and Access Management".)
This one controls access rights to the bucket.
Go to "Services", "All services" and "IAM".
Then, in the left column, choose "Roles".
In the list of roles, choose the unauthenticated role's name from
the previous section.
Now, in the "Permissions" tab for that role, click on the policy name
which takes you to a JSON tab.  Edit the JSON to look like this:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "mobileanalytics:PutEvents",
                "cognito-sync:*",
                "s3:*"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```
Then click on "Review policy" and "Save changes".

## Javascript program

Still with me?  Fantastic!  Now let's write the actual program.
This HTML-file with some javascript will let you upload one or more files to the bucket.
In the code below you will need to change (1), (2) and (3).

```
<!DOCTYPE html>
<html>

<head>
    <title>AWS S3 File Upload</title>
    <script src="https://sdk.amazonaws.com/js/aws-sdk-2.1.12.min.js"></script>
</head>

<body>
    <input type="file" id="file-chooser" multiple />
    <button id="upload-button">Upload to S3</button>
    <div id="results"></div>
    <script type="text/javascript">
    AWS.config.region = 'eu-west-2'; // 1. Enter your region
    AWS.config.credentials = new AWS.CognitoIdentityCredentials({
        IdentityPoolId: "eu-west-2:2081718f-8d70-4232-bf1f-xxxxxxxxxxxx" // 3. Your pool ID
    });

    AWS.config.credentials.get(function(err) {
        if (err)
	    alert(err);
        console.log(AWS.config.credentials);
    });

    var bucket = new AWS.S3({
        params: {
            Bucket: 'BUCKETNAME'  // 2. Replace with your bucket name
        }
    });
    var fileChooser = document.getElementById('file-chooser');
    var button = document.getElementById('upload-button');
    var results = document.getElementById('results');

    function listObjs() {
        var prefix = ''; 	// this can be a user ID or project ID
        bucket.listObjects({
            Prefix: prefix
        }, function(err, data) {
            if (err) {
                results.innerHTML = 'ERROR: ' + err;
            } else {
                var objKeys = "";
                data.Contents.forEach(function(obj) {
                    objKeys += obj.Key + "<br>";
                });
                results.innerHTML = objKeys;
            }
        });
    }

    function upload(file) {
	if (!file)
	    return;
        results.innerHTML = '';
        var objKey = file.name;
        var params = {
            Key: objKey,
            ContentType: file.type,
            Body: file,
            ACL: 'public-read'
        };
        bucket.putObject(params, function(err, data) {
            if (err) {
                results.innerHTML = 'ERROR: ' + err;
            } else {
		listObjs(); // list all uploaded files
                // Here you can also add your code to update your database
            }
        });
    }

    button.addEventListener('click', function() {
        var files = fileChooser.files;
	for (var i = 0; i < files.length; i++)
	    upload(files[i]);
    }, false);

    </script>
</body>
</html>
```
(This program is based on https://github.com/rishiv3/s3-multiple-file-upload-angularJS )

If there are any errors, the program will tell you.
