---
title: "Cluster API"
description: "The cluster management API endpoint provides HTTP access to Sensu cluster data. Here’s a reference for the cluster API in Sensu Go, including examples for returning the cluster definition, creating a cluster member, and more. Read on for the full reference."
version: "5.15"
product: "Sensu Go"
menu:
  sensu-go-5.15:
    parent: api
---

- [The `/cluster/members` API endpoint](#the-clustermembers-API-endpoint)
  - [`/cluster/members` (GET)](#clustermembers-get)
  - [`/cluster/members` (POST)](#clustermembers-post)
- [The `/cluster/members/:member` API endpoint](#the-clustermembersmember-API-endpoint)
  - [`/cluster/members/:member` (PUT)](#clustermembersmember-put)
  - [`/cluster/members/:member` (DELETE)](#clustermembersmember-delete)
- [The `/cluster/id` API endpoint](#the-clusterid-API-endpoint)
  - [`/cluster/id` (GET)](#clusterid-get)

## The `/cluster/members` API endpoint {#the-clustermembers-API-endpoint}

### `/cluster/members` (GET) {#clustermembers-get}

The `/cluster/members` API endpoint provides HTTP GET access to [Sensu cluster][1] data.

#### EXAMPLE {#clustermembers-get-example}

The following example demonstrates a request to the `/cluster/members` API, resulting in
a JSON Map containing a Sensu cluster definition.

{{< highlight shell >}}
curl -H "Authorization: Bearer $SENSU_TOKEN" \
http://127.0.0.1:8080/api/core/v2/cluster/members

HTTP/1.1 200 OK
{
  "header": {
    "cluster_id": 4255616304056076734,
    "member_id": 9882886658148554927,
    "raft_term": 2
  },
  "members": [
    {
      "ID": 9882886658148554927,
      "name": "default",
      "peerURLs": [
        "http://127.0.0.1:2380"
      ],
      "clientURLs": [
        "http://127.0.0.1:2379"
      ]
    }
  ]
}
{{< /highlight >}}

#### API Specification {#clustermembers-get-specification}

/cluster/members (GET)  | 
---------------|------
description    | Returns the etcd cluster definition.
example url    | http://hostname:8080/api/core/v2/cluster/members
response type  | Map
response codes | <ul><li>**Success**: 200 (OK)</li><li>**Error**: 500 (Internal Server Error)</li></ul>
example output | {{< highlight shell >}}
{
  "header": {
    "cluster_id": 4255616304056076734,
    "member_id": 9882886658148554927,
    "raft_term": 2
  },
  "members": [
    {
      "ID": 9882886658148554927,
      "name": "default",
      "peerURLs": [
        "http://127.0.0.1:2380"
      ],
      "clientURLs": [
        "http://127.0.0.1:2379"
      ]
    }
  ]
}
{{< /highlight >}}

### `/cluster/members` (POST) {#clustermembers-post}

The `/cluster/members` API endpoint provides HTTP POST access to create a Sensu cluster member.

#### EXAMPLE {#clustermembers-post-example}

{{< highlight shell >}}
curl -X POST \
-H "Authorization: Bearer $SENSU_TOKEN" \
http://127.0.0.1:8080/api/core/v2/cluster/members?peer-addrs=http://127.0.0.1:2380

HTTP/1.1 200 OK
{
  "header": {
    "cluster_id": 4255616304056077000,
    "member_id": 9882886658148555000,
    "raft_term": 2
  },
  "members": [
    {
      "ID": 9882886658148555000,
      "name": "default",
      "peerURLs": [
        "http://127.0.0.1:2380"
      ],
      "clientURLs": [
        "http://localhost:2379"
      ]
    }
  ]
}
{{< /highlight >}}

#### API Specification {#clustermembers-post-specification}

/cluster/members/:member (POST) | 
----------------|------
description     | Creates a cluster member.
example url     | http://hostname:8080/api/core/v2/cluster/members?peer-addrs=http://127.0.0.1:2380
query parameters| <ul><li>`peer-addrs` (required): A comma-delimited list of peer addresses</li></ul>
response codes   | <ul><li>**Success**: 200 (OK)</li><li> **Missing**: 404 (Not Found)</li><li>**Error**: 500 (Internal Server Error)</li></ul>

## The `/cluster/members/:member` API endpoint {#the-clustermembersmember-API-endpoint}

### `/cluster/members/:member` (PUT) {#clustermembersmember-put}

#### EXAMPLE {#clustermembersmember-put-example}

{{< highlight shell >}}
curl -X PUT \
-H "Authorization: Bearer $SENSU_TOKEN" \
http://127.0.0.1:8080/api/core/v2/cluster/members/8927110dc66458af?peer-addrs=http://127.0.0.1:2380

HTTP/1.1 200 OK
{
  "header": {
    "cluster_id": 4255616304056077000,
    "member_id": 9882886658148555000,
    "raft_term": 2
  },
  "members": [
    {
      "ID": 9882886658148555000,
      "name": "default",
      "peerURLs": [
        "http://127.0.0.1:2380"
      ],
      "clientURLs": [
        "http://localhost:2379"
      ]
    }
  ]
}
{{< /highlight >}}

#### API Specification {#clustermembersmember-put-specification}

/cluster/members/:member (PUT) | 
----------------|------
description     | Creates a cluster member.
example url     | http://hostname:8080/api/core/v2/cluster/members/8927110dc66458af?peer-addrs=http://127.0.0.1:2380
url parameters  | <ul><li>`8927110dc66458af` (required): Required hex-encoded uint64 cluster member ID generated using `sensuctl cluster member-list`</li></ul>
query parameters| <ul><li>`peer-addrs` (required): A comma-delimited list of peer addresses</li></ul>
response codes   | <ul><li>**Success**: 200 (OK)</li><li> **Missing**: 404 (Not Found)</li><li>**Error**: 500 (Internal Server Error)</li></ul>

### `/cluster/members/:member` (DELETE) {#clustermembersmember-delete}

The `/cluster/members/:member` API endpoint provides HTTP DELETE access to remove a Sensu cluster member.

### EXAMPLE {#clustermembersmember-delete-example}
The following example shows a request to remove the Sensu cluster member with the ID `8927110dc66458af`, resulting in a successful HTTP 204 No Content response.

{{< highlight shell >}}
curl -X DELETE \
-H "Authorization: Bearer $SENSU_TOKEN" \
http://127.0.0.1:8080/api/core/v2/namespaces/default/cluster/members/8927110dc66458af

HTTP/1.1 204 No Content
{{< /highlight >}}

#### API Specification {#clustermembersmember-delete-specification}

/cluster/ members/:member (DELETE) | 
--------------------------|------
description               | Removes a member from a Sensu cluster given the member ID.
example url               | http://hostname:8080/api/core/v2/cluster/members/8927110dc66458af
url parameters            | <ul><li>`8927110dc66458af` (required): Required hex-encoded uint64 cluster member ID generated using `sensuctl cluster member-list`</li></ul>
response codes            | <ul><li>**Success**: 204 (No Content)</li><li>**Missing**: 404 (Not Found)</li><li>**Error**: 500 (Internal Server Error)</li></ul>

## The `/cluster/id` API endpoint {#the-clusterid-API-endpoint}

### `/cluster/id` (GET) {#clusterid-get}

The `/cluster/id` API endpoint provides HTTP GET access to the Sensu cluster ID.

#### EXAMPLE {#clusterid-get-example}

The following example demonstrates a request to the `/cluster/id` API, resulting in
a string containing the Sensu cluster ID.

{{< highlight shell >}}
curl -H "Authorization: Bearer $SENSU_TOKEN" \
http://127.0.0.1:8080/api/core/v2/cluster/members

HTTP/1.1 200 OK
"23481e76-5844-4d07-b714-6e2ffbbf9315"
{{< /highlight >}}

#### API Specification {#clusterid-get-specification}

/cluster/id (GET) |  |
---------------|------
description    | Returns the unique Sensu cluster ID.
example url    | http://hostname:8080/api/core/v2/cluster/id
response type  | String
response codes | <ul><li>**Success**: 200 (OK)</li><li>**Error**: 500 (Internal Server Error)</li></ul>
example output | {{< highlight shell >}}
"23481e76-5844-4d07-b714-6e2ffbbf9315"
{{< /highlight >}}

[1]: ../../guides/clustering
