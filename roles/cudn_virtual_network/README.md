# CUDN VirtualNetwork Ansible Role (`cudn_virtual_network`)

## Overview

This role provisions VirtualNetwork resources as part of the OSAC (OpenShift-as-a-Service-in-a-Container) networking stack.

This role implements the `cudn` (ClusterUserDefinedNetwork) strategy for the VirtualNetwork resource. It is selected when `NetworkClass.implementation_strategy = "cudn"`.

**IMPORTANT ARCHITECTURAL NOTE:** VirtualNetwork is a **logical grouping** for network configuration. This role is a **stub implementation** that does NOT create any OpenShift resources (no ClusterUserDefinedNetwork/CUDN). The actual network provisioning happens when Subnet resources are created (see `cudn_subnet` role).

## Role Selection

The playbook dynamically selects this role based on the NetworkClass implementation strategy:

```yaml
- name: Call the selected virtual network template
  ansible.builtin.include_role:
    name: "{{ network_class_implementation_strategy }}_virtual_network"
    tasks_from: "create"
```

## Architecture Decision

Based on v3.0 architecture research, VirtualNetwork serves as a parent resource for organizing Subnets and SecurityGroups. The actual User Defined Network (UDN) resources are created by the `cudn_subnet` role, not this role.

This design choice aligns with the OVN-Kubernetes networking model where:
- **Subnet** → ClusterUserDefinedNetwork (CUDN) - cluster-scoped, creates the actual network
- **VirtualNetwork** → Logical grouping - provides configuration context for child Subnets

## Usage

This role is called by AAP Direct via Event-Driven Ansible (EDA) when a VirtualNetwork CR is created or deleted.

### Input Variables

| Variable | Type | Required | Description |
|----------|------|----------|-------------|
| `virtual_network` | dict | Yes | VirtualNetwork CR from EDA payload |
| `virtual_network_name` | string | Yes | Name of the VirtualNetwork resource |
| `template_parameters` | dict | No | Template-specific parameters (reserved for future use) |

### Example Playbook

```yaml
- name: Create VirtualNetwork
  hosts: localhost
  gather_facts: false
  vars:
    virtual_network: "{{ ansible_eda.event.payload }}"
    virtual_network_name: "{{ ansible_eda.event.payload.metadata.name }}"
    template_parameters: {}

  tasks:
    - name: Call VirtualNetwork template
      ansible.builtin.include_role:
        name: "{{ network_class_implementation_strategy }}_virtual_network"
        tasks_from: create
```

Where `network_class_implementation_strategy` is determined from the NetworkClass lookup (e.g., `"cudn"`).

### VirtualNetwork CR Structure

Expected CR structure from EDA payload:

```yaml
apiVersion: osac.innabox.com/v1alpha1
kind: VirtualNetwork
metadata:
  name: my-vnet
  namespace: fulfillment
  annotations:
    osac.io/tenant-id: "tenant-123"
spec:
  networkClass: default-network-class
  region: us-east-1
  ipv4Cidr: "10.128.0.0/14"
  ipv6Cidr: "fd01::/48"  # Optional - for dual-stack
```

## Tasks

### create.yaml

Logs VirtualNetwork configuration information and returns success. Does NOT create any Kubernetes resources.

### delete.yaml

Logs VirtualNetwork deletion completion and returns success. No resources to clean up.

## What This Role Does

1. Extracts VirtualNetwork configuration from EDA payload
2. Logs network configuration details (name, CIDRs, NetworkClass, region, tenant-id)
3. Returns success

## What This Role Does NOT Do

1. Does NOT create ClusterUserDefinedNetwork (CUDN) resources
2. Does NOT create UserDefinedNetwork (UDN) resources
3. Does NOT validate CIDR formats (validation happens in fulfillment-service API)
4. Does NOT provision any network infrastructure

## Related Roles

- **cudn_subnet**: Creates Namespace and ClusterUserDefinedNetwork (CUDN) resources - the actual network provisioning
- **cudn_security_group**: Creates NetworkPolicy resources for security rules (future)

## Integration with AAP

This role integrates with Ansible Automation Platform (AAP) Direct via:
- `meta/osac.yaml`: Defines `template_type: virtual_network` for AAP discovery
- `meta/argument_specs.yaml`: Defines variable specifications for AAP validation
- EDA payload structure: Receives VirtualNetwork CR via `ansible_eda.event.payload`

## Requirements

- Ansible 2.15.0+
- kubernetes.core collection 6.3.0+ (for shared kubeconfig helper)

## License

Proprietary - OSAC Project

## Author

OSAC Team
