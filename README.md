# Elasticsearch index lifecycle

This are a group of policies and templates using [Index State Management](https://opendistro.github.io/for-elasticsearch-docs/docs/ism/) of [Open Distro for Elasticsearch](https://opendistro.github.io/for-elasticsearch-docs/)

* policies
* templates

## Start Elasticsearch Cluster

__Note:__ `docker-compose.yml` file came from [SAMPLE DOCKER COMPOSE FILE](https://opendistro.github.io/for-elasticsearch-docs/docs/install/docker/)


```bash
docker-compose down -v
docker-compose up
```

## Kibana dev tool commands

### Case 1

```
# ISM Policy delete_after_15d
PUT _opendistro/_ism/policies/delete_after_15d
{
    "policy": {
        "policy_id": "delete_after_15d",
        "description": "Maintains the indices open by 2 days, then closes those and delete indices after 15 days",
        "default_state": "ReadWrite",
        "schema_version": 1,
        "states": [
            {
                "name": "ReadWrite",
                "actions": [
                    {
                        "read_write": {}
                    }
                ],
                "transitions": [
                    {
                        "state_name": "ReadOnly",
                        "conditions": {
                            "min_index_age": "2d"
                        }
                    }
                ]
            },
            {
                "name": "ReadOnly",
                "actions": [
                    {
                        "read_only": {}
                    }
                ],
                "transitions": [
                    {
                        "state_name": "Delete",
                        "conditions": {
                            "min_index_age": "13d"
                        }
                    }
                ]
            },
            {
                "name": "Delete",
                "actions": [
                    {
                        "delete": {}
                    }
                ]
            }
        ]
    }
}

# Template sample-logs to apply the ISM Policy delete_after_15d to new indices
PUT _template/sample-logs
{
    "index_patterns": [
        "sample-logs-*"
    ],
    "settings": {
        "index.opendistro.index_state_management.policy_id": "delete_after_15d"
    }
}

# Change the oldest indices definition to apply the ISM Policy delete_after_15d
PUT sample-logs-2020-*/_settings
{
  "settings": {
    "index.opendistro.index_state_management.policy_id": "delete_after_15d"
  }
}

# Bulk load sample
POST _bulk
{"index": { "_index": "sample-logs-2020-04-26"}}
{"message": "This is a log sample 1", "@timestamp": "2020-04-26T11:07:00+0000"}
{"index": { "_index": "sample-logs-2020-04-26"}}
{"message": "This is a log sample 2", "@timestamp": "2020-04-26T11:08:00+0000"}
```

### Case 2

```
# ISM Policy rollover_1d_delete_after_15
PUT _opendistro/_ism/policies/rollover_1d_delete_after_15
{
    "policy": {
        "policy_id": "rollover_1d_delete_after_15",
        "description": "Rollover every 1d, then closes those and delete indices after 15 days",
        "default_state": "Rollover",
        "schema_version": 1,
        "states": [
            {
                "name": "Rollover",
                "actions": [
                    {
                        "rollover": {
                            "min_index_age": "1d"
                        }
                    }
                ],
                "transitions": [
                    {
                        "state_name": "ReadOnly",
                        "conditions": {
                            "min_index_age": "2d"
                        }
                    }
                ]
            },
            {
                "name": "ReadOnly",
                "actions": [
                    {
                        "read_only": {}
                    }
                ],
                "transitions": [
                    {
                        "state_name": "Delete",
                        "conditions": {
                            "min_index_age": "13d"
                        }
                    }
                ]
            },
            {
                "name": "Delete",
                "actions": [
                    {
                        "delete": {}
                    }
                ],
                "transitions": []
            }
        ]
    }
}

# Template sample-logs-rollover to apply the ISM Policy rollover_1d_delete_after_15 to new indices
PUT _template/sample-logs-rollover
{
    "index_patterns": [
        "sample-logs-rollover-*"
    ],
    "settings": {
        "index.opendistro.index_state_management.policy_id": "rollover_1d_delete_after_15",
        "index.opendistro.index_state_management.rollover_alias": "sample-logs-rollover"
    }
}

# Create the first rollover manually (it is necessary) to trigger ISM Policy association
PUT sample-logs-rollover-000001
{
    "aliases": {
        "sample-logs-rollover":{
            "is_write_index": true
        }
    }
}

# Bulk load sample, NOTE: To insert data use the rollover aliases
POST _bulk
{"index": { "_index": "sample-logs-rollover"}}
{"message": "This is a log sample 1", "@timestamp": "2020-04-26T11:07:00+0000"}
{"index": { "_index": "sample-logs-rollover"}}
{"message": "This is a log sample 2", "@timestamp": "2020-04-26T11:08:00+0000"}
```