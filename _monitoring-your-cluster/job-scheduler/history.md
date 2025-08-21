---
layout: default
title: History API
parent: Job Scheduler
nav_order: 30
redirect_from:
    - /monitoring-plugins/job-scheduler/api/
---

# Job Scheduler Locks API 
Introduced 3.3
{: .label .label-purple }

The Job Scheduler has a history service that logs job executions. This API provides user access to this information.

## Endpoints

```json
GET /_plugins/_job_scheduler/api/history
GET /_plugins/_job_scheduler/api/history/<history_id>
```

## Path parameters

The following table lists the available path parameters. All path parameters are optional.

| Parameter | Data type | Description |
| :--- | :--- | :--- |
| `<history_id>` | String | A unique identifier for the lock, formatted as `"index"-"job_id"` (for example, `.scheduler_sample_extension-jobid1`). The index name and job ID must be separated by a hyphen (`-`). This will get every entry containing this job.|

## Example request

```json
GET /_plugins/_job_scheduler/api/history
```
{% include copy-curl.html %}

## Example response

```json
{
  "total_history": 1,
  "history": {
    ".scheduler_sample_extension-jobid1-1755814120": {
      "job_index_name": ".scheduler_sample_extension",
      "job_id": "jobid1",
      "start_time": 1755814120,
      "completion_status": 0,
      "end_time": 1755814120
    }
  }
}
```

## Response body fields

The following table lists all response body fields.

| Field | Data type | Description |
| :--- | :--- | :--- |
| `total_history` | Integer | The total number of history entries. |
| `history` | Map | A map of history IDs and their associated history information. |
| `job_index_name` | String | The name of the index in which the job is stored. |
| `job_id` | String | The job ID. |
| `start_time` | Seconds since the epoch | Time at which the job started executing. |
| `completion_status` | Integer | Returns 0 for success and 1 for failure. |
| `rend_time` | Seconds since the epoch | 	Time at which the job stopped executing. |