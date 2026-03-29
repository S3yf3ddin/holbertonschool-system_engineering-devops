# Web Infrastructure Design

This directory contains design tasks covering progressively more complex web infrastructure architectures. Each file provides a detailed explanation and diagram of the infrastructure, along with the rationale behind each design decision and its known limitations.

## Files

| File | Description |
|------|-------------|
| [0-simple_web_stack](./0-simple_web_stack) | **Simple Web Stack** — A single-server infrastructure hosting `www.foobar.com`. Covers DNS (A record), Nginx web server, application server, MySQL database, and the request flow. Discusses SPOF, maintenance downtime, and scalability limitations. |
| [1-distributed_web_infrastructur](./1-distributed_web_infrastructur) | **Distributed Web Infrastructure** — A three-server setup using an HAProxy load balancer (Round Robin) with two application servers in Active-Active mode and a MySQL Primary-Replica database cluster. Discusses remaining SPOFs, lack of firewall, and missing HTTPS. |
| [2-secured_and_monitored_web_infrastructure](./2-secured_and_monitored_web_infrastructure) | **Secured and Monitored Web Infrastructure** — Builds on the distributed design by adding firewalls to each server, an SSL certificate for HTTPS termination at the load balancer, and monitoring agents reporting to a centralized service (e.g., Datadog). Discusses SSL termination issues, single write master problem, and resource contention. |
| [3-scale_up](./3-scale_up) | **Scale Up** — Eliminates the load balancer SPOF by clustering two HAProxy instances (Active-Passive via Keepalived) and separates all components (web server, application server, database) onto dedicated servers. Explains the distinction between a web server and an application server. |

## Concepts Covered

- DNS and domain name resolution
- Web servers (Nginx) vs. application servers
- Load balancing algorithms (Round Robin)
- Active-Active vs. Active-Passive setups
- Database Primary-Replica (Master-Slave) replication
- Firewalls and HTTPS/SSL termination
- Infrastructure monitoring and observability
- Component separation for scalability and resource isolation
