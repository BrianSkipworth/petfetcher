# :rabbit: Petfetcher
#### _An AWS Lambda function written in Python to get the pets listed on Petfinder.com that are available for adoption from a specific animal rescue organization_

This script generates a static web page that can be shown inside an iframe on the organization's existing website. See the example inner page this generates for <a href="https://herdandflock.s3.us-west-1.amazonaws.com/animals.html" target="_blank">Herd and Flock Animal Sanctuary</a>. 

Schedule the function to run as frequently as needed, for most scenarios a nightly run is adequate and the rescue staff can expect that any animals they add to Petfinder will appear on the website the following day. This script uses only AWS services, including:

- Lambda to run the script that gets the list
- S3 to store the static web page that shows the animals
- EventBridge to schedule regular updates to the page (manually configured separate from this code)

## Set Up

On AWS create a new S3 bucket and select Access: Public. The following Bucket Policy allows anyone to view the page.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicRead",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::herdandflock/*"
        }
    ]
}
```

Next create a new Lambda function. The code in lambda.py can be copy-pasted on the Code tab in the Lambda screen, though a few additional steps will be required to add external Python libraries to the Lambda function's runtime environment. Also, to avoid a current issue with a conflict between a necessary library and AWS's boto3 SDK, we have to revert the library to an older version before deploying the Lambda function.

Complete Step 1 described in the Medium post <a href="https://medium.com/@gauravkachariya/how-to-add-external-python-libraries-to-aws-lambda-499674113fb7" target="_blank">How to add External Python Libraries to AWS Lambda</a>.

Before moving on to Step 2, remove the directory for the package `urllib3` which the requests library has installed as a dependency. Then, manually re-install an older version using the command `pip install --upgrade urllib3==1.26.18 -t .` This extra step is necessary to workaround a conflict between urllib3 and Amazon's boto3 SDK.

```
(venv) [cloudshell-user@ip-10-6-25-93 python]$ pip install --upgrade urllib3==1.26.18 -t .
Collecting urllib3==1.26.18
  Using cached urllib3-1.26.18-py2.py3-none-any.whl (143 kB)
Installing collected packages: urllib3
Successfully installed urllib3-1.26.18
```
Now complete Steps 2 and 3.


### Petfinder API
You will need to generate a client and secret. Sign up on their site to get your credentials: https://www.petfinder.com/user/developer-settings/
 <a href="https://www.petfinder.com/developers/v2/docs/">Petfinder API Docs</a>

### Environment Variables
Almost all configuration details that are specific to the rescue organization are stored in environment variables, with the exception of style code to make the generated web page match the rescue's existing website (styling is hardcoded). This enables the script can be set up for a new rescue organization quickly and allows information that may change frequently to be updated without needing to edit the code.

| Key | Value |
|---|---|
| BUCKET | The name of the S3 bucket |
| CLIENT | A Petfinder API Key |
| SECRET | A Petfinder Secret |
| ORG | The sanctuary's Organization ID on Petfinder in lowercase, e.g. 'ca3085' |
| FORM | Full URL of the adoption form including the query param ?animal_name= |
| RECIPIENT_EMAIL | Where to send alert emails when an error occurs |
| SENDER_EMAIL | Reply-to email for alert emails |

When you have added all of the necessary variables, the Configuration tab will appear as follows.

<figure>
    <img src="https://herdandflock.s3.us-west-1.amazonaws.com/herd+and+flock+petfinder+lambda+vars.png" width="600"
         alt="Complete Environment Variables">
    <figcaption></figcaption>
</figure>

In the function code we begin by retrieving these environment variables.

```python
bucket_name = os.environ['BUCKET']
client_id = os.environ['CLIENT']
client_secret = os.environ['SECRET']
default_img = os.environ['IMG'] # URL of an image to show if none is available for the animal
form_url = os.environ['FORM'] # Full URL of the adoption form including the query param ?animal_name=
org_id = os.environ['ORG'] # The sanctuary's Organization ID on Petfinder in lowercase, e.g. 'ca3085'
```

### Styling
The remaining configuration is optional, but recommended to make the adoption list match the style of the rescue organization's website.

Within the definition of `make_html` there is hard-coded CSS styling. Where the variable `html` is instantiated, edit the code following `style=` to make the buttons shape and color match the website. 

You may also want to choose a different button label, which is hardcoded into the variable named `para` to display "Apply to Adopt" on the buttons.

Ready to fetch!
