# CUDN Subnet Ansible Role (`cudn_subnet`)

Provisions a Subnet resource as a **Namespace** and **ClusterUserDefinedNetwork (CUDN)** on remote OpenShift clusters.

This role implements the `cudn` (ClusterUserDefinedNetwork) strategy for the Subnet resource. It is selected when `NetworkClass.implementation_strategy = "cudn"`.

## Role Selection

The playbook dynamically selects this role based on the `osac.openshift.io/implementation-strategy` annotation. This annotation is set by the osac-operator SubnetReconciler after looking up the NetworkClass from the parent VirtualNetwork.

```
osac-operator: Subnet → VirtualNetwork.spec.networkClass → sets annotation
Playbook: Subnet.metadata.annotations['osac.openshift.io/implementation-strategy'] → role name
```

```yaml
- name: Call the selected subnet template
  ansible.builtin.include_role:
    name: "{{ implementation_strategy }}_subnet"
    tasks_from: "create"
  vars:
    implementation_strategy: >-
      {{ ansible_eda.event.payload.metadata.annotations
         ['osac.openshift.io/implementation-strategy'] }}
```

## Architecture Decision

> **IMPORTANT**: This role creates a **Namespace** with proper OVN labels and a **ClusterUserDefinedNetwork (CUDN)**.

The architecture decision was made to use CUDN because:
- CUDN is cluster-scoped, providing better multi-tenant isolation
- namespaceSelector allows targeting specific tenant namespaces via labels
- Aligns with OSAC's multi-tenant architecture where each Subnet provisions a network for a specific tenant namespace

### Resource Mapping

| OSAC Resource | OpenShift Resources | Scope |
|---------------|---------------------|-------|
| VirtualNetwork | None (logical grouping) | N/A |
| Subnet | Namespace + ClusterUserDefinedNetwork | Cluster |

The `cudn_virtual_network` role is a stub that does NOT create any OpenShift resources. Actual network provisioning happens in this `cudn_subnet` role via Namespace and CUDN creation.

## Requirements

- OpenShift 4.18+ with NetworkSegmentation feature gate enabled (GA)
- kubernetes.core collection 6.3.0+

## Role Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `subnet` | Yes | Subnet CR from fulfillment-api via EDA payload |
| `subnet_name` | Yes | Name of the Subnet resource |
| `template_parameters` | No | Template-specific parameters (reserved) |

### Subnet CR Structure

The `subnet` variable should contain a Subnet CR with the following structure:

```yaml
apiVersion: fulfillment.osac.io/v1
kind: Subnet
metadata:
  name: my-subnet
  namespace: fulfillment-system
  annotations:
    osac.io/subnet-id: "my-subnet-id"
    osac.io/tenant-id: "tenant-123"
spec:
  ipv4Cidr: "10.100.0.0/16"
  ipv6Cidr: "fd00:100::/64"  # Optional, for dual-stack
  virtualNetwork: my-virtual-network
```

## Example Playbook

```yaml
---
- name: Provision Subnet
  hosts: localhost
  gather_facts: false
  vars:
    subnet: "{{ ansible_eda.event.payload }}"
    subnet_name: "{{ subnet.metadata.name }}"
    # osac-operator sets this annotation from NetworkClass
    implementation_strategy: >-
      {{ subnet.metadata.annotations['osac.io/implementation-strategy'] }}

  tasks:
    - name: Call the selected subnet template
      ansible.builtin.include_role:
        name: "{{ implementation_strategy }}_subnet"
        tasks_from: "create"
```

## Created Resources

This role creates two resources on the remote cluster:

### 1. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <subnet-name>
  labels:
    osac.io/managed-by: osac-fulfillment
    k8s.ovn.org/primary-user-defined-network: ""  # Required by OVN
    osac.io/subnet-id: <subnet-id>
    osac.io/tenant-id: <tenant-id>        # If provided
    osac.io/virtual-network: <vnet-name>  # If provided
```

The `k8s.ovn.org/primary-user-defined-network: ""` label is required by OVN to indicate this namespace uses a primary user-defined network.

### 2. ClusterUserDefinedNetwork (CUDN)

```yaml
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: <subnet-name>
  labels:
    osac.io/managed-by: osac-fulfillment
spec:
  namespaceSelector:
    matchLabels:
      osac.io/subnet-id: <subnet-id>
  network:
    layer2:
      role: Primary
      ipamLifecycle: Persistent
      subnets:
        - cidr: <ipv4-cidr>
        - cidr: <ipv6-cidr>  # Only if dual-stack
```

### Namespace Selector

The CUDN uses `namespaceSelector` with a label match:
- Label: `osac.io/subnet-id`
- Value: The subnet-id from the Subnet CR's annotations (defaults to metadata.name)

The namespace is created by this role with the matching label, ensuring the CUDN targets the correct namespace.

## What This Role Does

1. Extracts configuration from the Subnet CR
2. Creates a Namespace with required OVN labels (`k8s.ovn.org/primary-user-defined-network`)
3. Builds the CUDN subnets array (IPv4, IPv6, or dual-stack)
4. Creates a ClusterUserDefinedNetwork with Layer2 topology
5. Configures Persistent IPAM for VM compatibility
6. Sets namespaceSelector to target the created namespace

## What This Role Does NOT Do

- Create VirtualNetwork resources (`cudn_virtual_network` is a stub role)
- Manage NetworkPolicy or security rules
- Create pods or workloads in the namespace

## Task Files

| File | Description |
|------|-------------|
| `tasks/create.yaml` | Creates Namespace and CUDN on remote cluster |
| `tasks/delete.yaml` | Deletes CUDN and Namespace from remote cluster |

## Layer2 Configuration

The role uses Layer2 topology by default, which:
- Provides a single broadcast domain suitable for VMs
- Uses Persistent IPAM to retain IP addresses across pod restarts
- Supports VM live migration with stable IP addresses
- Sets role to "Primary" for default network behavior

## Integration with osac-operator

The osac-operator triggers this role via AAP Direct when:
1. A new Subnet CR is created in the fulfillment-service database
2. The SubnetReconciler creates a Subnet CR on the hub cluster
3. AAP Direct triggers the job template with the Subnet CR as payload

The job status is tracked via the CR's status.jobs field.

## Requirements

- `kubernetes.core` collection
- Remote cluster kubeconfig (via OSAC_REMOTE_CLUSTER_KUBECONFIG env var)

## License

Apache-2.0
