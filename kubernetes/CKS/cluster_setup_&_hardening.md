# Cluster setup and hardening

## CIS Benchmark
The Center for Internet Security (CIS) Kubernetes Benchmark provides security recommendations for configuring Kubernetes clusters. This guide helps you implement these recommendations using kube-bench.

## Kube-bench

Kube-bench is a tool fromt aquasecurity hat checks whether your Kubernetes cluster is deployed according to security best practices defined in the CIS Kubernetes Benchmark.

### 1. Install kube-bench

```shell
# Download the latest version of kube-bench
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.10.6/kube-bench_0.10.6_linux_amd64.tar.gz -o kube-bench_0.10.6_linux_amd64.tar.gz

# Extract the files
tar -xvf kube-bench_0.10.6_linux_amd64.tar.gz
```
Ref Setup Documentation: [Download 0.4.0 binary and follow the user guide](https://github.com/aquasecurity/kube-bench/blob/main/docs/installation.md#download-and-install-binaries)

### 2. Run kube-bench

```shell
# Run kube-bench with default configuration
./kube-bench
```

If you need to specify config directories:
```shell
./kube-bench --config-dir `pwd`/cfg --config `pwd`/cfg/config.yaml
```
### 3. Understanding the Results

The scan produces four types of results:
- **[PASS]**: The check was successful
- **[FAIL]**: The check failed and needs remediation
- **[WARN]**: The check needs manual verification or further attention
- **[INFO]**: Informational output only

Example output:
```log
[INFO] 4 Worker Node Security Configuration
[INFO] 4.1 Worker Node Configuration Files
[PASS] 4.1.1 Ensure that the kubelet service file permissions are set to 644 or more restrictive
[FAIL] 4.1.3 Ensure that the proxy configuration file permissions are set to 644 or more restrictive
== Remediations ==
4.1.3 Run the below command on each worker node:
chmod 644 /etc/kubernetes/proxy.conf
== Summary ==
16 checks PASS
1 check FAIL
6 checks WARN
0 checks INFO
```
    
### 4. Fixing Issues

For each FAIL or WARN result, kube-bench provides specific remediation steps:

The Output Report clearly identifies the status (FAIL, WARN, and PASS). For any failed or warning checks, detailed remediation instructions are provided. Simply follow these instructions to address each security issue:

Example:
- **1.3.6** Edit the Controller Manager pod specification file `/etc/kubernetes/manifests/kube-controller-manager.yaml` 
on the master node and add the `--feature-gates` parameter with `RotateKubeletServerCertificate=true`.

Solution:
```shell
vi /etc/kubernetes/manifests/kube-controller-manager.yaml
```

Add or modify the line to include: `--feature-gates=RotateKubeletServerCertificate=true`

### 5. Verify Fixes

After implementing the recommended changes, re-run kube-bench to verify that all issues have been successfully fixed:

```shell
./kube-bench --config-dir `pwd`/cfg --config `pwd`/cfg/config.yaml
```