---
title: Kubernetes Egress and Public IPs
tags:
  - kubernetes
  - aws
  - networking
  - integrations
---

A Kubernetes Pod does not necessarily reach the Internet using an IP address that uniquely identifies that Pod.

The public IP seen by an external service depends on the network path used for **egress traffic**.

A useful mental model is:

```text
Pod
 │
 ▼
Kubernetes / VPC network
 │
 ▼
Subnet
 │
 ▼
Routing
 │
 ▼
Egress / NAT mechanism
 │
 ▼
Internet
```

The important idea is:

> The IP address of the workload and the public IP address seen by an external system are different concepts.

## What made this concrete for me

While debugging an external integration, I queried a public IP-discovery service from different application Pods running in the same Kubernetes workload.

Different Pods could appear on the Internet using different public IP addresses.

Nothing relevant had changed in the application code.

The difference came from the infrastructure path used by the outbound traffic.

That immediately changed the debugging question from:

> Which IP does this application have?

to:

> Through which network path does traffic from this workload leave the infrastructure?

## Pod IP vs public egress IP

A Pod may have an IP that is meaningful only inside the Kubernetes cluster or VPC.

For example:

```text
Pod
10.x.x.x
```

An external API on the Internet cannot necessarily see that address.

Instead, traffic may leave through infrastructure that performs address translation:

```text
Pod
10.x.x.x
   │
   ▼
private network
   │
   ▼
NAT / egress
   │
   ▼
public IP
   │
   ▼
External API
```

The external service sees the final public address used by the egress path.

It does not need to know anything about:

- the Pod IP;
- the Rails process;
- the Kubernetes namespace;
- the internal node or subnet topology.

## Why different Pods may expose different public IPs

Multiple public egress IPs do not necessarily imply anything unusual at the application level.

Conceptually, two Pods could follow different paths:

```text
Pod A
  │
  ▼
Subnet A
  │
  ▼
Egress A
  │
  ▼
Public IP A
```

and:

```text
Pod B
  │
  ▼
Subnet B
  │
  ▼
Egress B
  │
  ▼
Public IP B
```

The exact mechanism depends on the infrastructure.

Possible factors include:

- which node runs the Pod;
- which subnet is involved;
- route table configuration;
- Availability Zone;
- NAT infrastructure;
- other explicit egress components.

Observing multiple public IPs tells me that multiple egress identities exist.

It does **not**, by itself, tell me exactly which AWS topology produced them.

That must be verified from the actual infrastructure configuration.

## Why this matters for IP allowlisting

Some external providers restrict access using an IP allowlist.

They may configure something conceptually like:

```text
Allowed:
203.0.113.10
```

and reject traffic originating from any other public IP.

If the application can leave through:

```text
203.0.113.10
203.0.113.11
```

allowlisting only the first address creates an intermittent-looking failure:

```text
Pod / path A
    │
    ▼
203.0.113.10
    │
    ▼
allowed
```

while:

```text
Pod / path B
    │
    ▼
203.0.113.11
    │
    ▼
rejected
```

From the application perspective, both requests may be identical.

The failure is caused by the network identity presented to the external system.

## Infrastructure changes can break integrations without code changes

This is one reason external integrations can fail after an infrastructure migration or routing change even when the application itself has not changed.

For example:

```text
Old infrastructure
        │
        ▼
Public IP A
        │
        ▼
Provider allowlist
```

may become:

```text
New infrastructure
      │
      ├──► Public IP B
      │
      └──► Public IP C
```

If the provider still trusts only `Public IP A`, requests start failing.

The application may be functioning perfectly.

The contract that changed was effectively:

> Which network identities are allowed to communicate with the provider?

That makes egress configuration part of the operational behavior of an integration.

## What I would investigate

If a workload unexpectedly started using a new public IP, I would work backwards from the observed behavior.

### 1. Confirm the observation

Run the same outbound IP check from several Pods.

The first question is whether the behavior is:

- consistent;
- Pod-dependent;
- node-dependent;
- intermittent.

### 2. Identify where the Pods are running

Look at:

- nodes;
- Availability Zones;
- subnets.

If Pods using different public IPs correlate with different infrastructure placement, that is useful evidence.

### 3. Inspect routing

Determine which route tables apply to the relevant subnets.

The question becomes:

> Where does traffic destined for the Internet go?

### 4. Identify the egress mechanism

For example:

```text
private subnet
      │
      ▼
route table
      │
      ▼
NAT / egress component
      │
      ▼
public IP
```

### 5. Compare with the provider allowlist

Finally, compare all possible public egress addresses with what the external provider accepts.

This prevents debugging an application problem that is actually a networking contract problem.

## A useful distinction

I keep these identities separate:

```text
APPLICATION IDENTITY
Rails process / workload

        │

KUBERNETES IDENTITY
Pod / namespace / service

        │

PRIVATE NETWORK IDENTITY
Pod or node IP inside the VPC

        │

PUBLIC NETWORK IDENTITY
IP visible to an external provider
```

They answer different questions.

Knowing the Pod IP does not necessarily tell me which public IP a provider sees.

## Mental model

The model I keep is:

```text
Application
    │
    ▼
   Pod
    │
    ▼
Private network
    │
    ▼
Routing
    │
    ▼
Egress
    │
    ▼
Public identity
    │
    ▼
External system
```

When debugging an outbound integration, the important question is not only:

> Can the application reach the provider?

but also:

> What identity does the provider see when that traffic arrives?

## Related concepts

- [[Egress]]
- [[NAT]]
- [[VPC]]
- [[Subnets]]
- [[Routing]]
- [[Availability Zones]]
- [[Kubernetes Networking]]
- [[IP Allowlisting]]
- [[External Integrations]]

---

This note came from debugging an integration where Pods from the same application workload could reach an external service through different public IP addresses. The observation was clear; the exact infrastructure topology responsible for choosing each egress path had to be verified separately rather than inferred from the IPs alone.