# Design decisions

## Umbrella Glance Service (Split API deployment)

As mentioned in [OSSN-0090](https://wiki.openstack.org/wiki/OSSN/OSSN-0090),
when deploying Glance in a popular configuration where Glance shares a
common storage backend with Nova and/or Cinder, it is possible to open some
known attack vectors by which malicious data modification can occur. If you
choose to deploy a glance operator with Ceph as a backend then by default you
will get a split API (Internal Vs External Glance API) deployed.

- A ``user facing`` glance-api service, accessible via the Public and Admin
keystone endpoints.
- An ``internal facing only`` service, accessible via
the Internal keystone endpoint.

The user facing service is configured to not expose image locations, namely by
setting the following options in glance-api.conf:

```editorconfig
[DEFAULT]
show_image_direct_url = False
show_multiple_locations = False
```

The internal service, operating on a different port (e.g. 9293), is configured
identically to the public facing service, except for the following:

```editorconfig
[DEFAULT]
show_image_direct_url = True
show_multiple_locations = True
```

OpenStack services that use glance (cinder and nova) is configured to access
it via the new internal service. That way both cinder and nova will have
access to the image location data.


## Enable Per Tenant Quotas

Glance supports resource consumption quotas on tenants through the use of
Keystone’s unified limits functionality. In order to enable this feature, as
per the [official
documentation](https://docs.openstack.org/glance/latest/admin/quotas.html), the
`Glance CRD` exposes the `quotas` structure, where the limits defined in the
documentation can be defined. As the full [Glance example
CR](https://github.com/openstack-k8s-operators/glance-operator/blob/main/config/samples/glance_v1beta1_glance_quota.yaml)
shows, when the following is added in the top level definition, all the
`GlanceAPI` Pods will enable per tenant quotas via `use_keystone_limits =
True` in their config, that will be automatically generated by the `Glance`
operator.

```
apiVersion: glance.openstack.org/v1beta1
kind: Glance
metadata:
  name: glance
spec:
  ...
  ...
  quotas:
    imageSizeTotal: 1000
    imageStageTotal: 1000
    imageCountUpload: 100
    imageCountTotal: 100
```

As a follow up work, tha Glance operator might rely on the existing `webhooks`
to enforce the `Quota` Limits values.
