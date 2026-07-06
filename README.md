# 🛡️ ARCHISOC — Architecture de Cybersécurité Virtualisée

> Laboratoire SOC complet : segmentation réseau, pare-feu FortiGate + pfSense/Suricata, honeypots DMZ, Active Directory, SIEM Wazuh et SOAR Shuffle.

![Status](https://img.shields.io/badge/status-completed-brightgreen)
![Platform](https://img.shields.io/badge/platform-VMware%20Workstation%20Pro-informational)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh%204.14-orange)

## 📖 Table des matières

1. [Plan d'adressage](#1-plan-dadressage)
2. [Configuration VMware](#2-configuration-vmware)
3. [Installation FortiGate](#3-installation-et-configuration-fortigate)
4. [Installation pfSense](#4-installation-de-pfsense)
5. [Routes statiques FortiGate](#5-routes-statiques-fortigate)
6. [Configuration GUI pfSense](#6-configuration-gui-pfsense)
7. [VM Ubuntu DMZ (DVWA + Cowrie)](#7-vm-ubuntu-dmz-dvwa--cowrie)
8. [VM Wazuh Server](#8-vm-wazuh-server-sur-lan-soc)
9. [Connectivité DMZ → SOC](#9-connectivité-dmz-to-soc)
10. [Agent Wazuh sur DMZ](#10-installation-de-lagent-wazuh-sur-vm-dmz)
11. [Logs FortiGate → Wazuh](#11-intégration-logs-fortigate-dans-wazuh)
12. [Suricata sur pfSense → Wazuh](#12-installation-suricata-et-envoi-logs)
13. [Agent Wazuh sur pfSense](#13-installation-agent-wazuh-sur-pfsense)
14. [LAN AD & Client Windows](#14-lan-ad--client)
15. [Agents Wazuh sur DC et client](#15-agent-wazuh-sur-dc-et-client-windows)
16. [Collecte logs AD & Sysmon](#16-collecte-logs-ad-et-winclient)
17. [Démonstration des scénarios d'attaque](#17-démonstration-scénarios-dattaque)

## 🏗️ Architecture globale

![Architecture globale](screenshots/00_architecture_globale.png)
*`00_architecture_globale.png` — Schéma Draw.io complet montrant FortiGate, pfSense/Suricata, la DMZ (honeypot SSH + DVWA), le LAN AD+Client, et le LAN SOC (Wazuh + Shuffle) avec toutes les IP de chaque interface.*
