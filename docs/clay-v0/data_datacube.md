# Creating datacubes

## How to create a datacube

The `datacube.py` script collects Sentinel-2, Sentinel-1, and DEM data over individual MGRS tiles. The source list of the MGRS tiles to be processed is provided in an input file with MGRS geometries. Each run of the script will collect data for one of the MGRS tiles in the source file. The tile to be processed is based on the row index number provided as input. The MGRS tile ID is expected to be in the `name` property of the input file.

For the target MGRS tile, the script loops through the years between 2017 and 2023 in random order. For each year, it will search for the least cloudy Sentinel-2 scene. Based on the date of the selected Sentinel-2 scene, it will search for the Sentinel-1 scenes that are the closest match to that date, with a maximum of +/- 3 days of difference. It will include multiple Sentinel-1 scenes until the full MGRS tile is covered. If no matching Sentinel-1 scenes can be found, the script moves to the next year. The script stops when 3 matching datasets have been collected for 3 different years. Finally, the script will also select the intersecting part of the Copernicus Digital Elevation Model (DEM).

The script will then download the Sentinel-2 scene and match the data cube with the corresponding Sentinel-1 and DEM data. The scene-level data is then split into smaller chips of a fixed size of 512x512 pixels. The Sentinel-2, Sentinel-1 and DEM bands are then packed together in a single TIFF file for each chip. These are saved locally and synced to a S3 bucket at the end of the script. The bucket name can be specified as input.

For testing and debugging, the data size can be reduced by specifying a pixel window using the `subset` parameter. Data will then be requested only for the specified pixel window. This will reduce the data size considerably which speeds up the processing during testing.

The example run below will search for data for the geometry with row index 1 in a local MGRS sample file for a 1000x1000 pixel window.

```bash
python datacube.py --sample /home/user/Desktop/mgrs_sample.fgb --bucket "my-bucket" --subset "1000,1000,2000,2000" --index 1
```

## Running the datacube pipeline as a batch job

This section describes how to containerize the data pipeline and run it on AWS Batch Spot instances using
a [fetch-and-run](https://aws.amazon.com/blogs/compute/creating-a-simple-fetch-and-run-aws-batch-job/)
approach.

### Prepare docker image in ECR

Build the docker image and push it to a ecr repository.

```bash
ecr_repo_id=12345
cd scripts/pipeline/batch
docker build -t $ecr_repo_id.dkr.ecr.us-east-1.amazonaws.com/fetch-and-run .

aws ecr get-login-password --profile clay --region us-east-1 | docker login --username AWS --password-stdin $ecr_repo_id.dkr.ecr.us-east-1.amazonaws.com

docker push $ecr_repo_id.dkr.ecr.us-east-1.amazonaws.com/fetch-and-run
```

### Prepare AWS batch

To prepare a batch, we need to create a compute environment, job queue, and job
definition.

Example configurations for the compute environment and the job definition are
provided in the `batch` directory.

The `submit.py` script contains a loop for submitting jobs to the queue. An
alternative to these individual job submissions would be to use array jobs, but
for now the individual submissions are simpler and failures are easier to track.

### Create ZIP file with the package to execute

Package the model and the inference script into a zip file. The `datacube.py`
script is the one that will be executed on the instances.

Put the scripts in a zip file and upload the zip package into S3 so that
the batch fetch-and-run can use it.

```bash
zip -FSrj "batch-fetch-and-run.zip" ./scripts/pipeline* -x "scripts/pipeline*.pyc"

aws s3api put-object --bucket clay-fetch-and-run-packages --key "batch-fetch-and-run.zip" --body "batch-fetch-and-run.zip"
```

### Submit job

We can now submit a batch job to run the pipeline. The `submit.py` file
provides an example on how to submit jobs in python.
