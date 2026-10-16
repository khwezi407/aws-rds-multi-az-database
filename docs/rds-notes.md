# RDS Multi-AZ Notes

## Multi-AZ vs Read Replicas

| Feature | Multi-AZ | Read Replica |
| :--- | :--- | :--- |
| Purpose | High availability | Read scaling |
| Replication | Synchronous | Asynchronous |
| Standby accessible | No | Yes (for reads) |
| Failover | Automatic | Manual |
| Number of copies | 1 standby | Up to 5 replicas |
| Cross-region | No | Yes |

## What Happens During Failover

1. Primary AZ fails or instance becomes unhealthy
2. RDS detects the failure within 60-120 seconds
3. The standby instance is promoted to primary
4. The DNS endpoint stays the same
5. Applications reconnect automatically

## DB Subnet Group Requirements

- Must contain subnets in at least 2 Availability Zones
- Subnets should be private (no internet route)
- RDS uses only the subnets in the AZ of the primary and standby

## Security Group Chaining

Instead of allowing an IP range, we reference the Web Security Group as the source. This means only EC2 instances with the Web Security Group can connect. No hardcoded IPs. No changes needed if the web server is replaced.

## Why Databases Should Not Be Public

A public RDS instance has a public IP. This means:
- It is reachable from the internet
- Attackers can attempt to connect
- Brute force attacks become possible

By placing RDS in a private subnet:
- Attackers cannot reach it from the internet
- Only resources inside the VPC can connect
- Security groups enforce which resources are allowed
