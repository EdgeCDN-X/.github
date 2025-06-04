# EdgeCDN-X 👋
EdgeCDN-X is an open source CDN solution built purely on top of k8s and CNCF projects. 
I decided to start to work on this projects, since there are no Open Source CDN solutions available, except for Apache TrafficControl.

My plan with this project is to document the progress and create a project which will be easily understandable and deployable by community members.

For updates feel free to sign up to the [newsletter](https://mailing.edgecdnx.com/subscription/form)

## Features
EdgeCDN-X is a OpenSource CDN solutio built completely on top of Kubernetes and consists of the following components:
* Control-plane - Hosting the UI endpoint and responsible for rolling out the services and configuration via ArgoCD Cluster Generators
* Routing - Routing engine hosts CoreDNS with custom plugins with additional features
* Caching - Caching engine handled via Nginx Ingress controller with customized annotations

## Control-Plane
Control plane is using ArgoCD and custom CRDs and operators to describe the topology, services. This component is WIP, will be using ArgoCD cluster generators to roll out the desired configuration to the individual locations

## Routing
Routing component supports 3 different routing engines:
* DNS Routing - In Progress
* 302 Redirection - Planned
* URL Rewriting API - Planned

Routing component routes the individual requests via the following steps:
* Prefix static routing to individual location (sourced from static prefix list or BGP)
* GeoLookup to locations if static routing returns no destination
* Consistent hashing in location to maximize cache-hit ratio.
* Active healthchecks to make sure destinations are healthy and available
* Fallback routing to different location if location has no active nodes

Routing engine is rolled out to each location with **edgecdnx.com/routing** label in the cluster [metadata](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Cluster/)

### Static Prefix routing
[edgecdnx-prefixlist](https://github.com/EdgeCDN-X/edgecdnx-prefixlist) CoreDNS module gathers all the prefixes for the individual locations. These prefixes must be normalized and must be non overlapping. There's a helper [operator](https://github.com/EdgeCDN-X/prefixlist-controller) which helps to achieve Prefix Consolidation and Supernet subnetting, to make sure there are no overlaps in the Prefixes. The Module is using CoreDNS's Metadata interface to find the desired destination for a given prefix.

Features:
* Routing to location based on Client IP address
* Prefixes are stored in a fast balanced AVL Tree to ensure speedy lookups
* EDNS0 Subnet extension support
* IPv4 and IPv6 Supported

Status: PoC Implemented

### GeoLookup routing
[edgecdnx-geolookup](https://github.com/EdgeCDN-X/edgecdnx-geolookup) CoreDNS module finds the most suitable locatio based on MMDB2 DB. This module uses [geoip](https://coredns.io/plugins/geoip/) metadata to enrich the necessary fields.
Geolookup module assigns weights and score for each request and the location is based on this score. If multiple locations are found with the same score, the requests are balanced based on associated weight. (e.g. eu-west and eu-east routing to Germany in ratio of 40:60)


Status: PoC Implemented

### Service catalog
[edgecdnx-services](https://github.com/EdgeCDN-X/edgecdnx-services) CoreDNS module loads the configuration from yaml files for the individual services. This module is simple and makes sure that we do not return a success response for services which are not defined in the configuration file. This module handles the final routing decision with consistent hashing in a selected region. Healthchecks make sure that the desired node in a given location is available (Likely consul to be used)

Status: In Progress

## Caching
[edgecdnx-cache](https://github.com/EdgeCDN-X/bootstrap/blob/main/edgecdnx/edgecdnx-cache.yaml) manifest rolls out an NGINX Deamonset for each location specified. Each location is specified as a k8s cluster and cluster definition must be labeled with **edgecdnx.com/caching** label in the cluster [metadata](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Cluster/)

The Ingress controller is rolled out as a daemonset. Multiple instances of the caching engine can be rolled out for different caching tiers (e.g., ram, ssd). The underlying storage for caching must be prepared beforehand. To ensure maximum performance the ingress controller is attached to the hostNetwork and mounts the Caches as HostPath Volume. These settings can be modified editing the base manifest.


