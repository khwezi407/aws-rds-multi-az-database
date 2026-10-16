# AWS RDS Multi-AZ Database Deployment

A production-grade Amazon RDS MySQL database deployed with Multi-AZ high availability, connected to a web application running on EC2.

## Overview

This project demonstrates how to provision and configure a highly available managed database on AWS. The database runs in private subnets across two Availability Zones with automatic failover, and is accessible only from a specific web server security group.

## Architecture

![Architecture](screenshots/01-architecture.png)

**Flow:**
1. The web server runs in a public subnet in Availability Zone B
2. The database primary runs in a private subnet in Availability Zone A
3. The database standby runs in a private subnet in Availability Zone B
4. The DB Security Group only allows MySQL (3306) traffic from the Web Security Group
5. Synchronous replication keeps the standby up to date with the primary

## What Was Built

| Component | Purpose |
| :--- | :--- |
| DB Security Group | Allows inbound MySQL (port 3306) from the Web Security Group only |
| DB Subnet Group | Contains two private subnets in different Availability Zones |
| RDS MySQL Instance | db.t3.medium, Multi-AZ, no public access |
| Web Application | Connects to RDS and stores contact data |

## Why Multi-AZ?

Multi-AZ deployments provide:

- High availability — If the primary fails, RDS automatically fails over to the standby
- Data durability — Synchronous replication means no data loss
- Automatic failover — Happens automatically in 60–120 seconds
- Zero configuration — AWS handles the standby instance and replication

## Security Design

- RDS is not publicly accessible
- Only the Web Security Group can connect on port 3306
- Database credentials are required to connect
- RDS runs in private subnets with no internet route
- Security is layered: DB subnet group → DB security group → authentication

## Configuration Details

| Setting | Value |
| :--- | :--- |
| Engine | MySQL Community 8.4.11 |
| Instance class | db.t3.medium (2 vCPU, 4 GB RAM) |
| Deployment | Multi-AZ (2 instances) |
| Storage | 20 GiB gp3, 3000 IOPS |
| Encryption | Enabled (AWS managed KMS key) |
| Public access | No |
| Initial database | lab |
| Master username | main |
| VPC | Lab VPC |
| Subnets | Private Subnet 1 (10.0.1.0/24) and Private Subnet 2 (10.0.3.0/24) |
| Security group | DB Security Group |

## Project Structure
