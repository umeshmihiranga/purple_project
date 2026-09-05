# BLACKFORGE — Segmentation Baseline

## Date
2026-09-05

## Phase
Phase 1 — Network Segmentation

## Objective

Establish the baseline network architecture before implementing
DMZ/private segmentation.

## Current Architecture

- Server #1: DMZ candidate
- Server #2: Private candidate
- Platform: AWS Lightsail
- Attacker/administrator: Kali Linux

## Current Network

Server #1:
- Private IP: 172.26.6.116/20

Server #2:
- Private IP: 172.26.1.48/20

Both servers currently exist in the same private /20 network.

## Baseline Connectivity

Server #1 → Server #2:
- ICMP: allowed
- TCP/22: allowed
- TCP/2244: allowed

Server #2 → Server #1:
- ICMP: allowed
- TCP/2244: allowed

## Initial Security Assessment

The current architecture is effectively flat.

The Lightsail private network allows direct communication
between the two instances.

Therefore, the current state does NOT provide true
DMZ/private subnet isolation.

## Target Architecture

Server #1:
- DMZ / edge
- Nginx
- ModSecurity
- WireGuard
- Docker

Server #2:
- Private/internal
- Backend applications
- Database
- Internal services

Management:
- Kali → WireGuard → internal infrastructure

## Security Goal

Internet-facing services should terminate on Server #1.

Server #2 should not be directly reachable from the Internet.

Only explicitly authorized traffic should cross from Server #1
to Server #2.

## Important Constraint

AWS Lightsail does not provide the same custom subnet-level
segmentation available with a full EC2/VPC architecture.

Therefore BLACKFORGE will implement additional segmentation
using Linux firewalling, routing, WireGuard, service binding,
and application-layer controls.

## Evidence

Terminal outputs:
- IP addressing
- Routing table
- Listening services
- Neighbor table
- Connectivity tests

## Status

- [x] Current topology identified
- [x] Private addressing identified
- [x] Inter-server connectivity confirmed
- [ ] DMZ controls implemented
- [ ] Private host controls implemented
- [ ] WireGuard management plane implemented
- [ ] Traffic policy validated