Continuous integration | License
 -----------------------|--------
![Continuous integration](https://github.com/chimera-kube/pod-privileged-policy/workflows/Continuous%20integration/badge.svg) | [![License: Apache 2.0](https://img.shields.io/badge/License-Apache2.0-brightgreen.svg)](https://opensource.org/licenses/Apache-2.0)

This project contains a Chimera policy written using [AssemblyScript](https://assemblyscript.org/),
a subset of TypeScript.

# The goal

Given the following scenario:

> As an operator of a Kubernetes cluster used by multiple users,
> I want to have tight control over who can schedule privileged containers.

Kubernetes containers can be run in privileged mode by providing a well crafted
[SecurityContext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/).

Cluster administrators can prevent regular users to create privileged containers
by using a Kubernetes built-in feature called [Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/).

However, Pod Security Polices are still in Beta phase and they are probably
going to be [deprecated](https://github.com/kubernetes/enhancements/issues/5)
in the near future.

Pod Security Policies could be replaced by using policies provided by an
external Admission Controller, like Chimera.

This policy inspects the [AdmissionReview](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#request)
objects generated by the Kubernetes API server and either accept or reject
them.

The policy can be used to inspect `CREATE` and `UPDATE` requests of
`Pod` resources.

## Examples

The following Pod specification doesn't have any security context defined:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

This workload can be scheduled by all the users of the cluster.

This Pod specification has one of its containers running in
privileged mode:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  runtimeClassName: containerd-runc
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
    securityContext:
      privileged: true
  - name: sleeping-sidecar
    image: alpine
    command: ["sleep", "1h"]
```


# Configuration

The policy behaviour can be influenced by these environment variables:

  * `TRUSTED_USERS`: comma separated list of users who are allowed to create
    privileged containers. Optional.
  * `TRUSTED_GROUPS`: comma separated list of groups who are allowed to create
    privileged containers. Optional.

# Obtain policy

The policy is automatically published as an OCI artifact inside of
[this](https://github.com/orgs/chimera-kube/packages/container/package/policies%2Fpod-privileged)
container registry:

# Building

This policy is written using [AssemblyScript](https://www.assemblyscript.org/).
The [as-wasi](https://github.com/jedisct1/as-wasi) library is used to target
the [WASI interface](https://wasi.dev/).

JSON parsing and encoding is handled via the [assemblyscript-json](https://github.com/nearprotocol/assemblyscript-json)
library.

Unit testing is done using the [as-pect](https://github.com/jtenner/as-pect)
library.

Building can be done using this command:

```
$ make build
```

This will produce the following Wasm binaries:

  * `build/optimized.wasm`: binary built with release optimizations
  * `build/untouched.wasm`: binary built without optimizations

# Trying the policy

The policy is a stand-alone Wasm module, you can invoke it in this way:

```shell
$ cat assembly/__tests__/fixtures/privileged_container.json  | wasmtime run \
              --env TRUSTED_USERS="alice" \
              --env TRUSTED_GROUPS="trusted-users,system:masters" \
              build/optimized.wasm
```

This will produce the following output:

```shell
{"accepted":true,"message":""}
```

# Testing

Unit tests can be run via:

```shell
$ make test
```

# Benchmark

Some benchmarks can be run via this command:

```
$ make bench
```

The benchmarks rely on [hyperfine](https://github.com/sharkdp/hyperfine). The
make file will automatically install it using [Cargo](https://doc.rust-lang.org/cargo/).
