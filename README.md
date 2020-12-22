# Label Studio with Pachyderm

[Label Studio](https://labelstud.io/) supports many different types of data labeling tasks, while Pachyderm allows you to incorporate data versioning and data-driven pipelines. This integration connects a Pachyderm versioned data backend with Label Studio to support versioning datasets and tracking the data lineage of pipelines built off the versioned datasets.

## How it works

Label Studio can utilize an s3 backend, reading data from an S3 bucket and writing labels to an output S3 location. Pachyderm has an S3 compliant gateway that allows reading data from its file system and writing data to its filesystem (organizing it with commits that can start pipelines).

In this example, we:

1. Start a Label Studio instance that uses Pachyderm as its backend
2. Push data to Pachyderm that automatically populates Label Studio
3. Label the data in Label Studio
4. Version our dataset in Pachyderm


## TLDR;

``` bash
# Pachyderm Setup
pachctl create repo raw_data
pachctl create repo labeled_data
pachctl create branch labeled_data@master
pachctl create branch raw_data@master

# Start a local instance of Label Studio (needs the .aws/config for the Pach s3 gateway)
# docker run -it --entrypoint=/bin/sh -v $(pwd)/.aws:/root/.aws -v $(pwd):/pfs/project_config/ -p 8080:8080 jimmywhitaker/label-studio:latest
docker run -it --entrypoint=/bin/sh --env-file .env -v $(pwd)/my-text-project:/my-text-project -p 8080:8080 jimmywhitaker/label-studio:latest

# Copy s3 config into container
cp -r /pfs/project_config/.aws ~/.aws

# Create a project and run label studio
label-studio start /pfs/project_config/my_s3_project/ --init --source s3 --source-path master.raw_data --input-format=text --target s3-completions --target-path master.labeled_data --force --log-level DEBUG --source-params "{\"use_blob_urls\": false, \"regex\": \".*\"}"
## Note: need to check LS configuration - select Text classification or copy it over from starter project 

# Upload data
pachctl put file raw_data@master:/test-example.json -f raw_data/test-example.json --split json --target-file-datums 1

# Update the data
pachctl put file raw_data@master:/test-example.json -f raw_data/test-example2.json --split json --target-file-datums 1 --overwrite

# Label data (2 examples) in the UI

# Version your dataset (v1)
pachctl list branch labeled_data
pachctl create branch labeled_data@v1 --head master
pachctl list branch labeled_data

# Label more data in the UI

# Version your dataset (v2)
pachctl list branch labeled_data
pachctl create branch labeled_data@v2 --head master

# Download dataset for v1 locally
pachctl get file -r labeled_data@v1:/ -o labeled_data/

```

## Configuring .env file
The `.env` file needs to be configured for your Pachyderm s3 gateway. Pachyderm's s3 gateway is accessed through an `http` endpoint that is available on port `30600` on the Pachyderm cluster. This address is used to as the `ENDPOINT_URL` for the Label Studio backend in the `.env` file. 

### Minikube configuration
If you are running Pachyderm locally on minikube, you can get the `ENDPOINT_URL` for the Pachyderm s3 gateway by running the command:

```shell
minikube ip
```


## Current Issues/Not Supported

Future support:The output does have a reference for what the input file location was (could potentially be used to track consistency between raw and labeled if raw changes).

We can load data into Pachyderm, label it, and output the result. 
**We can change labels and see the versions of the labels? 

### Next Steps

* Build an image that can connect with the s3 gateway (right now only supports s3 and gcs)
  * Add the s3 config and change the s3.py file with the function below. 
  * Either need to modify the s3 connection to take an endpoint_url or have a dedicated minio handler
  * How are credentials mapped into the container? env variables? 
* Make deployment super easy
  * Build a helm chart to deploy label studio in Hub
  * Standardize label studio project creation - do we need different examples of configs? 
* Ability to update `input raw data` - currently if it's labeled, then it's captured in the source and target json files. 
* Rectify the source and target files to have provenance for the labeling


### Known Issues and Gotchas 

* One example per file
* Must be json files or figure out how to get s3 signed urls to frontend. 
* When file is updated after labeled, it's not re-loaded (not sure what should happen here - should it be removed from the labeled data repo when the raw data is removed?)
  * When raw data is changed after that example is labeled, the task doesn't update. It does update when 
  * It seems as though the target and the source states are tied somehow, so it won't automatically update
  * If a raw file is removed or changed, then labels associated with that file should be removed. Since it's a single file per example, a changed file should be the deleting of one and addition of another. For now, this would need to be an external process that 
* Once you've created a project and labeled data, the same starting configuration should be used. Label Studio automatically tries to start an image labeling config and if there is labeled data, this will throw errors until you load a compatible config for what's already labeled (i.e. you should not use the `--init` and `--force` flags after you've created the project).


## Notes:

Label studio currently doesn't support arbitrary s3 storage, so this was a hack that allowed us to connect to the Pachyderm s3 gateway running locally. 

```python
def get_client_and_resource(
    aws_access_key_id=None, aws_secret_access_key=None, aws_session_token=None, region=None
):
    session = boto3.Session(
        profile_name='default')
    settings = {}
    region = region or S3_REGION
    if region:
        settings['region_name'] = region
    settings['endpoint_url'] = 'http://192.168.64.8:30600'
    client = session.client('s3', config=boto3.session.Config(signature_version='s3v4'), **settings)
    resource = session.resource('s3',
                    endpoint_url='http://192.168.64.8:30600',
                    aws_access_key_id='',
                    aws_secret_access_key='',
                    config=Config(signature_version='s3v4'),
                    region_name='us-east-1')
    return client, resource
```

```py
ENDPOINT_URL = os.environ.get('ENDPOINT_URL')

def get_client_and_resource(
    aws_access_key_id=None, aws_secret_access_key=None, aws_session_token=None, region=None
):
    session = boto3.Session(
        aws_access_key_id=aws_access_key_id,
        aws_secret_access_key=aws_secret_access_key,
        aws_session_token=aws_session_token)
    settings = {}
    region = region or S3_REGION
    if region:
        settings['region_name'] = region
    settings['endpoint_url'] = ENDPOINT_URL or None

    client = session.client('s3', config=boto3.session.Config(signature_version='s3v4'), **settings)
    resource = session.resource('s3',
                    endpoint_url=settings['endpoint_url'])
    return client, resource
```