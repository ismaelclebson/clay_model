# Running embeddings for Worldcover Sentinel-2 Composites
This package is made to generate embeddings from the [ESA Worldcover](https://esa-worldcover.org/en/data-access)
Sentinel-2 annual composites. The target region is all of the
Contiguous United States.

We ran this script for 2020 and 2021.

## The algorithm

The `run.py` script will run through a column of image chips of 512x512 pixels.
Each run is a column that spans the Contiguous United States from north to
south. For each chip in that column, embeddings are generated and stored
together in one geoparquet file. These files are then uploaded to the
`clay-worldcover-embeddings` bucket on S3.

There are 1359 such columns to process in order to cover all of the Conus US.

The embeddings are stored alongside with the bbox of the data chip used for
generating the embedding. To visualize the underlying data or an embedding
the WMS and WMTS endpoints provided by the ESA Worldcover project can be used.

So the geoparquet files only have the following two columns

|    embeddings    |   bbox   |
|------------------|--------------|
| [0.1, 0.4, ... ] | POLYGON(...) |
| [0.2, 0.5, ... ] | POLYGON(...) |
| [0.3, 0.6, ... ] | POLYGON(...) |

## Exploring results

The `embeddings_db.py` script provides a way to locally explore the embeddings.
It will create a `lancedb` database and allow for search. The search results are
visualizded by requesting the RGB image from the WMS endpoint for the bbox of
each search result.

## Running on Batch

### Upload package to fetch and run bucket
This snippet will create the zip package that is used for the fetch-and-run
instance in our ECR registry.

```bash
# Add clay src and scripts to zip file
zip -FSr batch-fetch-and-run-wc.zip src scripts -x *.pyc -x scripts/worldcover/wandb/**\*

# Add run to home dir, so that fetch-and-run can see it.
zip -uj batch-fetch-and-run-wc.zip scripts/worldcover/run.py

# Upload fetch-and-run package to S3
aws s3api put-object --bucket clay-fetch-and-run-packages --key "batch-fetch-and-run-wc.zip" --body "batch-fetch-and-run-wc.zip"
```

### Push array job
This command will send the array job to AWS batch to run all of the
1359 jobs to cover the US.

```python
import boto3

batch = boto3.client("batch", region_name="us-east-1")
year = 2020
job = {
    "jobName": f"worldcover-conus-{year}",
    "jobQueue": "fetch-and-run",
    "jobDefinition": "fetch-and-run",
    "containerOverrides": {
        "command": ["run.py"],
        "environment": [
            {"name": "BATCH_FILE_TYPE", "value": "zip"},
            {
                "name": "BATCH_FILE_S3_URL",
                "value": "s3://clay-fetch-and-run-packages/batch-fetch-and-run-wc.zip",
            },
            {"name": "YEAR", "value": f"{year}"}
        ],
        "resourceRequirements": [
            {"type": "MEMORY", "value": "7500"},
            {"type": "VCPU", "value": "4"},
            # {"type": "GPU", "value": "1"},
        ],
    },
    "arrayProperties": {
        "size": int((125 - 67) * 12000 / 512)
    },
    "retryStrategy": {
      "attempts": 5,
      "evaluateOnExit": [
        {"onStatusReason": "Host EC2*", "action": "RETRY"},
        {"onReason": "*", "action": "EXIT"}
      ]
    },
}

print(batch.submit_job(**job))
```
