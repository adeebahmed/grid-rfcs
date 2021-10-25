---
Type: `How-to guide`
Tags: `docker`, `vmaas`, `golden-patterns`, `ci-cd`
---

This guide walks through the end-to-end process to provision and deploy a Docker application to a VM in the DMZ.

## Executive Summary

The DMZ env does not provide access to any Docker repositories. To run a Docker application, teams should instead create a tarball of their application, push it to Artifactory, and extract that tarball on their VM.

## Overview

|  |  |
| --- | --- |
| Contributors / Supporting Team | Blockchain |
| Assumed Knowledge | GitHub, Docker, Ansible, Vela |
| Tools/Systems Needed | Docker, GHE, VMaaS, Ansible, Vela, FirewallFast |
| Entitlements Needed | APP-AFT-User-Prod, APP-VMW-VMAAS, APP-PWV-GUIUSERS-PRD |
| Entitlement Time Requirements | Near immediate (pending manager approval) |
| Start Point | A Dockerized application |
| End Point | A VM node in the DMZ running the Dockerized application |
| Pattern Time Requirements | 1-2 days |

Resources

- // Ansible overview

## 1. Provision VMaaS node in the DMZ

> [VMaaS docs site](https://pages.git.target.com/vmaas/doco/)

### 1.1 Create VMaaS AD groups/checkout IDs

In order to provision VMs in the self-service portal, you must have an Active Directory security group containing checkout accounts in which to put in the built form.

- [Creating an AD security group](https://wiki.target.com/tgtwiki/index.php/Active_Directory#How_To_Create_a_new_AD_security_group_create_an_AD_group)
- [Creating checkout IDs](https://wiki.target.com/tgtwiki/index.php/Portal:Project_Team_Security_Toolkit/Access_Control_and_Identity_Management)

### 1.2 Determine DMZ network configuration

Choose the appropriate network configuration outlined in [this Wiki doc](https://wiki.target.com/tgtwiki/index.php/VMaaS_DMZ_guide).

### 1.3 Submit VM provisioning request

Follow the [instructions on the VMaaS docs site](https://pages.git.target.com/vmaas/doco/docs/where_begin/how_to_navigate/) on submitting a VM provisioning request.

## 2. Create firewall access requests

Once your VM has been provisioned, you will need to configure firewall access rules for the traffic from the DMZ -\> Core network.

> Note: External -\> DMZ traffic requires a more thorough vetting process than a simple firewall access request. Engage with your BISO early and often to determine your unique needs.

Visit [go/firewall](https://firewallfast.prod.target.com/) to submit your requests. You will need to first create a firewall group for your VM, and then submit the access request for the group.

Use the [`Is my traffic allowed?`](https://firewallfast.prod.target.com/firewall/is_my_traffic_allowed) tool to verify the access once it has been provisioned.

> Tip: If your DMZ service is communicating with a TAP service, there is an existing firewall group for all TAP servers: [`tapv1`](https://firewallfast.prod.target.com/group/view?name=tapv1)
> 

## 3. Create Binrepo mirror for DMZ 

The Artifactory team maintains a Slack integration that is the easiest way to create an Artifactory repo. Follow along withe steps to [`Create a local repository`](https://pages.git.target.com/artifactory/doc-site/02-selfservice/slack/
).

Next, we will need to create a mirror for this repository in the DMZ.

Create a PR in the [`artifactory-dmz-repositories`](https://git.target.com/chef-dmz/artifactory-dmz-repositories) repo to do this. 

> [Sample mirror request PR](https://git.target.com/chef-dmz/artifactory-dmz-repositories/pull/63)

## 4. Publish our Docker image as a Tarball to Binrepo DMZ



## 5. Create an ansible playbook

Anisble should be used for automated provisioning/configuration of the VM.

> If you are new to Ansible, [read the `Ansible Core` documentation](https://docs.ansible.com/ansible-core/devel/index.html)

The Blockchain team's [`grid-ansible`](https://git.target.com/blockchain/grid-ansible) has a sample playbook that demonstrates using Ansible for DMZ VMaaS provisioning. The relevant details are outlined below.

**Add Binrepo DMZ as a `yum` repo**
```yml
- name: Add DMZ Binrepo
  yum_repository:
    name: epel
    description: Target DMZ Binrepo
    baseurl: https://binrepo-dmz.target.com/
```

**Import our Docker image from Binrepo**
```yml
- name: Import Docker image from Binrepo DMZ
  command: "docker import https://binrepo-dmz.target.com/artifactory/<PATH-TO-YOUR-TARBALL>.tar <YOUR-IMAGE>"
  changed_when: false
```

## 6. Create a `vela.yml`

Our Vela pipeline will need a step the executes our Ansible playbook.

```yml
version: "1"

steps:
  - name: Build <YOUR_SERVICE>
    image: docker.target.com/path-to-your-service:latest

    ## Secrets needed for both the ansible user, and the COID stored in Vault
    secrets: [ANSIBLE_USER, ANSIBLE_PASSWORD, vault_username, vault_password]

    ruleset:
      event: push
      branch: main

    ## [-i INVENTORY] [--list-hosts]
    ## [-e EXTRA_VARS]
    ## [-t TAGS]
    commands:
      - ansible-playbook -i inventories/stage/vmaas/grid/hosts -e @inventories/stage/vmaas/grid/stg.yaml playbook.yml -t vmaas
```
