# Cloud Native Security 4C's

<p align="center">
    <img src="../images/4C&apos;s-cks.png" alt="Cloud Native Security 4C's"/>
</p>

The 4C's represent four layers of cloud native security that build upon each other:

## 1. Cloud/Infrastructure Security
- **Datacenter**: Physical security measures and environmental controls
- **Network**: Firewalls, VPNs, and network segregation to protect infrastructure
- **Servers**: Host-level hardening, regular patching, and minimal attack surface

## 2. Cluster Security
- **Authentication**: Verifying identity through certificates, tokens, or OIDC
- **Authorization**: Controlling access with RBAC policies to limit what authenticated users can do
- **Admission**: Validating and/or modifying requests to the Kubernetes API after authentication
- **Network Policies**: Controlling pod-to-pod communication and enforcing network segmentation

## 3. Container Security
- **Restrict Images**: Using trusted registries and minimal base images
- **Supply Chain**: Ensuring security throughout the container lifecycle from build to deployment
- **Sandboxing**: Isolating containers using runtime security controls
- **Privileged**: Restricting container capabilities and avoiding privileged mode

## 4. Code Security
- **Secure Coding**: Following security best practices in application development
- **Secrets Management**: Using secure methods to handle credentials and sensitive data
- **Common Anti-patterns to Avoid**:
  - Hardcoding credentials in application code
  - Passing sensitive information through environment variables
  - Deploying applications without proper TLS encryption