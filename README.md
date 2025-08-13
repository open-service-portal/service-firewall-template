# Firewall Rule Service Template

A Backstage Software Template for managing firewall rules using Crossplane.

## Overview

This template creates the necessary Crossplane resources to manage firewall rules in your infrastructure. It includes:

- **XRD (Composite Resource Definition)**: Defines the FirewallRule API
- **Composition**: Implements the firewall rule management logic
- **Example Claim**: Shows how to create a firewall rule

## Features

- 🔒 **Security Management**: Define network access control rules
- 🌐 **Protocol Support**: TCP, UDP, ICMP, or all protocols
- ⚡ **Action Control**: Accept, drop, or reject traffic
- 📊 **CIDR Support**: Full IPv4 CIDR notation support

## Prerequisites

### Crossplane Installation

Ensure Crossplane is installed in your cluster:

```bash
kubectl create namespace crossplane-system
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

### Required Functions

Install the necessary Crossplane composition functions:

```bash
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-go-templating
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-go-templating:v0.10.0
---
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-auto-ready
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-auto-ready:v0.2.1
EOF
```

## Usage

### Through Backstage

1. Navigate to the Software Catalog
2. Click "Create Component"
3. Select "Firewall Rule"
4. Fill in the required parameters:
   - **Rule Name**: Unique identifier for this firewall rule
   - **Namespace**: Kubernetes namespace (default: `default`)
   - **Source IP/CIDR**: Source address or network
   - **Destination IP/CIDR**: Destination address or network
   - **Protocol**: ALL, TCP, UDP, or ICMP
   - **Action**: ACCEPT, DROP, or REJECT
   - **Owner**: Team or user who owns this resource

### Manual Deployment

1. Apply the XRD:
```bash
kubectl apply -f content/definition.yaml
```

2. Apply the Composition:
```bash
kubectl apply -f content/composition.yaml
```

3. Create a claim:
```bash
kubectl apply -f content/example.yaml
```

## Parameters

| Parameter | Description | Type | Default | Required |
|-----------|-------------|------|---------|----------|
| `name` | Rule name | string | - | Yes |
| `namespace` | Kubernetes namespace | string | `default` | No |
| `source` | Source IP/CIDR | string | - | Yes |
| `destination` | Destination IP/CIDR | string | - | Yes |
| `protocol` | Network protocol | string | `ALL` | No |
| `action` | Rule action | string | `ACCEPT` | No |
| `owner` | Resource owner | string | `group:platform` | No |

## Rule Examples

### Allow All Traffic
```yaml
spec:
  source: 0.0.0.0/0
  destination: 0.0.0.0/0
  protocol: ALL
  action: ACCEPT
```

### Block Specific Network
```yaml
spec:
  source: 192.168.1.0/24
  destination: 10.0.0.0/8
  protocol: ALL
  action: DROP
```

### Allow HTTPS Traffic
```yaml
spec:
  source: 0.0.0.0/0
  destination: 10.0.0.100
  protocol: TCP
  action: ACCEPT
  # Note: Port specification would be added in production
```

### Allow Ping (ICMP)
```yaml
spec:
  source: 10.0.0.0/24
  destination: 10.0.1.0/24
  protocol: ICMP
  action: ACCEPT
```

## Actions Explained

### ACCEPT
Allows the traffic to pass through. This is the default action for permissive rules.

### DROP
Silently discards the packet without sending any response to the source. Used for stealth blocking.

### REJECT
Blocks the traffic and sends an ICMP response to the source indicating the traffic was rejected.

## Architecture

The template creates a composite resource that:

1. **ConfigMap**: Stores firewall rule configuration
2. **Provider Integration**: Would connect to actual firewall/security group providers
3. **Rule Application**: Applies rules to network infrastructure

## Production Considerations

For production use, you'll need to:

1. **Install a Cloud Provider**: Such as provider-aws, provider-azure, or provider-gcp
2. **Configure Provider Credentials**: Set up authentication for your cloud provider
3. **Update Composition**: Replace the mock implementation with actual security group resources

Example with AWS Security Groups:
```yaml
- step: create-security-group-rule
  functionRef:
    name: function-go-templating
  input:
    apiVersion: gotemplating.fn.crossplane.io/v1beta1
    kind: GoTemplate
    source: Inline
    inline:
      template: |
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: SecurityGroupRule
        spec:
          forProvider:
            region: us-east-1
            type: ingress
            fromPort: 443
            toPort: 443
            protocol: tcp
            cidrBlocks:
              - {{ .observed.composite.resource.spec.source }}
            securityGroupIdRef:
              name: my-security-group
```

Example with Azure Network Security Groups:
```yaml
- step: create-nsg-rule
  functionRef:
    name: function-go-templating
  input:
    apiVersion: gotemplating.fn.crossplane.io/v1beta1
    kind: GoTemplate
    source: Inline
    inline:
      template: |
        apiVersion: network.azure.upbound.io/v1beta1
        kind: SecurityRule
        spec:
          forProvider:
            access: {{ .observed.composite.resource.spec.action }}
            direction: Inbound
            priority: 100
            protocol: {{ .observed.composite.resource.spec.protocol }}
            sourceAddressPrefix: {{ .observed.composite.resource.spec.source }}
            destinationAddressPrefix: {{ .observed.composite.resource.spec.destination }}
```

## CIDR Notation

The template supports standard CIDR notation:

- **Individual IP**: `192.168.1.1`
- **Subnet**: `192.168.1.0/24` (256 addresses)
- **Large Network**: `10.0.0.0/8` (16,777,216 addresses)
- **All IPs**: `0.0.0.0/0`

## Troubleshooting

### Common Issues

**Rule not being applied**
- Check if the composition is properly configured
- Verify ConfigMap was created successfully
- Check composition logs: `kubectl describe composition firewallrule`

**Invalid CIDR format**
- Ensure IP addresses are valid (0-255 for each octet)
- CIDR suffix must be 0-32
- Use online CIDR calculators to verify format

**Rule conflicts**
- Check for overlapping rules with different actions
- Verify rule priority/order if supported by provider

## Security Best Practices

1. **Principle of Least Privilege**: Only allow necessary traffic
2. **Default Deny**: Start with blocking all, then allow specific traffic
3. **Segmentation**: Use different rules for different network segments
4. **Logging**: Enable logging for DROP/REJECT actions
5. **Regular Review**: Periodically audit firewall rules

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

## License

MIT License - see LICENSE file for details

## Support

For issues and questions:
- GitHub Issues: [open-service-portal/service-firewall-template](https://github.com/open-service-portal/service-firewall-template/issues)
- Discussions: [open-service-portal/discussions](https://github.com/orgs/open-service-portal/discussions)