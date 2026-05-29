# Architecture

> This document is updated as the lab is built.
> Last updated: 29/05/2026
> Current phase: Phase 0 (Setup)

## What this lab is

A six-VM purple team detection environment simulating a small company (`corp.lab.lan`) on a single Linux workstation via KVM/QEMU. The goal is to attack the environment and engineer the detections that catch each attack, end to end.

## High level design

Four isolated libvirt networks (corporate, management, DMZ, attacker), with pfSense as the only router and IDS sensor between them. Every cross-network packet and every outbound connection passes through pfSense, which means firewall policy and intrusion detection actually apply to all traffic.

Telemetry flows inward to a Wazuh SIEM (Security Information and Event Management platform) on the management network. Attacks originate from a Kali Linux host on the attacker network. The attacker is assumed to have already gained internal network access, which is how almost all real internal pentests and ransomware intrusions are framed.

## Components

- **pfSense** — the firewall, router, DHCP server, DNS resolver, and IDS sensor
- **Wazuh** — the SIEM, plus host intrusion detection via agents on each endpoint
- **Windows Server 2022 Domain Controller (DC01)** — the Active Directory brain
- **Windows 11 workstation (WS01)** — domain-joined employee endpoint
- **Ubuntu Server (LINTGT01)** — domain-joined Linux server running DVWA
- **Kali Linux (KALI01)** — the attacker box

Detailed network design is in [02-network-design.md](02-network-design.md). Per-VM build notes are in [03-build/](03-build/).

![Network topology](../diagrams/network-topology.png)
