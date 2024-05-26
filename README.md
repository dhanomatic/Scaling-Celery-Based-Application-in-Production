# Scaling Celery Based Application in Production

This documentation covers how to scale a Celery-based application for document extraction and comparison using FastAPI, Celery, and Redis. The guide includes steps for task splitting, configuring task dependencies, and scaling individual tasks.

## Table of Contents

1. [Introduction](#introduction)
2. [Task Definitions](#task-definitions)
3. [Orchestrating Tasks with Parallel Processing](#orchestrating-tasks-with-parallel-processing)
4. [FastAPI Integration](#fastapi-integration)
5. [Scaling Celery Workers](#scaling-celery-workers)
6. [Using Dedicated Queues for Each Task Type](#using-dedicated-queues-for-each-task-type)
7. [Autoscaling](#autoscaling)
8. [Distributed Task Execution](#distributed-task-execution)
9. [Monitoring and Management](#monitoring-and-management)
10. [Load Balancing and High Availability](#load-balancing-and-high-availability)
11. [Summary](#summary)

## Introduction

This guide provides a detailed explanation of how to scale a Celery-based application that performs document extraction and comparison. It covers breaking down the tasks, orchestrating them for parallel processing, and scaling the application to handle increased loads in a production environment.

## Task Definitions

Define the tasks for fetching, extracting, and comparing documents:

```python
# tasks.py

from celery_config import celery_app
import logging

logger = logging.getLogger(__name__)

@celery_app.task
def fetch_documents_task(blob_path):
    try:
        documents = fetch_documents(blob_path)  # Replace with your actual fetch logic
        return documents  # Assume this returns a list of document paths or contents
    except Exception as e:
        logger.error(f"Error fetching documents: {e}")
        raise

@celery_app.task
def extract_data_task(document):
    try:
        extracted_data = extract_data(document)  # Replace with your actual extraction logic
        return extracted_data
    except Exception as e:
        logger.error(f"Error extracting data: {e}")
        raise

@celery_app.task
def compare_data_task(extracted_data_list):
    try:
        comparison_results = compare_data(extracted_data_list)  # Replace with your actual comparison logic
        return comparison_results
    except Exception as e:
        logger.error(f"Error comparing data: {e}")
        raise
```

## Orchestrating Tasks with Parallel Processing

Use a combination of chains and groups to handle dependencies and parallel processing:

```python
# main.py or workflow.py

from celery import chain, group
from tasks import fetch_documents_task, extract_data_task, compare_data_task

def process_documents(blob_path):
    # Step 1: Fetch documents
    fetch_task = fetch_documents_task.s(blob_path)

    # Step 2: Extract data from each document in parallel
    extract_tasks = fetch_task | group(extract_data_task.s(doc) for doc in fetch_task.get())

    # Step 3: Compare the extracted data
    compare_task = compare_data_task.s()

    # Combine the workflow into a single chain
    workflow = chain(fetch_task, extract_tasks, compare_task)
    result = workflow.apply_async()
    return result
```

## FastAPI Integration

Integrate the workflow with a FastAPI endpoint:

```python
# main.py

from fastapi import FastAPI
from workflow import process_documents  # Import your workflow function
from celery_config import celery_app

app = FastAPI()

@app.post("/process/")
async def process_endpoint(blob_path: str):
    result = process_documents(blob_path)
    return {"task_id": result.id}

@app.get("/status/{task_id}")
async def get_status(task_id: str):
    result = celery_app.AsyncResult(task_id)
    if result.state == 'PENDING':
        return {"status": "Pending..."}
    elif result.state == 'SUCCESS':
        return {"status": "Completed", "result": result.result}
    elif result.state == 'FAILURE':
        return {"status": "Failed", "result": str(result.result)}
    else:
        return {"status": result.state}
```

## Scaling Celery Workers

### Increasing the Number of Workers

Start multiple Celery worker processes:

```bash
celery -A celery_config worker --loglevel=info --concurrency=4
```

To scale further, start more workers:

```bash
celery -A celery_config worker --loglevel=info --concurrency=4
celery -A celery_config worker --loglevel=info --concurrency=4
```

### Distributed Workers

Run workers on different machines by pointing them to the same message broker:

```bash
celery -A celery_config worker --loglevel=info --concurrency=4 -Q fetch_queue
celery -A celery_config worker --loglevel=info --concurrency=8 -Q extract_queue
celery -A celery_config worker --loglevel=info --concurrency=2 -Q compare_queue
```

## Using Dedicated Queues for Each Task Type

### Defining Queues

Configure Celery to define multiple queues:

```python
# celery_config.py

from celery import Celery

celery_app = Celery('tasks', broker='redis://localhost:6379/0', backend='redis://localhost:6379/0')

celery_app.conf.task_queues = (
    Queue('fetch_queue', routing_key='fetch.#'),
    Queue('extract_queue', routing_key='extract.#'),
    Queue('compare_queue', routing_key='compare.#'),
)

celery_app.conf.task_routes = {
    'tasks.fetch_documents_task': {'queue': 'fetch_queue', 'routing_key': 'fetch.documents'},
    'tasks.extract_data_task': {'queue': 'extract_queue', 'routing_key': 'extract.data'},
    'tasks.compare_data_task': {'queue': 'compare_queue', 'routing_key': 'compare.data'},
}
```

### Starting Workers for Specific Queues

```bash
celery -A celery_config worker --loglevel=info --concurrency=4 -Q fetch_queue
celery -A celery_config worker --loglevel=info --concurrency=8 -Q extract_queue
celery -A celery_config worker --loglevel=info --concurrency=2 -Q compare_queue
```

## Autoscaling

Enable autoscaling to dynamically adjust the number of worker processes:

```bash
celery -A celery_config worker --loglevel=info --autoscale=10,3
```

- `--autoscale=10,3`: Scales between 3 and 10 worker processes based on load.

## Distributed Task Execution

Distribute Celery workers across multiple machines:

### Example Setup

1. **Machine 1 (Message Broker and Backend):**
   - Run Redis as your broker and backend.

2. **Machine 2 (Worker Node):**
   - Start Celery workers:
     ```bash
     celery -A celery_config worker --loglevel=info --concurrency=4 -Q fetch_queue
     ```

3. **Machine 3 (Worker Node):**
   - Start Celery workers:
     ```bash
     celery -A celery_config worker --loglevel=info --concurrency=8 -Q extract_queue
     ```

4. **Machine 4 (Worker Node):**
   - Start Celery workers:
     ```bash
     celery -A celery_config worker --loglevel=info --concurrency=2 -Q compare_queue
     ```

## Monitoring and Management

Use monitoring tools like Flower, Prometheus, and Grafana to monitor Celery tasks:

### Flower

Start Flower to monitor Celery workers:

```bash
celery -A celery_config flower
```

## Load Balancing and High Availability

Implement load balancing for high availability and fault tolerance:

### Example Load Balancer Setup

Use HAProxy or another load balancer to distribute requests across multiple Redis instances.

## Summary

- **Scale Workers:** Increase the number of Celery workers to handle more tasks concurrently.
- **Dedicated Queues:** Use different queues for different types of tasks and scale them independently.
- **Autoscaling:** Enable autoscaling to dynamically adjust the number of worker processes based on load.
- **Distributed Execution:** Distribute workers across multiple machines to improve scalability and fault tolerance.
- **Monitoring:** Use monitoring tools to keep track of the performance and health of your Celery workers.
- **Load Balancing:** Implement load balancing for high availability and fault tolerance.

By following these strategies, you can effectively scale your Celery-based application to handle increased loads and ensure reliable task execution in a production environment.

---
