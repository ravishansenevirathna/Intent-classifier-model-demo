# Intent Classifier Model - AWS Deployment Guide

This repository contains an Intent Classifier ML model with a complete AWS deployment setup using EC2, Auto Scaling Group (ASG), and Application Load Balancer (ALB).

---

## Architecture Overview

The deployment creates a highly available, scalable infrastructure:

- **VPC** with 2 public subnets across different Availability Zones
- **Auto Scaling Group (ASG)** to automatically scale EC2 instances
- **Application Load Balancer (ALB)** to distribute traffic

---

## Complete Workflow Architecture

```
                                    INTERNET
                                        │
                                        ▼
                            ┌───────────────────────┐
                            │    Application Load   │
                            │    Balancer (ALB)     │
                            │    Port 80 (HTTP)     │
                            └───────────┬───────────┘
                                        │
                    ┌───────────────────┴───────────────────┐
                    │                                       │
                    ▼                                       ▼
        ┌───────────────────────┐           ┌───────────────────────┐
        │   Availability Zone A │           │   Availability Zone B │
        │   Subnet: 10.10.1.0/24│           │   Subnet: 10.10.2.0/24│
        │                       │           │                       │
        │   ┌───────────────┐   │           │   ┌───────────────┐   │
        │   │  EC2 Instance │   │           │   │  EC2 Instance │   │
        │   │               │   │           │   │               │   │
        │   │  ┌─────────┐  │   │           │   │  ┌─────────┐  │   │
        │   │  │  Nginx  │  │   │           │   │  │  Nginx  │  │   │
        │   │  │  :80    │  │   │           │   │  │  :80    │  │   │
        │   │  └────┬────┘  │   │           │   │  └────┬────┘  │   │
        │   │       │       │   │           │   │       │       │   │
        │   │  ┌────▼────┐  │   │           │   │  ┌────▼────┐  │   │
        │   │  │Gunicorn │  │   │           │   │  │Gunicorn │  │   │
        │   │  │ :6000   │  │   │           │   │  │ :6000   │  │   │
        │   │  └────┬────┘  │   │           │   │  └────┬────┘  │   │
        │   │       │       │   │           │   │       │       │   │
        │   │  ┌────▼────┐  │   │           │   │  ┌────▼────┐  │   │
        │   │  │  Flask  │  │   │           │   │  │  Flask  │  │   │
        │   │  │  App    │  │   │           │   │  │  App    │  │   │
        │   │  └─────────┘  │   │           │   │  └─────────┘  │   │
        │   └───────────────┘   │           │   └───────────────┘   │
        └───────────────────────┘           └───────────────────────┘
                    │                                   │
                    └───────────────┬───────────────────┘
                                    │
                            ┌───────▼───────┐
                            │ Auto Scaling  │
                            │    Group      │
                            │  (min:1 max:3)│
                            └───────────────┘
```

---

## Request Flow (Step by Step)

| Step | Component | What Happens |
|------|-----------|--------------|
| 1 | **User** | Sends HTTP request to ALB's public DNS |
| 2 | **ALB** | Receives request, checks target group for healthy instances |
| 3 | **ALB** | Routes to an EC2 instance (round-robin load balancing) |
| 4 | **Nginx** (:80) | Receives request, acts as reverse proxy |
| 5 | **Gunicorn** (:6000) | WSGI server receives request from Nginx |
| 6 | **Flask App** | Processes `/predict` endpoint, runs ML model inference |
| 7 | **Response** | Travels back: Flask → Gunicorn → Nginx → ALB → User |

---

## Component Responsibilities

| Component | Role |
|-----------|------|
| **ALB** | Load balancing, health checks (`/health`), SSL termination (if configured) |
| **ASG** | Maintains desired instance count, replaces unhealthy instances, scales up/down |
| **Nginx** | Reverse proxy, handles static files, connection buffering |
| **Gunicorn** | Python WSGI server, manages 3 worker processes |
| **Flask** | Application logic, ML model inference |

---

## VPC and Networking

### Why 10.10.0.0/16?

This is a **private IP range** (from RFC 1918). The `/16` means you get 65,536 IP addresses (10.10.0.0 - 10.10.255.255). It's a common choice for VPCs - large enough for most workloads.

### Why 2 Subnets in Different AZs?

**1. ALB Requirement**
Application Load Balancers **require at least 2 subnets in different Availability Zones**. This is an AWS hard requirement.

**2. High Availability**
Each AZ is a physically separate data center. If one AZ goes down:
- Subnet in AZ-a fails → instances there become unavailable
- Subnet in AZ-b still works → ALB routes traffic to healthy instances
- Your app stays online

```
        ┌─────────────────────────────────────┐
        │           VPC 10.10.0.0/16          │
        │                                     │
        │  ┌─────────────┐  ┌─────────────┐   │
        │  │   AZ-a      │  │   AZ-b      │   │
        │  │ 10.10.1.0/24│  │ 10.10.2.0/24│   │
        │  │   [EC2]     │  │   [EC2]     │   │
        │  └──────┬──────┘  └──────┬──────┘   │
        │         │                │          │
        │         └───────┬────────┘          │
        │                 │                   │
        │              [ ALB ]                │
        └─────────────────┼───────────────────┘
                          │
                      Internet
```

---

## Target Group

A **Target Group** is a collection of destinations (EC2 instances, IPs, or Lambda functions) where the ALB sends traffic.

```
┌─────────────────────────────────────────────────────────┐
│                    TARGET GROUP                         │
│                  "mlops-target-group"                   │
│                                                         │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐         │
│   │  EC2-1   │    │  EC2-2   │    │  EC2-3   │         │
│   │ Healthy  │    │ Healthy  │    │Unhealthy │         │
│   │    ✓     │    │    ✓     │    │    ✗     │         │
│   └──────────┘    └──────────┘    └──────────┘         │
│                                                         │
│   Protocol: HTTP                                        │
│   Port: 80                                              │
│   Health Check: GET /health (expect 200)                │
└─────────────────────────────────────────────────────────┘
```

### What Target Group Does

| Function | Description |
|----------|-------------|
| **Registers targets** | Keeps list of instances that can receive traffic |
| **Health checks** | Periodically pings `/health` endpoint to verify instances are working |
| **Routes only to healthy** | ALB only sends traffic to targets marked "healthy" |
| **Deregistration** | Removes unhealthy or terminated instances |

---

## ALB Listener

A **Listener** is a process that checks for incoming connection requests on a specific port/protocol and defines rules for routing them.

```
                        INTERNET
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│                          ALB                              │
│                                                           │
│   ┌─────────────────────────────────────────────────┐     │
│   │              LISTENER (Port 80, HTTP)           │     │
│   │                                                 │     │
│   │   Rule 1: IF path = /api/*  → Target Group A    │     │
│   │   Rule 2: IF path = /admin* → Target Group B    │     │
│   │   Default: Forward all     → Target Group C     │     │
│   └─────────────────────────────────────────────────┘     │
│                                                           │
│   ┌─────────────────────────────────────────────────┐     │
│   │             LISTENER (Port 443, HTTPS)          │     │
│   │                                                 │     │
│   │   SSL Certificate: *.example.com                │     │
│   │   Default: Forward all → Target Group C         │     │
│   └─────────────────────────────────────────────────┘     │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### What Listener Does

| Function | Description |
|----------|-------------|
| **Listens on port** | Waits for incoming requests (e.g., port 80 for HTTP, 443 for HTTPS) |
| **Protocol handling** | Handles HTTP, HTTPS, or both |
| **Routing rules** | Decides which target group receives the request |
| **SSL termination** | Decrypts HTTPS traffic (if configured) |

---

## How ALB, Listener, and Target Group Work Together

```
User Request (http://alb-dns.com/predict)
                │
                ▼
        ┌───────────────┐
        │     ALB       │
        └───────┬───────┘
                │
                ▼
        ┌───────────────┐
        │   LISTENER    │  ← "I listen on port 80"
        │   Port 80     │
        └───────┬───────┘
                │
                ▼
        ┌───────────────┐
        │ DEFAULT RULE  │  ← "Forward to target group"
        │   forward →   │
        └───────┬───────┘
                │
                ▼
        ┌───────────────┐
        │ TARGET GROUP  │  ← "Here are my healthy instances"
        │               │
        │  EC2-1  EC2-2 │
        └───────┬───────┘
                │
                ▼
        ┌───────────────┐
        │ EC2 Instance  │  ← Request delivered
        │ (Nginx:80)    │
        └───────────────┘
```

### Summary

| Component | Analogy |
|-----------|---------|
| **ALB** | Reception desk |
| **Listener** | Receptionist listening for calls on specific phone line (port) |
| **Rules** | Instructions on where to route different types of calls |
| **Target Group** | Department with available staff (healthy instances) |

---

## Auto Scaling Behavior

```
Low Traffic                          High Traffic
    │                                     │
    ▼                                     ▼
┌────────┐                         ┌────────┐ ┌────────┐ ┌────────┐
│  EC2   │    ──── scales to ────▶ │  EC2   │ │  EC2   │ │  EC2   │
└────────┘                         └────────┘ └────────┘ └────────┘
 (min: 1)                                   (max: 3)
```

The ASG monitors instance health and can scale between 1-3 instances based on demand or if an instance becomes unhealthy.

---

## Files in This Repository

| File | Purpose |
|------|---------|
| `userdata.sh` | Bootstrap script that runs on EC2 instance launch - installs dependencies, clones repo, configures Nginx & Gunicorn |
| `complete-model-deployment.md` | Step-by-step AWS CLI commands for deployment |
| `model/` | ML model training code |
| `wsgi.py` | WSGI entry point for Gunicorn |
| `requirements.txt` | Python dependencies |

---

## Quick Start

Refer to [complete-model-deployment.md](complete-model-deployment.md) for detailed AWS CLI commands to deploy this infrastructure.
