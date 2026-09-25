# S1

> [!WARNING]
> **S1 is no longer supported and has been replaced by [hadro](https://github.com/mabel-dev/hadro).**
> This repository is kept for reference only and will not receive updates or fixes.

S1 was a minimal S3-compatible API (GetBucketLocation, ListObjects, GetObject and
S3 Select over Parquet) backed by Google Cloud Storage or the local filesystem.

## Moving to hadro

hadro is S1's successor. It is a pip-installable, read-only S3-compatible server that
also covers ListBuckets, HeadObject, range requests, ListObjectsV2 pagination, optional
SigV4 authentication and a richer S3 Select.

```bash
pip install hadro
hadro ./data        # each sub-directory is a bucket
```

The main differences for existing S1 users:

- Start the server with `hadro` (or `python -m hadro`) instead of `python src/main.py`.
- Environment variables have a `HADRO_` prefix: `STORAGE_BACKEND` → `HADRO_BACKEND`,
  `LOCAL_STORAGE_PATH` → `HADRO_DATA`, `GCS_PROJECT` → `HADRO_GCS_PROJECT`.
  `STORAGE_CACHE_SIZE` is replaced by `HADRO_CACHE_MB`.
- The default backend is `local` rather than `gcs`.

See the [hadro README](https://github.com/mabel-dev/hadro#readme) for full documentation.

> [!CAUTION]
> The last S1 release lets clients read files outside the data directory
> (for example `GET /bucket/..%2F..%2Fetc/passwd` with the local backend).
> This is fixed in hadro. Do not expose S1 to untrusted clients.
