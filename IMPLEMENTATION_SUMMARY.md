# Implementation Summary - SageMaker MLflow Artifact Path Preservation

## Japanese Summary (日本語要約)

### 実装内容
SageMaker MLflow間での実験とランの移行時に、S3のアーティファクトをダウンロードせずにパスを保持する機能を実装しました。

### 主な変更点
1. **`--skip-download-run-artifacts`オプションを追加**
   - `export-experiment`コマンドで使用可能
   - `export-experiments`コマンド（バルクエクスポート）で使用可能

2. **artifact_uriの保持**
   - アーティファクトをスキップしてもartifact_uriはJSONに保存されます
   - インポート後も元のS3パスが維持されます

3. **アーティファクトの閲覧可能性**
   - 両方のMLflowインスタンスがS3にアクセスできればアーティファクトは閲覧可能です

### 使用方法

**単一の実験をエクスポート:**
```bash
export-experiment \
  --experiment my-experiment \
  --output-dir /tmp/export \
  --skip-download-run-artifacts True
```

**複数の実験をエクスポート:**
```bash
export-experiments \
  --experiments all \
  --output-dir /tmp/export \
  --skip-download-run-artifacts True
```

**インポート:**
```bash
import-experiment \
  --experiment-name imported-experiment \
  --input-dir /tmp/export
```

### 質問への回答

1. **skip_download_artifactのようなオプションがあるか？**
   - はい、`--skip-download-run-artifacts`オプションを追加しました

2. **移行後のBのrun画面でもartifactが閲覧できるか？**
   - はい、両方のMLflowインスタンスが同じS3バケットにアクセスできる場合、閲覧可能です

3. **exportされたrunのjsonのartifact_uriに値は入るか？**
   - はい、`artifact_uri`は常にJSONに保存されます

### 詳細なドキュメント
- [SageMaker移行ガイド](README_sagemaker_migration.md) - 詳細な手順とトラブルシューティング
- [シングルツールドキュメント](README_single.md) - 単一オブジェクトのエクスポート/インポート
- [バルクツールドキュメント](README_bulk.md) - 複数オブジェクトのエクスポート/インポート

---

## English Summary

### Implementation Overview
Implemented functionality to migrate experiments and runs between SageMaker MLflow instances while preserving S3 artifact paths without downloading artifact files.

### Key Changes
1. **Added `--skip-download-run-artifacts` option**
   - Available in `export-experiment` command
   - Available in `export-experiments` command (bulk export)

2. **artifact_uri Preservation**
   - The `artifact_uri` field is always saved in the exported JSON, even when artifacts are skipped
   - Original S3 paths are maintained after import

3. **Artifact Accessibility**
   - Artifacts remain viewable if both MLflow instances have S3 access

### Usage

**Export a single experiment:**
```bash
export-experiment \
  --experiment my-experiment \
  --output-dir /tmp/export \
  --skip-download-run-artifacts True
```

**Export multiple experiments:**
```bash
export-experiments \
  --experiments all \
  --output-dir /tmp/export \
  --skip-download-run-artifacts True
```

**Import:**
```bash
import-experiment \
  --experiment-name imported-experiment \
  --input-dir /tmp/export
```

### Answers to Original Questions

1. **Is there an option like skip_download_artifact?**
   - Yes, we added the `--skip-download-run-artifacts` option

2. **Will artifacts be viewable in destination MLflow UI after migration?**
   - Yes, if both MLflow instances have access to the same S3 bucket

3. **Will the artifact_uri field have a value in the exported run JSON?**
   - Yes, the `artifact_uri` is always preserved in the exported JSON

### Detailed Documentation
- [SageMaker Migration Guide](README_sagemaker_migration.md) - Step-by-step guide with troubleshooting
- [Single Tools Documentation](README_single.md) - Single object export/import
- [Bulk Tools Documentation](README_bulk.md) - Bulk object export/import

---

## Technical Details

### Files Modified
- `mlflow_export_import/common/click_options.py` - Added CLI option decorator
- `mlflow_export_import/experiment/export_experiment.py` - Added parameter to function and CLI
- `mlflow_export_import/bulk/export_experiments.py` - Added parameter to bulk function and CLI
- `tests/open_source/test_experiments.py` - Added test for the new feature
- `README.md`, `README_single.md`, `README_bulk.md` - Updated documentation
- `README_sagemaker_migration.md` - New comprehensive guide (created)
- `.gitignore` - Updated to exclude build artifacts

### Testing
- ✅ Manual validation script confirmed functionality
- ✅ Unit test added and syntax verified
- ✅ Code review passed with no issues
- ✅ Security scan (CodeQL) passed with no alerts
- ✅ CLI help text verified

### Benefits
- **Performance**: Significantly faster migrations (no artifact download/upload)
- **Cost**: Reduced data transfer costs
- **Storage**: No duplicate artifact storage needed
- **Simplicity**: Artifacts remain in single location with consistent paths
- **Flexibility**: Works with any shared storage accessible by both instances

### Requirements
- Both MLflow instances must have IAM permissions to access the S3 bucket
- S3 bucket must allow both instances to read objects
- Same S3 bucket should be accessible from both instances

### Migration Example (SageMaker to SageMaker)

```bash
# Step 1: Set source tracking URI
export MLFLOW_TRACKING_URI=<source-sagemaker-mlflow-url>

# Step 2: Export from source
export-experiments \
  --experiments all \
  --output-dir /tmp/migration \
  --skip-download-run-artifacts True

# Step 3: Set destination tracking URI  
export MLFLOW_TRACKING_URI=<destination-sagemaker-mlflow-url>

# Step 4: Import to destination
import-experiments \
  --input-dir /tmp/migration
```

That's it! Your experiments, runs, and artifact references are now migrated, with artifacts remaining in the original S3 location.
