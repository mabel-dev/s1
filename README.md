# S1

Implementing only a little bit of S3.

## Overview

S1 is a lightweight, read-only, S3-compatible API that serves data from Google Cloud Storage (GCS) or the local filesystem. It implements just enough of S3 for S3 clients (such as MinIO's Python SDK, and query engines like Opteryx) to list, read and query objects, with an in-memory LRU cache in front of the storage backend.

### Implemented APIs

| API | Request | Notes |
| --- | --- | --- |
| [GetBucketLocation](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetBucketLocation.html) | `GET /{bucket}?location` | Always returns `eu-west-2` |
| [ListObjects](https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListObjects.html) | `GET /{bucket}` | Prefix filtering and marker-style pagination |
| [GetObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObject.html) | `GET /{bucket}/{key}` | Returns the whole object |
| [SelectObjectContent](https://docs.aws.amazon.com/AmazonS3/latest/API/API_SelectObjectContent.html) (S3 Select) | `POST /{bucket}/{key}?select&select-type=2` | Parquet input only |

Anything else (writes, deletes, multipart uploads, bucket management, etc.) is not implemented. Request signatures are not checked, so any access key and secret key will be accepted.

## Quick start

The repository includes sample data in `data/` (the `astronauts`, `planets` and `tweets` buckets), so you can run S1 against the local filesystem:

```bash
pip install fastapi "uvicorn[standard]" google-cloud-storage pyarrow

STORAGE_BACKEND=local LOCAL_STORAGE_PATH=./data python src/main.py
```

The service listens on port 8080, or the port set in the `PORT` environment variable.

The Makefile has a `run` target that creates a virtual environment in `.venv`, installs the package and starts uvicorn with the local backend pointed at `./data`:

```bash
make run              # STORAGE_BACKEND=local, LOCAL_STORAGE_PATH=./data, PORT=8080
make run PORT=9000    # override any of these on the command line
```

> **Note:** `make run` and `pip install -e .` currently fail because the `google-cloud-storage` and `pyarrow` version ranges in `pyproject.toml` can't be satisfied. Until that's fixed, install the dependencies directly as shown above.

Then try it out:

```bash
curl "http://localhost:8080/astronauts?location"
curl "http://localhost:8080/astronauts"
curl -o astronauts.parquet "http://localhost:8080/astronauts/astronauts.parquet"
```

Or with an S3 client:

```python
from minio import Minio

client = Minio("localhost:8080", access_key="any", secret_key="any", secure=False)
for obj in client.list_objects("astronauts", recursive=True):
    print(obj.object_name)
```

## API details

### GetBucketLocation

```
GET /{bucket}?location
```

Returns a `LocationConstraint` of `eu-west-2` for every bucket.

### ListObjects

```
GET /{bucket}?prefix={prefix}&max-keys={max-keys}&marker={marker}
```

Returns a `ListBucketResult` XML document. Query parameters:

- `prefix` - only return keys that begin with this prefix
- `max-keys` - maximum number of keys to return (default: 1000)
- `marker` - only return keys after this key; `start-after` and `continuation-token` are handled the same way
- `delimiter` - accepted and echoed back in the response, but keys are **not** grouped into `CommonPrefixes`; listings are always recursive

`IsTruncated` is always `false`, and listings are not cached.

### GetObject

```
GET /{bucket}/{key}
```

Returns the whole object as `application/octet-stream`, or `404` if it does not exist. `Range` requests are not supported.

### SelectObjectContent (S3 Select)

```
POST /{bucket}/{key}?select&select-type=2
```

Runs a SQL expression against a **Parquet** object and streams the result back in the AWS event stream format (a `Records` event followed by an `End` event), so standard S3 clients can read the response. CSV and JSON objects can't be queried; fetch them with GetObject instead.

Example request body:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<SelectObjectContentRequest xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <Expression>SELECT name, space_flights FROM S3Object WHERE space_flights > 2</Expression>
    <ExpressionType>SQL</ExpressionType>
    <InputSerialization>
        <Parquet/>
    </InputSerialization>
    <OutputSerialization>
        <JSON/>
    </OutputSerialization>
</SelectObjectContentRequest>
```

Supported:

- **Input serialization**: `Parquet` only; any other format returns `400`
- **Output serialization**:
  - `CSV`, with optional `FieldDelimiter` and `RecordDelimiter`
  - `JSON`, with optional `Type` (`LINES`, the default, or `DOCUMENT`) and `RecordDelimiter`
  - JSON Lines is used if `OutputSerialization` is omitted
- **SQL**: `SELECT <columns | *> FROM S3Object [WHERE <column> <op> <value>]`
  - Columns may be quoted with `"` or `` ` ``
  - A single `WHERE` condition using `=`, `!=`, `<>`, `>`, `<`, `>=` or `<=`
  - Values can be numbers, `true`/`false` or quoted strings, and are compared with integer, float, boolean and string columns

Not supported: `AND`/`OR`, functions, aggregates, `LIMIT`, table aliases, and the `Stats`/`Progress` events.

## Storage backends

S1 reads from one of two backends, selected with `STORAGE_BACKEND`.

### Google Cloud Storage (default)

Buckets and keys map directly to GCS buckets and blobs. The client uses Application Default Credentials. If `STORAGE_EMULATOR_HOST` is set, S1 connects to that GCS emulator with anonymous credentials instead.

### Local filesystem

Objects are read from `{LOCAL_STORAGE_PATH}/{bucket}/{key}`, so each top-level directory is a bucket. This is useful for development and testing.

### Configuration

| Variable | Default | Description |
| --- | --- | --- |
| `STORAGE_BACKEND` | `gcs` | `gcs` or `local` |
| `LOCAL_STORAGE_PATH` | `/data` | Root directory for the local backend |
| `GCS_PROJECT` | `PROJECT` | GCS project name |
| `STORAGE_EMULATOR_HOST` | _unset_ | GCS emulator URL, e.g. `http://localhost:9023` |
| `STORAGE_CACHE_SIZE` | `128` | Maximum number of objects held in the LRU cache |
| `PORT` | `8080` | Port to listen on when started with `python src/main.py` |

See [`examples/storage_backend_config.py`](examples/storage_backend_config.py) for more configuration examples.

### LRU caching

Object content reads (used by GetObject and SelectObjectContent) go through a `functools.lru_cache` that holds up to `STORAGE_CACHE_SIZE` objects in memory. Repeated reads of the same object are served from memory, which makes S1 useful as a caching layer in front of GCS for query engines such as Opteryx.

The cache is never invalidated. If an object changes in the underlying storage, S1 keeps serving the cached version until it is evicted or the process restarts. Missing objects are cached too.

## Project layout

```
src/
  main.py                     FastAPI app and request routing
  services/
    get_bucket_location.py
    list_objects.py
    get_object.py
    select_object_content.py  SQL parsing, filtering and event stream encoding
    storage.py                GCS and local backends, plus the LRU cache
tests/                        Integration tests using the MinIO client
data/                         Sample buckets for local development
examples/                     Configuration examples
```

## Development

Run the tests (they start S1 against `data/` and use the MinIO client):

```bash
pip install fastapi "uvicorn[standard]" google-cloud-storage pyarrow pytest minio
python -m pytest tests
```

Lint and format the code:

```bash
make lint
```
