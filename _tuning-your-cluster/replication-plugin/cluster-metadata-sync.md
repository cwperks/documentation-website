---
layout: default
title: Cluster metadata sync
nav_order: 25
parent: Cross-cluster replication
---

# Cluster metadata sync

Cluster metadata sync replicates composable index templates from a leader cluster to a follower cluster. This ensures that the follower cluster has the templates needed to create new indexes after replication is stopped (for example, during data stream rollover or ISM-managed index creation).

## How it works

When you start cluster metadata sync, a persistent task runs on the follower cluster that periodically polls the leader cluster for composable index templates. Any new or changed templates are applied to the follower's cluster state. The sync runs every 30 seconds.

## Prerequisites

You need to [set up a cross-cluster connection]({{site.url}}{{site.baseurl}}/replication-plugin/get-started/#set-up-a-cross-cluster-connection) between two clusters before you can start cluster metadata sync.

## Start cluster metadata sync

Start syncing index templates from the leader cluster. Send this request to the follower cluster.

#### Request

```json
POST /_plugins/_replication/_cluster_metadata_sync
{
   "leader_alias": "<connection-alias-name>"
}
```

Specify the following options:

Options | Description | Type | Required
:--- | :--- |:--- |:--- |
`leader_alias` | The name of the cross-cluster connection. You define this alias when you [set up a cross-cluster connection]({{site.url}}{{site.baseurl}}/replication-plugin/get-started/#set-up-a-cross-cluster-connection). | `string` | Yes

#### Example response

```json
{
   "acknowledged": true
}
```

## Stop cluster metadata sync

Stop syncing index templates from the leader cluster. Existing templates that have already been synced remain on the follower.

#### Request

```json
DELETE /_plugins/_replication/_cluster_metadata_sync
{
   "leader_alias": "<connection-alias-name>"
}
```

Specify the following options:

Options | Description | Type | Required
:--- | :--- |:--- |:--- |
`leader_alias` | The name of the cross-cluster connection. | `string` | Yes

#### Example response

```json
{
   "acknowledged": true
}
```

## What gets synced

Cluster metadata sync replicates **composable index templates** (`_index_template`). This includes:

- Index patterns
- Template settings (number of shards, replicas, etc.)
- Template mappings
- Template aliases
- Data stream configuration

The following are **not** synced:

- Legacy templates (`_template`)
- Ingest pipelines
- Cluster settings
- Component templates (planned for a future release)

## Example usage

Use cluster metadata sync alongside auto-follow replication to keep both index data and templates in sync:

```bash
# On the follower cluster
# Start auto-follow for index data
curl -XPOST 'https://localhost:9200/_plugins/_replication/_autofollow' -H 'Content-Type: application/json' -d '
{
   "leader_alias": "leader-cluster",
   "name": "replicate-all",
   "pattern": "*"
}'

# Start cluster metadata sync for templates
curl -XPOST 'https://localhost:9200/_plugins/_replication/_cluster_metadata_sync' -H 'Content-Type: application/json' -d '
{
   "leader_alias": "leader-cluster"
}'
```
