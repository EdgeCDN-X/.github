# EdgeCDN-X 👋
Thank you for visiting my page. 

EdgeCDN-X is an open source CDN solution built purely on top of k8s and CNCF projects. 
I decided to start to work on this projects, since there are no Open Source CDN solutions available, except for Apache TrafficControl.

My plan with this project is to document the progress and create a project which will be easily understandable and deployable by community members.
For updates feel free to sign up to the [newsletter](https://mailing.edgecdnx.com/subscription/form)



## Features
EdgeCDN-X is consists of the following components:
* Control-plane - Hosting the UI endpoint and responsible for rolling out the services and configuration via ArgoCD Cluster Generators
* Routing - Routing engine hosts CoreDNS with custom plugins with additional features
* Caching - Caching engine handled via Nginx Ingress controller with customized annotations

## Control-Plane
Control plane is using ArgoCD and custom CRDs and operators to describe the topology, services. This component is WIP, will be using ArgoCD cluster generators to roll out the desired configuration to the individual locations

## Routing
Routing component supports 3 different routing engines:
* DNS Routing - In Alpha ✅
* 302 Redirection - Planned 🔜
* URL Rewriting API - Planned 🔜

Routing component routes the individual requests via the following steps:
* Prefix static routing to individual location (sourced from static prefix list ✅ or BGP 🔜) ✅
* GeoLookup to locations if static routing returns no destination ✅
* Consistent hashing in location to maximize cache-hit ratio. ✅
* Active healthchecks to make sure destinations are healthy and available 🔜
* Fallback routing to different location if location has no active nodes ✅

Routing engine is rolled out to each location with **edgecdnx.com/routing** label in the cluster [metadata](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Cluster/)

### Static Prefix routing
[edgecdnx-prefixlist](https://github.com/EdgeCDN-X/edgecdnx-prefixlist) CoreDNS module gathers all the prefixes for the individual locations. These prefixes must be normalized and must be non overlapping. There's a helper [operator](https://github.com/EdgeCDN-X/edgecdnx-controller) which helps to achieve Prefix Consolidation and Supernet subnetting, to make sure there are no overlaps in the Prefixes. The Module is using CoreDNS's Metadata interface to find the desired destination for a given prefix.

Features:
* Routing to location based on Client IP address 
* Prefixes are stored in a fast balanced AVL Tree to ensure speedy lookups
* EDNS0 Subnet extension support
* IPv4 and IPv6 Supported

Status:  ✅ 

### GeoLookup routing
[edgecdnx-geolookup](https://github.com/EdgeCDN-X/edgecdnx-geolookup) CoreDNS module finds the most suitable locatio based on MMDB2 DB. This module uses [geoip](https://coredns.io/plugins/geoip/) metadata to enrich the necessary fields.
Geolookup module assigns weights and score for each request and the location is based on this score. If multiple locations are found with the same score, the requests are balanced based on associated weight. (e.g. eu-west and eu-east routing to Germany in ratio of 40:60)


Status: ✅

### Service catalog
[edgecdnx-services](https://github.com/EdgeCDN-X/edgecdnx-services). This module is responsible for building the SOA and NS records and also enriches metadata with customer specific information for better routing decitions down the line. All thes services are auto loaded via the k8s client, so it is not required to reload the Configuration when a new service is configured.

Example configuration:
```
        edgecdnxservices {
            namespace edgecdnx-routing
            soa ns
            email noc.edgecdnx.com
            ns ns2 189.167.203.182
            ns ns2 190.167.203.183
        }
```

Status: ✅

## Caching
[edgecdnx-cache](https://github.com/EdgeCDN-X/bootstrap/blob/main/edgecdnx/edgecdnx-cache.yaml) manifest rolls out an NGINX Deamonset for each location specified. Each location is specified as a k8s cluster and cluster definition must be labeled with **edgecdnx.com/caching** label in the cluster [metadata](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Cluster/)

The Ingress controller is rolled out as a daemonset. Multiple instances of the caching engine can be rolled out for different caching tiers (e.g., ram, ssd). The underlying storage for caching must be prepared beforehand. To ensure maximum performance the ingress controller is attached to the hostNetwork and mounts the Caches as HostPath Volume. These settings can be modified editing the base manifest.


### Origin Support
We definitely want to support multiple origin types such as, S3 and Static Origins
* Static origins - HTTP, HTTPS based origins - In progress - 👷‍♂️
* S3 Origins -  🔜

### SSL Certificate management
This is a bit tricky use case. As per ACME, DNS based certificate issuance can be created for the owned domain (currently edgecdnx.com), we have to solve the challenges of distributing that certificate to the individual endpoints. For customer domains we do not have access to their registrar and NS so we have to fallback to HTTP based challenge. The problem is, that since this is a CDN, the challenge can end up on any of the nodes due to DNS redirection. For this purpose, we will start the challenge on the control plane and build a small helper reverse proxy, which will direct those requests from the individual endpoints to the control plane endpoint where the cert issuance is in progress. Once issued, we have to distribute the Certificates to the individual Endpoints. 🔜

# Planned features for MVP
* Multi cache support -  ✅ - Supported, Multiple Nginx definitions have to be defined
* S3 upstream connector -  🔜
* DNS routing -  ✅
* 302 redirection routing - 🔜 - Will be supported by attaching to the CoreDNS gRPC endpoint
* Active Healthchecks -  🔜
* Control plane based on CRDs - v1alpha1 version  ✅
* Controller UI  🔜

