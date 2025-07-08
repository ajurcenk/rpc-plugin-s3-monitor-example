# Redpanda Connect Custom Input Component Example

## Introduction

The example illustrates the process of creating custom input components for Redpanda Connect using the Redpanda Connect plugin framework in Python ([Plugins | Redpanda Connect](https://docs.redpanda.com/redpanda-connect/plugins/about/))

In our Redpanda custom input example, we created a component specifically designed for monitoring AWS S3 buckets. The responsibilities of the S3 monitoring component include:

- Periodically check the S3 buckets for new and updated objects.
- Support bookmarking: ability to start working from a previously checkpointed bookmark

## Custom Plugin Demo

1. Install uv for Python, more details: https://docs.astral.sh/uv/guides/install-python/ 
2. Create and activate a Python virtual environment

    ```bash
    uv venv
    source .venv/bin/activate
    ```

3. Install dependencies

    ```bash
    uv sync
    ```

4. Change the Redpanda Connect pipeline file `connect.yaml`
5. Start the pipeline

    ```bash
    redpanda-connect run --rpc-plugins=./s3_monitor_input.yaml ./connect.yaml
    ```

6. Upload the new file to AWS s3 bucket
7. Check the Rdepanda Connect Console output
The input s3 object monitor component sends the event to the output component, as example:

    ```json
    {
        "object_info": {
            "bucket": "my-data-bucket",
            "cache_control": null,
            "content_disposition": null,
            "content_encoding": null,
            "content_type": null,
            "etag": "ad63c572942f03e8ca8df687fee8ec50",
            "expires": null,
            "is_encrypted": false,
            "key": "data.txt",
            "kms_key_id": null,
            "last_modified": "2025-07-08T13:31:28+00:00",
            "metadata": {},
            "owner_display_name": null,
            "owner_id": null,
            "s3_uri": "s3://my-data-bucket/data.txt",
            "server_side_encryption": null,
            "size": 78,
            "size_gb": 7.264316082000732e-8,
            "size_mb": 0.0000743865966796875,
            "storage_class": "STANDARD",
            "version_id": null
        }
    }
    ```

8. Stop the Redpanda Connect pipeline

9. Check the bookmark file: `bookmarks.json`

10. Start the pipeline again when the bookmark file exists. No new events in the output.

    ```bash
    redpanda-connect run --rpc-plugins=./s3_monitor_input.yaml ./connect.yaml
    ```

11. Stop the Redpanda Connect pipeline.

12. Delete bookmark file:

    ```bash
    rm bookmarks.json
    ```

13. Start the pipeline again. We have to see the events again after the bookmark file is deleted.

14. Stop the Redpanda Connect pipeline

## s3 Object Monitor Architecture

### Core classes

- `Bookmark`: Represents a single bookmark.
- `BookmarkManage`: Manages bookmarks
- `AWSKeyObject`: Represents the AWS S3 object 
- `AsyncS3ChangeMonitor`: The main class monitors the S3 object changes and stores/loads bookmarks

```mermaid
sequenceDiagram
    participant Main as monitor_s3_objects
    participant BM as BookmarkManager
    participant Monitor as AsyncS3ChangeMonitor
    participant S3 as S3Client
    participant Queue as asyncio.Queue
    
    Main->>Main: Parse configuration
    Main->>BM: Create BookmarkManager("consumer-001", file_path)
    BM->>BM: _load_bookmarks()
    BM-->>Main: BookmarkManager instance
    
    Main->>BM: get_latest_bookmark_for_topic(bucket_name)
    BM-->>Main: latest_bookmark or None
    
    Main->>Main: Calculate cut_off_time from bookmark or TIME_THRESHOLD_HOURS
    Main->>Queue: Create asyncio.Queue()
    Main->>Monitor: Create AsyncS3ChangeMonitor(bucket_name, AWS_CONFIG, queue, cut_off_time, check_interval_min)
    
    Main->>Monitor: asyncio.create_task(run_monitor())
    
    Monitor->>Monitor: initialize_s3_client()
    Monitor->>S3: boto3.Session(**aws_config).client('s3')
    S3-->>Monitor: s3_client
    Monitor->>S3: head_bucket(Bucket=bucket_name)
    S3-->>Monitor: Success
    
    Monitor->>Monitor: check_for_changes()
    Monitor->>S3: get_paginator('list_objects_v2')
    S3-->>Monitor: paginator
    
    loop For each page
        Monitor->>S3: paginate(Bucket=bucket_name)
        S3-->>Monitor: page with Contents
        
        loop For each object
            Monitor->>Monitor: Check if last_modified > cut_off_time
            alt Object is recent
                Monitor->>Monitor: AWSKeyObject.from_boto3_response(obj, bucket)
                Monitor->>Monitor: Add to recent_changes list
            end
        end
    end
    
    Monitor->>Monitor: Sort recent_changes by last_modified
    Monitor->>Queue: put(recent_changes)
    Monitor->>Monitor: Update last_check_time
    
    Main->>Queue: get() - Wait for changes
    Queue-->>Main: recent_changes list
    
    loop For each object change event
        Main->>Main: Create Message(payload={"object_info": obj_change_event})
        Main->>Main: yield Message
        Main->>BM: set_bookmark(bucket_name, obj_key, 0)
        BM->>BM: Create new Bookmark
        BM->>BM: _save_bookmarks()
    end
    
    Main->>Queue: task_done()
    
    Note over Monitor: Periodic monitoring continues
    loop Every check_interval_min minutes
        Monitor->>Monitor: asyncio.sleep(check_interval_min * 60)
        Monitor->>Monitor: check_for_changes()
        Monitor->>Queue: put(new_changes)
    end
```