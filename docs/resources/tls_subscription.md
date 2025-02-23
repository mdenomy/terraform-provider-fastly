---
layout: "fastly"
page_title: "Fastly: tls_subscription"
sidebar_current: "docs-fastly-resource-tls_subscription"
description: |-
Enables TLS on a domain using a managed certificate
---

# fastly_tls_subscription

Enables TLS on a domain using a certificate managed by Fastly.

DNS records need to be modified on the domain being secured, in order to respond to the ACME domain ownership challenge.

There are two options for doing this: the `managed_dns_challenges`, which is the default method; and the `managed_http_challenges`, which points production traffic to Fastly.

~> See the [Fastly documentation](https://docs.fastly.com/en/guides/serving-https-traffic-using-fastly-managed-certificates#verifying-domain-ownership) for more information on verifying domain ownership.

The example below demonstrates usage with AWS Route53 to configure DNS, and the `fastly_tls_subscription_validation` resource to wait for validation to complete.

## Example Usage

Basic usage:

```terraform
resource "fastly_service_vcl" "example" {
  name = "example-service"

  domain {
    name = "example.com"
  }

  backend {
    address = "127.0.0.1"
    name    = "localhost"
  }

  force_destroy = true
}

resource "fastly_tls_subscription" "example" {
  domains = [for domain in fastly_service_vcl.example.domain : domain.name]
  certificate_authority = "lets-encrypt"
}
```

Usage with AWS Route 53:

```terraform
locals {
  domains = [
    "example.com",
    "*.example.com",
  ]
  aws_route53_zone_id = "your_route53_zone_id"
}

resource "fastly_service_vcl" "example" {
  name = "example-service"

  dynamic domain {
    for_each = local.domains
    content {
      name = domain.value
    }
  }

  backend {
    address = "127.0.0.1"
    name    = "localhost"
  }

  force_destroy = true
}

resource "fastly_tls_subscription" "example" {
  domains               = [for domain in fastly_service_vcl.example.domain : domain.name]
  certificate_authority = "lets-encrypt"
}

# Set up DNS record for managed DNS domain validation method
resource "aws_route53_record" "domain_validation" {
  depends_on = [fastly_tls_subscription.example]

  # NOTE: in this example, two domains are added to the cert ("example.com" and "*.example.com").
  # The "managed_dns_challenges" read-only attribute only includes one object
  # for "_acme-challenge.example.com" as the challenge record is common in these two domains.
  # 
  # In order to process a cert containing wildcard entries, remove wildcard prefix "*." from the key
  # and use ellipsis (...) to group results by key to avoid "Duplicate object key" error.
  # Therefore, a key may have multiple elements. For example, domains "example.com" and "*.example.com"
  # find the exact same object in the "managed_dns_challenges" attribute due to the "if" statement below.
  # 
  # A simplified version of this complex "for_each" usage would be:
  # ```
  # for_each = {
  #   for challenge in fastly_tls_subscription.example.managed_dns_challenges :
  #   trimprefix(challenge.record_name, "_acme-challenge.") => challenge
  # }
  # ```
  # but since the "managed_dns_challenges" attribute is only known after apply,
  # you will need to create this resource separately ("-target" option) and may not be ideal.
  for_each = {
    for domain in fastly_tls_subscription.example.domains :
    replace(domain, "*.", "") => element([
      for obj in fastly_tls_subscription.example.managed_dns_challenges :
      obj if obj.record_name == "_acme-challenge.${replace(domain, "*.", "")}"
    ], 0)...
  }
  
  # only reads the first element in the list since all elements are exactly the same (see above)
  name            = each.value[0].record_name
  type            = each.value[0].record_type
  zone_id         = local.aws_route53_zone_id
  allow_overwrite = true
  records         = [each.value[0].record_value]
  ttl             = 60
}

# Resource that other resources can depend on if they require the certificate to be issued
resource "fastly_tls_subscription_validation" "example" {
  subscription_id = fastly_tls_subscription.example.id
  depends_on      = [aws_route53_record.domain_validation]
}
```

## Argument Reference

The following arguments are supported:

* `domains` - (Required) List of domains on which to enable TLS.
* `certificate_authority` - (Required) The entity that issues and certifies the TLS certificates for your subscription. Valid values are `lets-encrypt` or `globalsign`.
* `configuration_id` - (Optional) The ID of the set of TLS configuration options that apply to the enabled domains on this subscription.
* `force_update` - (Optional) Always update subscription, even when active domains are present. Defaults to false.
* `force_destroy` - (Optional) Always delete subscription, even when active domains are present. Defaults to false.

!> **Warning:** by default, the Fastly API protects you from disabling production traffic by preventing updating or deleting subscriptions with active domains. The use of `force_update` and `force_destroy` will override these protections. Take extra care using these options if you are handling production traffic.

## Attributes Reference

In addition to the arguments listed above, the following attributes are exported:

* `created_at` - Timestamp (GMT) when the subscription was created.
* `updated_at` - Timestamp (GMT) when the subscription was last updated.
* `state` - The current state of the subscription. The list of possible states are: `pending`, `processing`, `issued`, and `renewing`.
* `managed_dns_challenges` - A list of options for configuring DNS to respond to ACME DNS challenge in order to verify domain ownership. See Managed DNS Challenge below for details.
* `managed_http_challenges` - A list of options for configuring DNS to respond to ACME HTTP challenge in order to verify domain ownership. See Managed HTTP Challenges below for details.

### Managed DNS Challenge

The available attributes in the `managed_dns_challenges` block are:

* `record_name` - The name of the DNS record to add. For example `_acme-challenge.example.com`. Accessed like this, `fastly_tls_subscription.tls.managed_dns_challenges.record_name`.
* `record_type` - The type of DNS record to add, e.g. `A`, or `CNAME`.
* `record_value` - The value to which the DNS record should point, e.g. `xxxxx.fastly-validations.com`.

### Managed HTTP Challenges

The `managed_http_challenges` attribute is a set of different records that could be added depending on requirements.
For example, whether you are adding TLS to an apex domain, or a subdomain will determine which record you require.
Please note that these records will redirect production traffic to Fastly, so make sure the service is configured correctly first.
Each record in the set has the following attributes:

* `record_name` - The name of the DNS record to add. For example `example.com`. Best accessed through a `for` expression to filter the relevant record.
* `record_type` - The type of DNS record to add, e.g. `A`, or `CNAME`.
* `record_values` - A list with the value(s) to which the DNS record should point.

## Import

A subscription can be imported using its Fastly subscription ID, e.g.

```sh
$ terraform import fastly_tls_subscription.demo xxxxxxxxxxx
```

<!-- schema generated by tfplugindocs -->
## Schema

### Required

- **certificate_authority** (String) The entity that issues and certifies the TLS certificates for your subscription. Valid values are `lets-encrypt` or `globalsign`.
- **domains** (Set of String) List of domains on which to enable TLS.

### Optional

- **common_name** (String) The common name associated with the subscription generated by Fastly TLS. If you do not pass a common name on create, we will default to the first TLS domain included. If provided, the domain chosen as the common name must be included in TLS domains.
- **configuration_id** (String) The ID of the set of TLS configuration options that apply to the enabled domains on this subscription.
- **force_destroy** (Boolean) Force delete the subscription even if it has active domains. Warning: this can disable production traffic if used incorrectly. Defaults to false.
- **force_update** (Boolean) Force update the subscription even if it has active domains. Warning: this can disable production traffic if used incorrectly.
- **id** (String) The ID of this resource.

### Read-Only

- **certificate_id** (String) The certificate ID associated with the subscription.
- **created_at** (String) Timestamp (GMT) when the subscription was created.
- **managed_dns_challenge** (Map of String, Deprecated) The details required to configure DNS to respond to ACME DNS challenge in order to verify domain ownership.
- **managed_dns_challenges** (Set of Object) A list of options for configuring DNS to respond to ACME DNS challenge in order to verify domain ownership. (see [below for nested schema](#nestedatt--managed_dns_challenges))
- **managed_http_challenges** (Set of Object) A list of options for configuring DNS to respond to ACME HTTP challenge in order to verify domain ownership. Best accessed through a `for` expression to filter the relevant record. (see [below for nested schema](#nestedatt--managed_http_challenges))
- **state** (String) The current state of the subscription. The list of possible states are: `pending`, `processing`, `issued`, and `renewing`.
- **updated_at** (String) Timestamp (GMT) when the subscription was updated.

<a id="nestedatt--managed_dns_challenges"></a>
### Nested Schema for `managed_dns_challenges`

Read-Only:

- **record_name** (String)
- **record_type** (String)
- **record_value** (String)


<a id="nestedatt--managed_http_challenges"></a>
### Nested Schema for `managed_http_challenges`

Read-Only:

- **record_name** (String)
- **record_type** (String)
- **record_values** (Set of String)