---
layout: default
title: Multi-tenancy configuration
parent: OpenSearch Dashboards multi-tenancy
nav_order: 145
---


# Multi-tenancy configuration

Multi-tenancy is enabled in OpenSearch Dashboards by default. If you need to disable or change settings related to multi-tenancy, see the `kibana` settings in `config/opensearch-security/config.yml`, as shown in the following example:

```yml
config:
  dynamic:
    kibana:
      multitenancy_enabled: true
      private_tenant_enabled: true
      default_tenant: global tenant
      server_username: kibanaserver
      index: '.kibana'
    do_not_fail_on_forbidden: false
```

| Setting | Description |
| :--- | :--- |
| `multitenancy_enabled` | Enable or disable multi-tenancy. Default is `true`. |
| `private_tenant_enabled` | Enable or disable the private tenant. Default is `true`. |
| `default_tenant` | Use to set the tenant that is available when users log in. |
| `server_username` | Must match the name of the OpenSearch Dashboards server user in `opensearch_dashboards.yml`. Default is `kibanaserver`. If a different user is configured, then make sure that user is mapped to the `kibana_server` role through the `role_mappings.yml` file in order to give them the appropriate permissions listed in [kibana_server role details]({{site.url}}{{site.baseurl}}/security/multi-tenancy/multi-tenancy-config/#kibana_server-role-details). |
| `index` | Must match the name of the OpenSearch Dashboards index from `opensearch_dashboards.yml`. Default is `.kibana`. |
| `do_not_fail_on_forbidden` | When `true`, the Security plugin removes any content that a user is not allowed to see from the search results. When `false`, the plugin returns a security exception. Default is `false`. |

The `opensearch_dashboards.yml` file includes additional settings:

```yml
opensearch.username: kibanaserver
opensearch.password: kibanaserver
opensearch.requestHeadersAllowlist: ["securitytenant","Authorization"]
opensearch_security.multitenancy.enabled: true
opensearch_security.multitenancy.tenants.enable_global: true
opensearch_security.multitenancy.tenants.enable_private: true
opensearch_security.multitenancy.tenants.preferred: ["Private", "Global"]
opensearch_security.multitenancy.enable_filter: false
```

| Setting | Description |
| :--- | :--- |
| `opensearch.requestHeadersAllowlist` | OpenSearch Dashboards requires that you add all HTTP headers to the allow list so that the headers pass to OpenSearch. Multi-tenancy uses a specific header, `securitytenant`, that must be present with the standard `Authorization` header. If the `securitytenant` header is not on the allow list, OpenSearch Dashboards starts with a red status.
| `opensearch_security.multitenancy.enabled` | Enables or disables multi-tenancy in OpenSearch Dashboards. Default is `true`. |
| `opensearch_security.multitenancy.tenants.enable_global` | Enables or disables the global tenant. Default is `true`. |
| `opensearch_security.multitenancy.tenants.enable_private` | Enables or disables private tenants. Default is `true`. |
| `opensearch_security.multitenancy.tenants.preferred` | Lets you change ordering in the **Tenants** tab of OpenSearch Dashboards. By default, the list starts with Global and Private (if enabled) and then proceeds alphabetically. You can add tenants here to move them to the top of the list. |
| `opensearch_security.multitenancy.enable_filter` | If you have many tenants, you can add a search bar to the top of the list. Default is `false`. |


## Add tenants

To create tenants, use OpenSearch Dashboards, the REST API, or `tenants.yml`.


#### OpenSearch Dashboards

1. Open OpenSearch Dashboards.
1. Choose **Security**, **Tenants**, and **Create tenant**.
1. Give the tenant a name and description.
1. Choose **Create**.


#### REST API

See [Create tenant]({{site.url}}{{site.baseurl}}/security/access-control/api/#create-tenant).


#### tenants.yml

```yml
---
_meta:
  type: "tenants"
  config_version: 2

## Demo tenants
admin_tenant:
  reserved: false
  description: "Demo tenant for admin user"
```

## Give roles access to tenants

After creating a tenant, give a role access to it using OpenSearch Dashboards, the REST API, or `roles.yml`.

- Read-write (`kibana_all_write`) permissions let the role view and modify objects in the tenant.
- Read-only (`kibana_all_read`) permissions let the role view objects, but not modify them.


#### OpenSearch Dashboards

1. Open OpenSearch Dashboards.
1. Choose **Security**, **Roles**, and a role.
1. For **Tenant permissions**, add tenants, press Enter, and give the role read and/or write permissions to it.


#### REST API

See [Create role]({{site.url}}{{site.baseurl}}/security/access-control/api/#create-role).


#### roles.yml

```yml
---
test-role:
  reserved: false
  hidden: false
  cluster_permissions:
  - "cluster_composite_ops"
  - "indices_monitor"
  index_permissions:
  - index_patterns:
    - "movies*"
    dls: ""
    fls: []
    masked_fields: []
    allowed_actions:
    - "read"
  tenant_permissions:
  - tenant_patterns:
    - "human_resources"
    allowed_actions:
    - "kibana_all_read"
  static: false
_meta:
  type: "roles"
  config_version: 2
```


## Manage OpenSearch Dashboards indexes

The open source version of OpenSearch Dashboards saves all objects to a single index: `.kibana`. The Security plugin uses this index for the global tenant, but separate indexes for every other tenant. Each user also has a private tenant, so you might see a large number of indexes that follow two patterns:

```
.kibana_<hash>_<tenant_name>
.kibana_<hash>_<username>
```

The Security plugin scrubs these index names of special characters, so they might not be a perfect match of tenant names and usernames.
{: .tip }

To back up your OpenSearch Dashboards data, [take a snapshot]({{site.url}}{{site.baseurl}}/opensearch/snapshots/snapshot-restore/) of all tenant indexes using an index pattern such as `.kibana*`.

## Tenant isolation and the `kibana_user` role

The built-in `kibana_user` role grants index-level permissions (`read`, `index`, `delete`, `manage`) on the `.kibana_*` index pattern. This broad pattern matches **all** tenant indexes, not just the indexes belonging to the user who is assigned the role.

Tenant access control is enforced by the Security plugin's privileges interceptor, which rewrites requests through the `.kibana` alias and checks the `securitytenant` header to determine whether a user has access to the requested tenant. The `tenant_permissions` section of a role definition controls which tenants a user can access and with what level of access (read or read/write).

However, because the `kibana_user` role grants direct index-level access to `.kibana_*`, a user who knows the concrete index name of another tenant (for example, `.kibana_<hash>_<tenant_name>`) may be able to read from that index using operations that do not go through the tenant interceptor.
{: .warning }

### Recommendations

If you require strict tenant isolation, consider the following:

- **Use `tenant_permissions`** in your role definitions to control tenant access. The `tenant_permissions` section is the intended mechanism for granting tenant-level access.
- **Avoid assigning `kibana_user` directly** when you need fine-grained tenant isolation. Instead, create custom roles that grant `.kibana_*` index access only to the specific tenant indexes the user needs, or rely solely on `tenant_permissions` with the standard Dashboards UI workflow (which always sends the `securitytenant` header).
- **Restrict API access** for users who should only interact with tenants through the Dashboards UI. Users with direct API access (for example, via Dev Tools or curl) can construct requests that reference concrete tenant indexes.

For related discussion on narrowing the scope of the `kibana_user` role, see [GitHub issue #5349](https://github.com/opensearch-project/security/issues/5349) and [GitHub issue #6206](https://github.com/opensearch-project/security/issues/6206).
{: .note }

## `kibana_server` role details

OpenSearch Dashboards uses the`kibana_server` role to perform necessary OpenSearch operations. By default, `kibanauser` is mapped to this role through the `role_mappings.yml` file. You can view the full list of permissions assigned to this role by sending a GET request to the `_plugins/_security/api/roles/kibana_server` API (include the admin certificate, key, and certificate authority file in the GET request).
The following list includes the permissions assigned to this role:

```
{
  "kibana_server" : {
    "reserved" : true,
    "hidden" : false,
    "description" : "Provide the minimum permissions for the Kibana server",
    "cluster_permissions" : [
      "cluster_monitor",
      "cluster_composite_ops",
      "manage_point_in_time",
      "indices:admin/template*",
      "indices:admin/index_template*",
      "indices:data/read/scroll*"
    ],
    "index_permissions" : [
      {
        "index_patterns" : [
          ".kibana",
          ".opensearch_dashboards"
        ],
        "fls" : [ ],
        "masked_fields" : [ ],
        "allowed_actions" : [
          "indices_all"
        ]
      },
      {
        "index_patterns" : [
          ".kibana-6",
          ".opensearch_dashboards-6"
        ],
        "fls" : [ ],
        "masked_fields" : [ ],
        "allowed_actions" : [
          "indices_all"
        ]
      },
      {
        "index_patterns" : [
          ".kibana_*",
          ".opensearch_dashboards_*"
        ],
        "fls" : [ ],
        "masked_fields" : [ ],
        "allowed_actions" : [
          "indices_all"
        ]
      },
      {
        "index_patterns" : [
          ".tasks"
        ],
        "fls" : [ ],
        "masked_fields" : [ ],
        "allowed_actions" : [
          "indices_all"
        ]
      },
      {
        "index_patterns" : [
          ".management-beats*"
        ],
        "fls" : [ ],
        "masked_fields" : [ ],
        "allowed_actions" : [
          "indices_all"
        ]
      },
      {
        "index_patterns" : [
          "*"
        ],
        "fls" : [ ],
        "masked_fields" : [ ],
        "allowed_actions" : [
          "indices:admin/aliases*"
        ]
      }
    ],
    "tenant_permissions" : [ ],
    "static" : true
  }
}
```
