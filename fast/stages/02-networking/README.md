# Networking

This stage sets up the shared network infrastructure for the whole org. It adopts the common “hub and spoke” reference design, which is well suited to multiple scenarios, and offers several advantages versus other designs:

- the “hub” VPC centralizes external connectivity to on-prem or other cloud environments, and is ready to host cross-environment services like CI/CD, code repositories, and monitoring probes
- the “spoke” VPCs allow partitioning workloads (e.g. by environment like in this setup), while still retaining controlled access to central connectivity and services
- Shared VPC in both hub and spokes splits management of network resources in specific (host) projects, while still allowing them to be consumed from workload (service) projects
- the design also lends itself to easy DNS centralization, both from on-prem to cloud and from cloud to on-prem

Connectivity between hub and spokes is established here via [VPN HA](https://cloud.google.com/network-connectivity/docs/vpn/concepts/topologies) tunnels, which offer easy interoperability with some key GCP features (GKE, services leveraging Service Networking like Cloud SQL, etc.), allowing clear partitioning of quota and limits between environments, and fine-grained control of routing. Different ways of implementing connectivity, and their respective pros and cons, are discussed below.

The following diagram illustrates the high-level design, and should be used as a reference for the following sections. The final number of subnets, and their IP addressing design will of course depend on customer-specific requirements, and can be easily changed via variables or external data files without having to edit the actual code.

![Networking diagram](diagram.png)

## Design overview and choices

### VPC design

The hub/landing VPC hosts external connectivity and shared services for spoke VPCs, which are connected to it via VPN HA tunnels. Spokes are used here to partition environments, which is a fairly common pattern:

- one spoke VPC for the production environment
- one spoke VPC for the development environment

Each VPC is created into its own project, and each project is configured as a Shared VPC host, so that network-related resources and access configurations via IAM are kept separate for each VPC.

The design easily lends itself to implementing additional environments, or adopting a different logical mapping for spokes (e.g. one spoke for each company entity, etc.). Adding spokes is a trivial operation, does not increase the design complexity, and is explained in operational terms in the following sections.

In multi-organization scenarios, where production and non-production resources use different Cloud Identity and GCP organizations, the hub/landing VPC is usually part of the production organization, and establishes connections with production spokes in its same organization, and non-production spokes in a different organization.

An additional VPC is also deployed by default with the provided code, disconnected from the other VPCs and hosting a single VM that emulates on-prem for testing purposes, via a Docker network and containers for VPN, DNS, HTTP, etc.

### External connectivity

External connectivity to on-prem is implemented here via VPN HA (two tunnels per region), as this is the minimum common denominator often used directly, or as a stop-gap solution to validate routing and transfer data, while waiting for [interconnects](https://cloud.google.com/network-connectivity/docs/interconnect) to be provisioned.

Connectivity to additional on-prem sites or other cloud providers should be implemented in a similar fashion, via VPN tunnels or interconnects in the landing VPC sharing the same regional router.

### Internal connectivity

As mentioned initially, there are of course other ways to implement internal connectivity other than VPN HA. These can be easily retrofitted with minimal code changes, but introduce additional considerations for service interoperability, quotas and management.

This is a summary of the main options:

- [VPN HA](https://cloud.google.com/network-connectivity/docs/vpn/concepts/topologies) (implemented here)
  - Pros: simple compatibility with GCP services that leverage peering internally, better control on routes, avoids peering groups shared quotas and limits
  - Cons: additional cost, marginal increase in latency, requires multiple tunnels for full bandwidth
- [VPC Peering](https://cloud.google.com/vpc/docs/vpc-peering)
  - Pros: no additional costs, full bandwidth with no configurations, no extra latency
  - Cons: no transitivity (e.g. to GKE masters, Cloud SQL, etc.), no selective exchange of routes, several quotas and limits shared between VPCs in a peering group
- [Multi-NIC appliances](https://cloud.google.com/architecture/best-practices-vpc-design#multi-nic)
  - Pros: additional security features (e.g. IPS), potentially better integration with on-prem systems by using the same vendor
  - Cons: complex HA/failover setup, limited by VM bandwidth and scale, additional costs for VMs and licenses, out of band management of a critical cloud component

### IP ranges, subnetting, routing

Minimizing the number of routes (and subnets) in use on the cloud environment is an important consideration, as it simplifies management and avoids hitting [Cloud Router](https://cloud.google.com/network-connectivity/docs/router/quotas) and [VPC](https://cloud.google.com/vpc/docs/quota) quotas and limits. For this reason, we recommend careful planning of the IP space used in your cloud environment, to be able to use large IP CIDR blocks in routes whenever possible.

This stage uses a dedicated /16 block (which should of course be sized to your needs) for each region in each VPC, and subnets created in each VPC derive their ranges from the relevant block.

Spoke VPCs also define and reserve two "special" CIDR ranges dedicated to [PSA (Private Service Access)](https://cloud.google.com/vpc/docs/private-services-access) and [Internal HTTPs Load Balancers (L7ILB)](https://cloud.google.com/load-balancing/docs/l7-internal).

Routes in GCP are either automatically created for VPC subnets, manually created via static routes, or dynamically programmed by [Cloud Routers](https://cloud.google.com/network-connectivity/docs/router#docs) via BGP sessions, which can be configured to advertise VPC ranges, and/or custom ranges via custom advertisements.

In this setup, the Cloud Routers are configured so as to exclude the default advertisement of VPC ranges, and they only advertise their respective aggregate ranges via custom advertisements. This greatly simplifies the routing configuration, and more importantly it allows to avoid quota or limit issues by keeping the number of routes small, instead of making it proportional to the subnets and secondary ranges in the VPCs.

The high-level routing plan implemented in this architecture is as follows:

| source      | target      | advertisement                  |
| ----------- | ----------- | ------------------------------ |
| VPC landing | onprem      | GCP aggregate                  |
| VPC landing | onprem      | Cloud DNS forwarders           |
| VPC landing | onprem      | Google private/restricted APIs |
| VPC landing | spokes      | RFC1918                        |
| VPC spoke   | VPC landing | spoke aggregate                |
| onprem      | VC landing  | onprem aggregates              |

As is evident from the table above, the hub/landing VPC acts as the route concentrator for the whole GCP network, implementing a full line of sight between environments, and between GCP and on-prem. While advertisements can be adjusted to selectively exchange routes (e.g. to isolate the production and the development environment), we recommend using [Firewall](#firewall) policies or rules to achieve the desired isolation.

### Internet egress

The path of least resistance for Internet egress is using Cloud NAT, and that is what's implemented in this setup, with a NAT gateway configured for each VPC.

Several other scenarios are possible of course, with varying degrees of complexity:

- a forward proxy, with optional URL filters
- a default route to on-prem to leverage existing egress infrastructure
- a full-fledged perimeter firewall to control egress and implement additional security features like IPS

Future pluggable modules will allow to easily experiment, or deploy the above scenarios.

### Firewall

The GCP Firewall is a stateful, distributed feature that allows the creation of L4 policies, either via VPC-level rules or more recently via hierarchical policies applied on the resource hierarchy (organization, folders).

The current setup adopts both firewall types, and uses hierarchical rules on the Networking folder for common ingress rules (egress is open by default), e.g. from health check or IAP forwarders ranges, and VPC rules for the environment or workload-level ingress.

Rules and policies are defined in simple YAML files, described below.

### DNS

DNS often goes hand in hand with networking, especially on GCP where Cloud DNS zones and policies are associated at the VPC level. This setup implements both DNS flows:

- on-prem to cloud via private zones for cloud-managed domains, and an [inbound policy](https://cloud.google.com/dns/docs/server-policies-overview#dns-server-policy-in) used as forwarding target or via delegation (requires some extra configuration) from on-prem DNS resolvers
- cloud to on-prem via forwarding zones for the on-prem managed domains

DNS configuration is further centralized by leveraging peering zones, so that

- the hub/landing Cloud DNS hosts configurations for on-prem forwarding and Google API domains, with the spokes consuming them via DNS peering zones
- the spokes Cloud DNS host configurations for the environment-specific domains, with the hub/landing VPC acting as consumer via DNS peering

To complete the configuration, the 35.199.192.0/19 range should be routed on the VPN tunnels from on-prem, and the following names configured for DNS forwarding to cloud:

- `private.googleapis.com`
- `restricted.googleapis.com`
- `gcp.example.com` (used as a placeholder)

From cloud, the `example.com` domain (used as a placeholder) is forwarded to on-prem.

This configuration is battle-tested, and flexible enough to lend itself to simple modifications without subverting its design, for example by forwarding and peering root zones to bypass Cloud DNS external resolution.

## How to run this stage

This stage is meant to be executed after the [resman](../01-resman) stage has run, as it leverages the automation service account and bucket created there, and additional resources configured in the [bootstrap](../00-boostrap) stage.

It's of course possible to run this stage in isolation, but that's outside the scope of this document, and you would need to refer to the code for the previous stages for the environmental requirements.

Before running this stage, you need to make sure you have the correct credentials and permissions, and localize variables by assigning values that match your configuration.

### Providers configuration

The default way of making sure you have the right permissions, is to use the identity of the service account pre-created for this stage during the [resource management](./01-resman) stage, and that you are a member of the group that can impersonate it via provider-level configuration (`gcp-devops` or `organization-admins`).

To simplify setup, the previous stage pre-configures a valid providers file in its output, and optionally writes it to a local file if the `outputs_location` variable is set to a valid path.

If you have set a valid value for `outputs_location` in the bootstrap stage, simply link the relevant `providers.tf` file from this stage's folder in the path you specified:

```bash
# `outputs_location` is set to `../../configs/example`
ln -s ../../configs/example/02-networking/providers.tf
```

If you have not configured `outputs_location` in bootstrap, you can derive the providers file from that stage's outputs:

```bash
cd ../00-bootstrap
terraform output -json providers | jq -r '.["02-networking"]' \
  > ../02-networking/providers.tf
```

### Variable configuration

There are two broad sets of variables you will need to fill in:

- variables shared by other stages (org id, billing account id, etc.), or derived from a resource managed by a different stage (folder id, automation project id, etc.)
- variables specific to resources managed by this stage

To avoid the tedious job of filling in the first group of variables with values derived from other stages' outputs, the same mechanism used above for the provider configuration can be used to leverage pre-configured `.tfvars` files.

If you have set a valid value for `outputs_location` in the bootstrap and in the resman stage, simply link the relevant `terraform-*.auto.tfvars.json` files from this stage's folder in the path you specified, where the `*` above is set to the name of the stage that produced it. For this stage, a single `.tfvars` file is available:

```bash
# `outputs_location` is set to `../../configs/example`
ln -s ../../configs/example/02-networking/terraform-bootstrap.auto.tfvars.json
ln -s ../../configs/example/02-networking/terraform-resman.auto.tfvars.json
```

Please refer to the [Variables](#variables) table below for a map of the variable origins, and to the sections below on how to adapt this stage to your networking configuration.

### VPCs

VPCs are defined in separate files, one for `landing` and one for each of `prod` and `dev`.
Each file contains the same resources, described in the following paragraphs.

The **project** ([`project`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/project)) contains the VPC, and enables the required APIs and sets itself as a "[host project](https://cloud.google.com/vpc/docs/shared-vpc)".

The **VPC** ([`net-vpc`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-vpc)) manages the DNS inbound policy (for Landing), explicit routes for `{private,restricted}.googleapis.com`, and its **subnets**. Subnets are created leveraging a "resource factory" paradigm, where the configuration is separated from the module that implements it, and stored in a well-structured file. To add a new subnet, simply create a new file in the `data_folder` directory defined in the module, following the examples found in the [Fabric `net-vpc` documentation](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-vpc#subnet-factory). 
Subnets for [L7 ILBs](https://cloud.google.com/load-balancing/docs/l7-internal/proxy-only-subnets) are defined in variable `l7ilb_subnets`, while ranges for [PSA](https://cloud.google.com/vpc/docs/configure-private-services-access#allocating-range) are in variable `psa_ranges` - such variables are consumed by spoke VPCs.

**Cloud NAT** ([`net-cloudnat`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-cloudnat)) manages the networking infrastructure required to enable internet egress.

### VPNs

#### External

Connectivity to on-prem is implemented with VPN HA ([`net-vpn`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-vpn)) and defined in [`vpn-onprem.tf`](./vpn-onprem.tf). The file provisionally implements a single logical connection between onprem and landing at `europe-west1`, and the relevant parameters for its configuration are found in variable `vpn_onprem_configs`.

#### Internal

VPNs ([`net-vpn`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-vpn)) used to interconnect landing and spokes are managed by `vpn-spoke-*.tf` files, each implementing both sides of the VPN connection. Per-gateway configurations (e.g. BGP advertisements and session ranges) are controlled by variable `vpn_onprem_configs`. VPN gateways and IKE secrets are automatically generated and configured.

### Routing and BGP

Each VPC network ([`net-vpc`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-vpc)) manages a separate routing table, which can define static routes (e.g. to private.googleapis.com) and receives dynamic routes from BGP sessions established with neighbor networks (e.g. landing receives routes from onprem and spokes, and spokes receive RFC1918 from landing).

Static routes are defined in `vpc-*.tf` files, in the `routes` section of each `net-vpc` module. 

BGP sessions for landing-spoke are configured through variable `vpn_spoke_configs`, while the ones for landing-onprem use variable `vpn_onprem_configs`

### Firewall

**VPC firewall rules** ([`net-vpc-firewall`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-vpc-firewall)) are defined per-vpc on each `vpc-*.tf` file and leverage a resource factory to massively create rules. 
To add a new firewall rule, create a new file or edit an existing one in the `data_folder` directory defined in the module `net-vpc-firewall`, following the examples of the "[Rules factory](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/net-vpc-firewall#rules-factory)" section of the module documentation.

**Hierarchical firewall policies** ([`folder`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/folder)) are defined in `main.tf`, and managed through a policy factory implemented by the `folder` module, which applies the defined hierarchical to the `Networking` folder, which contains all the core networking infrastructure. Policies are defined in the `rules_file` file - to define a new one simply use the instructions found on "[Firewall policy factory](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/organization#firewall-policy-factory)".


### DNS

The DNS ([`dns`](https://github.com/terraform-google-modules/cloud-foundation-fabric/tree/master/modules/dns)) infrastructure is defined in [`dns.tf`](dns.tf).

Cloud DNS manages onprem forwarding, the main GCP zone (in this example `gcp.example.com`) and is peered to environment-specific zones (i.e. `dev.gcp.example.com` and `prod.gcp.example.com`).

#### Cloud environment

Per the section above Landing acts as the source of truth for DNS within the Cloud environment. Resources defined in the spoke VPCs consume the Landing DNS infrastructure through DNS peering (e.g. `prod-landing-root-dns-peering`).
Spokes can optionally define private zones (e.g. `prod-dns-private-zone`) - granting visibility to the Landing VPC ensures that the whole cloud environment can query such zones.

#### Cloud to on-prem

Leveraging the forwarding zones defined on Landing (e.g. `onprem-example-dns-forwarding` and `reverse-10-dns-forwarding`), the cloud environment can resolve `in-addr.arpa.` and `onprem.example.com.` using the on-premises DNS infrastructure. Onprem resolvers IPs are set in variable `dns.onprem`.

DNS queries sent to the on-premises infrastructure come from the `35.199.192.0/19` source range, which is only accessible from within a VPC or networks connected to one.

#### On-prem to cloud

The [Inbound DNS Policy](https://cloud.google.com/dns/docs/server-policies-overview#dns-server-policy-in) defined in module `landing-vpc` ([`landing.tf`](./landing.tf)) automatically reserves the first available IP address on each created subnet (typically the third one in a CIDR) to expose the Cloud DNS service so that it can be consumed from outside of GCP. 

### Private Google Access
[Private Google Access](https://cloud.google.com/vpc/docs/private-google-access) (or PGA) enables VMs and on-prem systems to consume Google APIs from within the Google network, and is already fully configured on this environment.

For PGA to work:

* Private Google Access should be enabled on the subnet. \
Subnets created by the `net-vpc` module are PGA-enabled by default.

* 199.36.153.4/30 (`restricted.googleapis.com`) and 199.36.153.8/30 (`private.googleapis.com`) should be routed from on-prem to VPC, and from there to the `default-internet-gateway`. \
Per variable `vpn_onprem_configs` such ranges are advertised to onprem - furthermore every VPC (e.g. see `landing-vpc` in [`landing.tf`](./landing.tf)) has explicit routes set in case the `0.0.0.0/0` route is changed.

* A private DNS zone for `googleapis.com` should be created and configured per [this article](https://cloud.google.com/vpc/docs/configure-private-google-access-hybrid#config-domain, as implemented in module `googleapis-private-zone` in [`dns.tf`](./dns.tf) 

### Preliminar activities 

Before running `terraform apply` on this stage, make sure to adapt all of `variables.tf` to your needs, to update all reference to regions (e.g. `europe-west1` or `ew1`) in the whole directory to match your preferences.

If you're not using FAST, you'll also need to create a `providers.tf` file to configure the GCS backend and the service account to use to run the deployment. 

You're now ready to run `terraform init` and `apply`.

### Post-deployment activities

* On-prem routers should be configured to advertise all relevant CIDRs to the GCP environments. To avoid hitting GCP quotas, we recomment aggregating routes as much as possible.
* On-prem routers should accept BGP sessions from their cloud peers.
* On-prem DNS servers should have forward zones for GCP-managed ones.

## Customizations

### Adding an environment
To create a new environment (e.g. `staging`), a few changes are required.

Create a `vpc-spoke-staging.tf` file by copying `vpc-spoke-prod.tf` file, 
and adapt the new file by replacing the value "prod" with the value "staging".
Running `diff vpc-spoke-dev.tf vpc-spoke-prod.tf` can help to see how environment files differ.

The new VPC requires a set of dedicated CIDRs, one per region, added to variable `custom_adv` (for example as `spoke_staging_ew1` and `spoke_staging_ew4`).
>`custom_adv` is a map that "resolves" CIDR names to actual addresses, and will be used later to configure routing.
>
Variables managing L7 Interal Load Balancers (`l7ilb_subnets`) and Private Service Access (`psa_ranges`) should also be adapted, and subnets and firewall rules for the new spoke should be added as described above.

VPN HA connectivity (see also [VPNs](#vpns)) to `landing` is managed by the `vpn-spoke-*.tf` files.
Copy `vpn-spoke-prod.tf` to `vpn-spoke-staging.tf` - replace "prod" with "staging" where relevant.

VPN configuration also controls BGP advertisements, which requires the following variable changes: 
* `router_configs` to configure the new routers (one per region) created for the `staging` VPC
* `vpn_onprem_configs` to configure the new advertisments to on-premises for the new CIDRs
* `vpn_spoke_configs` to configure the new advertisements to `landing` for the new VPC - new keys (one per region) should be added, such as e.g. `staging-ew1` and `staging-ew4`

DNS configurations are centralised in the `dns.tf` file. Spokes delegate DNS resolution to Landing through DNS peering, and optionally define a private zone (e.g. `staging.gcp.example.com`) which the landing peers to. To configure DNS for a new environment, copy all the `prod-*` modules in the `dns.tf` file to `staging-*`, and update their content accordingly. Don't forget to add a peering zone from Landing to the newly created environment private zone.




<!-- BEGIN TFDOC -->

## Files

| name | description | modules | resources |
|---|---|---|---|
| [dns-dev.tf](./dns-dev.tf) | Development spoke DNS zones and peerings setup. | <code>dns</code> |  |
| [dns-landing.tf](./dns-landing.tf) | Landing DNS zones and peerings setup. | <code>dns</code> |  |
| [dns-prod.tf](./dns-prod.tf) | Production spoke DNS zones and peerings setup. | <code>dns</code> |  |
| [main.tf](./main.tf) | Networking folder and hierarchical policy. | <code>folder</code> |  |
| [monitoring.tf](./monitoring.tf) | Network monitoring dashboards. |  | <code>google_monitoring_dashboard</code> |
| [outputs.tf](./outputs.tf) | Module outputs. |  | <code>local_file</code> |
| [test-resources.tf](./test-resources.tf) | temporary instances for testing | <code>compute-vm</code> |  |
| [variables.tf](./variables.tf) | Module variables. |  |  |
| [vpc-landing.tf](./vpc-landing.tf) | Landing VPC and related resources. | <code>net-cloudnat</code> · <code>net-vpc</code> · <code>net-vpc-firewall</code> · <code>project</code> |  |
| [vpc-spoke-dev.tf](./vpc-spoke-dev.tf) | Dev spoke VPC and related resources. | <code>net-address</code> · <code>net-cloudnat</code> · <code>net-vpc</code> · <code>net-vpc-firewall</code> · <code>project</code> |  |
| [vpc-spoke-prod.tf](./vpc-spoke-prod.tf) | Production spoke VPC and related resources. | <code>net-address</code> · <code>net-cloudnat</code> · <code>net-vpc</code> · <code>net-vpc-firewall</code> · <code>project</code> |  |
| [vpn-onprem.tf](./vpn-onprem.tf) | VPN between landing and onprem. | <code>net-vpn-ha</code> |  |
| [vpn-spoke-dev.tf](./vpn-spoke-dev.tf) | VPN between landing and development spoke. | <code>net-vpn-ha</code> |  |
| [vpn-spoke-prod.tf](./vpn-spoke-prod.tf) | VPN between landing and production spoke. | <code>net-vpn-ha</code> |  |

## Variables

| name | description | type | required | default | producer |
|---|---|:---:|:---:|:---:|:---:|
| billing_account_id | Billing account id. | <code>string</code> | ✓ |  | <code>00-bootstrap</code> |
| organization | Organization details. | <code title="object&#40;&#123;&#10;  domain      &#61; string&#10;  id          &#61; number&#10;  customer_id &#61; string&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> | ✓ |  | <code>00-bootstrap</code> |
| prefix | Prefix used for resources that need unique names. | <code>string</code> | ✓ |  | <code>00-bootstrap</code> |
| custom_adv | Custom advertisement definitions in name => range format. | <code>map&#40;string&#41;</code> |  | <code title="&#123;&#10;  cloud_dns             &#61; &#34;35.199.192.0&#47;19&#34;&#10;  googleapis_private    &#61; &#34;199.36.153.8&#47;30&#34;&#10;  googleapis_restricted &#61; &#34;199.36.153.4&#47;30&#34;&#10;  rfc_1918_10           &#61; &#34;10.0.0.0&#47;8&#34;&#10;  rfc_1918_172          &#61; &#34;172.16.0.0&#47;16&#34;&#10;  rfc_1918_192          &#61; &#34;192.168.0.0&#47;16&#34;&#10;  landing_ew1           &#61; &#34;10.128.0.0&#47;16&#34;&#10;  landing_ew4           &#61; &#34;10.129.0.0&#47;16&#34;&#10;  spoke_prod_ew1        &#61; &#34;10.136.0.0&#47;16&#34;&#10;  spoke_prod_ew4        &#61; &#34;10.137.0.0&#47;16&#34;&#10;  spoke_dev_ew1         &#61; &#34;10.144.0.0&#47;16&#34;&#10;  spoke_dev_ew4         &#61; &#34;10.145.0.0&#47;16&#34;&#10;&#125;">&#123;&#8230;&#125;</code> |  |
| data_dir | Relative path for the folder storing configuration data for network resources. | <code>string</code> |  | <code>&#34;data&#34;</code> |  |
| dns | Onprem DNS resolvers | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code title="&#123;&#10;  onprem &#61; &#91;&#34;10.0.200.3&#34;&#93;&#10;&#125;">&#123;&#8230;&#125;</code> |  |
| folder_id | Folder to be used for the networking resources in folders/nnnnnnnnnnn format. If null, folder will be created. | <code>string</code> |  | <code>null</code> | <code>01-resman</code> |
| gke |  | <code title="map&#40;object&#40;&#123;&#10;  folder_id &#61; string&#10;  sa        &#61; string&#10;  gcs       &#61; string&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#123;&#125;</code> | <code>01-resman</code> |
| l7ilb_subnets | Subnets used for L7 ILBs. | <code title="map&#40;list&#40;object&#40;&#123;&#10;  ip_cidr_range &#61; string&#10;  region        &#61; string&#10;&#125;&#41;&#41;&#41;">map&#40;list&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;&#41;</code> |  | <code title="&#123;&#10;  prod &#61; &#91;&#10;    &#123; ip_cidr_range &#61; &#34;10.136.240.0&#47;24&#34;, region &#61; &#34;europe-west1&#34; &#125;,&#10;    &#123; ip_cidr_range &#61; &#34;10.137.240.0&#47;24&#34;, region &#61; &#34;europe-west4&#34; &#125;&#10;  &#93;&#10;  dev &#61; &#91;&#10;    &#123; ip_cidr_range &#61; &#34;10.144.240.0&#47;24&#34;, region &#61; &#34;europe-west1&#34; &#125;,&#10;    &#123; ip_cidr_range &#61; &#34;10.145.240.0&#47;24&#34;, region &#61; &#34;europe-west4&#34; &#125;&#10;  &#93;&#10;&#125;">&#123;&#8230;&#125;</code> |  |
| outputs_location | Path where providers and tfvars files for the following stages are written. Leave empty to disable. | <code>string</code> |  | <code>null</code> |  |
| project_factory_sa | IAM emails for project factory service accounts | <code>map&#40;string&#41;</code> |  | <code>&#123;&#125;</code> | <code>01-resman</code> |
| psa_ranges | IP ranges used for Private Service Access (e.g. CloudSQL). | <code>map&#40;map&#40;string&#41;&#41;</code> |  | <code title="&#123;&#10;  prod &#61; &#123;&#10;    cloudsql-mysql     &#61; &#34;10.136.250.0&#47;24&#34;&#10;    cloudsql-sqlserver &#61; &#34;10.136.251.0&#47;24&#34;&#10;  &#125;&#10;  dev &#61; &#123;&#10;    cloudsql-mysql     &#61; &#34;10.144.250.0&#47;24&#34;&#10;    cloudsql-sqlserver &#61; &#34;10.144.251.0&#47;24&#34;&#10;  &#125;&#10;&#125;">&#123;&#8230;&#125;</code> |  |
| router_configs | Configurations for CRs and onprem routers. | <code title="map&#40;object&#40;&#123;&#10;  adv &#61; object&#40;&#123;&#10;    custom  &#61; list&#40;string&#41;&#10;    default &#61; bool&#10;  &#125;&#41;&#10;  asn &#61; number&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code title="&#123;&#10;  onprem-ew1 &#61; &#123;&#10;    asn &#61; &#34;65534&#34;&#10;    adv &#61; null&#10;  &#125;&#10;  landing-ew1    &#61; &#123; asn &#61; &#34;64512&#34;, adv &#61; null &#125;&#10;  landing-ew4    &#61; &#123; asn &#61; &#34;64512&#34;, adv &#61; null &#125;&#10;  spoke-dev-ew1  &#61; &#123; asn &#61; &#34;64513&#34;, adv &#61; null &#125;&#10;  spoke-dev-ew4  &#61; &#123; asn &#61; &#34;64513&#34;, adv &#61; null &#125;&#10;  spoke-prod-ew1 &#61; &#123; asn &#61; &#34;64514&#34;, adv &#61; null &#125;&#10;  spoke-prod-ew4 &#61; &#123; asn &#61; &#34;64514&#34;, adv &#61; null &#125;&#10;&#125;">&#123;&#8230;&#125;</code> |  |
| vpn_onprem_configs | VPN gateway configuration for onprem interconnection. | <code title="map&#40;object&#40;&#123;&#10;  adv &#61; object&#40;&#123;&#10;    default &#61; bool&#10;    custom  &#61; list&#40;string&#41;&#10;  &#125;&#41;&#10;  session_range &#61; string&#10;  peer &#61; object&#40;&#123;&#10;    address   &#61; string&#10;    asn       &#61; number&#10;    secret_id &#61; string&#10;  &#125;&#41;&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code title="&#123;&#10;  landing-ew1 &#61; &#123;&#10;    adv &#61; &#123;&#10;      default &#61; false&#10;      custom &#61; &#91;&#10;        &#34;cloud_dns&#34;,&#10;        &#34;googleapis_restricted&#34;,&#10;        &#34;googleapis_private&#34;,&#10;        &#34;landing_ew1&#34;,&#10;        &#34;landing_ew4&#34;,&#10;        &#34;spoke_prod_ew1&#34;,&#10;        &#34;spoke_prod_ew4&#34;,&#10;        &#34;spoke_dev_ew1&#34;,&#10;        &#34;spoke_dev_ew4&#34;&#10;      &#93;&#10;    &#125;&#10;    session_range &#61; &#34;169.254.1.0&#47;29&#34;&#10;    peer &#61; &#123;&#10;      address   &#61; &#34;8.8.8.8&#34;&#10;      asn       &#61; 65534&#10;      secret_id &#61; &#34;foobar&#34;&#10;    &#125;&#10;  &#125;&#10;&#125;">&#123;&#8230;&#125;</code> |  |
| vpn_spoke_configs | VPN gateway configuration for spokes. | <code title="map&#40;object&#40;&#123;&#10;  adv &#61; object&#40;&#123;&#10;    default &#61; bool&#10;    custom  &#61; list&#40;string&#41;&#10;  &#125;&#41;&#10;  session_range &#61; string&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code title="&#123;&#10;  landing-ew1 &#61; &#123;&#10;    adv &#61; &#123;&#10;      default &#61; false&#10;      custom  &#61; &#91;&#34;rfc_1918_10&#34;, &#34;rfc_1918_172&#34;, &#34;rfc_1918_192&#34;&#93;&#10;    &#125;&#10;    session_range &#61; null &#35; values for the landing router are pulled from the spoke range&#10;  &#125;&#10;  landing-ew4 &#61; &#123;&#10;    adv &#61; &#123;&#10;      default &#61; false&#10;      custom  &#61; &#91;&#34;rfc_1918_10&#34;, &#34;rfc_1918_172&#34;, &#34;rfc_1918_192&#34;&#93;&#10;    &#125;&#10;    session_range &#61; null &#35; values for the landing router are pulled from the spoke range&#10;  &#125;&#10;  dev-ew1 &#61; &#123;&#10;    adv &#61; &#123;&#10;      default &#61; false&#10;      custom  &#61; &#91;&#34;spoke_dev_ew1&#34;, &#34;spoke_dev_ew4&#34;&#93;&#10;    &#125;&#10;    session_range &#61; &#34;169.254.0.0&#47;27&#34; &#35; resize according to required number of tunnels&#10;  &#125;&#10;  prod-ew1 &#61; &#123;&#10;    adv &#61; &#123;&#10;      default &#61; false&#10;      custom  &#61; &#91;&#34;spoke_prod_ew1&#34;, &#34;spoke_prod_ew4&#34;&#93;&#10;    &#125;&#10;    session_range &#61; &#34;169.254.0.64&#47;27&#34; &#35; resize according to required number of tunnels&#10;  &#125;&#10;  prod-ew4 &#61; &#123;&#10;    adv &#61; &#123;&#10;      default &#61; false&#10;      custom  &#61; &#91;&#34;spoke_prod_ew1&#34;, &#34;spoke_prod_ew4&#34;&#93;&#10;    &#125;&#10;    session_range &#61; &#34;169.254.0.96&#47;27&#34; &#35; resize according to required number of tunnels&#10;  &#125;&#10;&#125;">&#123;&#8230;&#125;</code> |  |

## Outputs

| name | description | sensitive | consumers |
|---|---|:---:|---|
| cloud_dns_inbound_policy | IP Addresses for Cloud DNS inbound policy. |  |  |
| project_ids | Network project ids. |  |  |
| project_numbers | Network project numbers. |  |  |
| shared_vpc_host_projects | Shared VPC host projects. |  |  |
| shared_vpc_self_links | Shared VPC host projects. |  |  |
| tfvars | Network-related variables used in other stages. | ✓ |  |
| vpn_gateway_endpoints | External IP Addresses for the GCP VPN gateways. |  |  |

<!-- END TFDOC -->


















