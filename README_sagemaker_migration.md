# SageMaker MLflow Migration with Preserved S3 Artifacts

## Overview

This guide explains how to migrate experiments and runs between SageMaker MLflow instances while preserving artifact paths in S3, without copying the artifact files themselves.

## Use Case

When migrating MLflow data from SageMaker MLflow instance A to instance B within the same AWS account:
- Both instances have access to the same S3 bucket containing artifacts
- You want to preserve artifact references without copying large files
- You want artifacts to remain viewable in the destination MLflow instance

## Solution: `skip_download_run_artifacts` Option

The `--skip-download-run-artifacts` option allows you to export MLflow experiments and runs while skipping the download of artifact files. This is useful when:

1. **Artifacts are in shared storage**: Both source and destination MLflow instances can access the same S3 bucket
2. **Large artifacts**: Avoid downloading and re-uploading large artifact files
3. **Preserve original URIs**: Maintain the original S3 paths for artifacts

## How It Works

### During Export

When `--skip-download-run-artifacts True` is specified:

1. **Metadata is exported**: All run metadata (parameters, metrics, tags, etc.) is exported normally
2. **Artifacts are NOT downloaded**: The artifact files themselves are not downloaded from S3
3. **artifact_uri is preserved**: The `artifact_uri` field in the exported JSON maintains the original S3 path

Example exported `run.json`:
```json
{
  "mlflow": {
    "info": {
      "run_id": "abc123...",
      "artifact_uri": "s3://my-mlflow-bucket/experiments/1/abc123/artifacts",
      ...
    },
    ...
  }
}
```

### During Import

When importing the exported data:

1. **Metadata is imported**: All run metadata is restored
2. **artifact_uri is available**: The original S3 path is preserved in the run information
3. **Artifacts remain accessible**: If both MLflow instances have S3 access, artifacts are viewable

## Usage Examples

### Single Experiment Export

```bash
# Export a single experiment without downloading artifacts
export-experiment \
  --experiment my-experiment \
  --output-dir /tmp/export \
  --skip-download-run-artifacts True

# Import to destination MLflow instance
import-experiment \
  --experiment-name my-experiment-imported \
  --input-dir /tmp/export
```

### Bulk Experiments Export

```bash
# Export multiple experiments without downloading artifacts
export-experiments \
  --experiments "experiment1,experiment2,experiment3" \
  --output-dir /tmp/bulk-export \
  --skip-download-run-artifacts True

# Import all experiments
import-experiments \
  --input-dir /tmp/bulk-export
```

### Export All Experiments

```bash
# Export all experiments without downloading artifacts
export-experiments \
  --experiments all \
  --output-dir /tmp/all-export \
  --skip-download-run-artifacts True
```

## SageMaker MLflow Migration Steps

### Prerequisites

1. **IAM Permissions**: Ensure both SageMaker MLflow instances have S3 access
2. **Same S3 Bucket**: Both instances should have access to the same artifact bucket
3. **MLflow Export-Import Tool**: Install the tool in your environment

### Step-by-Step Migration

#### 1. Set Source MLflow Tracking URI

```bash
export MLFLOW_TRACKING_URI=<source-sagemaker-mlflow-url>
```

#### 2. Export from Source (Instance A)

```bash
# Export all experiments without downloading artifacts
export-experiments \
  --experiments all \
  --output-dir /tmp/sagemaker-migration \
  --skip-download-run-artifacts True
```

#### 3. Switch to Destination MLflow Tracking URI

```bash
export MLFLOW_TRACKING_URI=<destination-sagemaker-mlflow-url>
```

#### 4. Import to Destination (Instance B)

```bash
# Import all experiments
import-experiments \
  --input-dir /tmp/sagemaker-migration
```

#### 5. Verify Migration

```python
import mlflow

# Set destination tracking URI
mlflow.set_tracking_uri("<destination-sagemaker-mlflow-url>")

# Get an experiment
client = mlflow.MlflowClient()
experiments = client.search_experiments()

for exp in experiments:
    runs = client.search_runs(exp.experiment_id)
    for run in runs:
        print(f"Run ID: {run.info.run_id}")
        print(f"Artifact URI: {run.info.artifact_uri}")
        # List artifacts to verify accessibility
        artifacts = client.list_artifacts(run.info.run_id)
        print(f"Artifacts: {[a.path for a in artifacts]}")
```

## Important Considerations

### Artifact Accessibility

For artifacts to remain viewable after migration:

1. **S3 Permissions**: Both SageMaker MLflow instances must have IAM roles with S3 read permissions for the artifact bucket
2. **Same Bucket**: Artifacts should remain in the same S3 bucket that both instances can access
3. **Bucket Policy**: Ensure the S3 bucket policy allows both MLflow instances to read objects

### What Gets Migrated

✅ **Migrated**:
- Experiment metadata (name, tags, description)
- Run metadata (parameters, metrics, tags)
- Run status and timestamps
- Artifact URI references
- Logged model information
- Dataset inputs
- Traces (if MLflow version supports)

❌ **Not Migrated** (when using `skip_download_run_artifacts`):
- Actual artifact files (remain in original S3 location)

### Answers to Common Questions

#### Q1: Will the artifact_uri field have a value in the exported JSON?
**Yes**. The `artifact_uri` is always preserved in the exported `run.json`, even when artifacts are not downloaded.

#### Q2: Will artifacts be viewable in the destination MLflow UI?
**Yes**, if both conditions are met:
- Both MLflow instances have S3 access to the artifact bucket
- The IAM roles/policies grant read permissions

#### Q3: What if I want to copy artifacts to a different bucket?
In that case, do NOT use `--skip-download-run-artifacts`. Instead:
1. Export normally (artifacts will be downloaded)
2. Before importing, update S3 paths or configure destination to use a different bucket
3. Import normally (artifacts will be uploaded to the new location)

## Troubleshooting

### Artifacts Not Visible After Import

**Problem**: Artifacts are not showing up in the destination MLflow UI.

**Solutions**:
1. Verify S3 permissions for the destination instance
2. Check that the artifact_uri in runs points to the correct S3 bucket
3. Confirm the destination MLflow instance's IAM role has `s3:GetObject` permission

### Permission Errors

**Problem**: "Access Denied" errors when viewing artifacts.

**Solutions**:
1. Add S3 read permissions to the destination MLflow instance's IAM role:
   ```json
   {
     "Effect": "Allow",
     "Action": [
       "s3:GetObject",
       "s3:ListBucket"
     ],
     "Resource": [
       "arn:aws:s3:::my-mlflow-bucket/*",
       "arn:aws:s3:::my-mlflow-bucket"
     ]
   }
   ```

2. Verify bucket policy allows access from both accounts (if using different AWS accounts)

## Benefits

1. **Faster Migration**: No need to download/upload large artifact files
2. **Reduced Storage**: No duplicate artifacts stored
3. **Cost Effective**: Minimize data transfer costs
4. **Preserve Lineage**: Original artifact locations are maintained
5. **Simple S3 Management**: Artifacts remain in a single bucket with consistent paths

## Related Documentation

- [MLflow Export Import README](README.md)
- [Single Tools Documentation](README_single.md)
- [Bulk Tools Documentation](README_bulk.md)
- [AWS SageMaker MLflow Documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/mlflow.html)
