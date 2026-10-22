# How WebRTC reaches a pod on a LAN Kubernetes cluster

Research for [#5](https://github.com/UnstoppableMango/mangomu/issues/5), part of the v1 map ([#1](https://github.com/UnstoppableMango/mangomu/issues/1)).

**Question.** A Go (Pion) WebRTC peer runs in one long-lived pod on thecluster. How should it expose ICE/UDP to a browser on the same LAN, or over Tailscale, with no TURN server?

**Answer.** Run Pion's UDP mux on one fixed port. Put a `type: LoadBalancer` UDP Service with `externalTrafficPolicy: Local` in front of it. Use Pion's address rewrite rules (the replacement for `SetNAT1To1IPs`) so the only host candidates Pion advertises are that Service's stable IPs. Those are the LAN LoadBalancer IP and, if wanted, a Tailscale operator ingress IP for the same Service. Do the signaling as plain HTTP POSTs of the SDP offer and answer, with no trickle ICE, through the HTTPRoute the charts already use. Keep `hostNetwork` as the fallback if thecluster has no LoadBalancer implementation that handles UDP.

Versions read: `github.com/pion/webrtc/v4` v4.2.25, `github.com/pion/ice/v4` v4.4.8, `tailscale.com` v1.104.2 (all from the Go module proxy), Gateway API v1.6.2 (the version `unmango/charts` pins), and `kubernetes/website` and `kubernetes/api` at `main`.

## Background facts

### Pion

- **Single-port UDP mux.** `SettingEngine.SetICEUDPMux` "allows ICE traffic to come through a single UDP port, drastically simplifying deployments where ports will need to be opened/forwarded." The mux must be started before PeerConnections are created ([`settingengine.go`](https://github.com/pion/webrtc/blob/v4.2.25/settingengine.go)). `ice.NewMultiUDPMuxFromPort(port)` listens on every interface on that port. `NewUDPMuxDefault` logs a warning if it is given an unspecified (`0.0.0.0`) address and tells you to use `NewMultiUDPMuxFromPort` instead ([`udp_mux.go`](https://github.com/pion/ice/blob/v4.4.8/udp_mux.go), [`udp_mux_multi.go`](https://github.com/pion/ice/blob/v4.4.8/udp_mux_multi.go)).
- **NAT 1:1 is deprecated in favour of address rewrite rules.** `SetNAT1To1IPs` is marked `Deprecated: Use SetICEAddressRewriteRules instead`, and the two cannot be combined (`errAddressRewriteWithNAT1To1`). A rule (`ICEAddressRewriteRule`) has these fields:
  - `External []string`: one or more IPs to advertise. The ICE side keeps a slice of catch-all IPs, so a single rule can publish several.
  - `Local`, `Iface`, `CIDR`, `Networks`: optional scoping.
  - `AsCandidateType`: host, srflx or relay.
  - `Mode`: replace or append. The default is replace for host candidates and append for srflx.
  - `OriginalPort` and `NewPort`: rewrite the advertised port.

  Sources: [`settingengine.go`](https://github.com/pion/webrtc/blob/v4.2.25/settingengine.go), [`icegatherer.go`](https://github.com/pion/webrtc/blob/v4.2.25/icegatherer.go), [`external_ip_mapper.go`](https://github.com/pion/ice/blob/v4.4.8/external_ip_mapper.go).
- **Rewrite applies to mux candidates.** In `gatherCandidatesLocalUDPMux`, each local mux address goes through `applyHostRewriteForUDPMux`, and one host candidate is emitted per resulting IP. After that, `rewriteCandidatePort` runs ([`gather.go`](https://github.com/pion/ice/blob/v4.4.8/gather.go)). So "advertise LB-IP:port and tailnet-IP:port, never the pod IP" is one rule. The port rewrite is what makes a NodePort usable, because the node port differs from the container port.
- **Port range alternative.** `SetEphemeralUDPPortRange(min, max)` limits which ports host candidates bind. Left at 0, Pion uses 1–65535 ([`settingengine.go`](https://github.com/pion/webrtc/blob/v4.2.25/settingengine.go)).
- **Browser mDNS candidates do not matter here.** Pion's default mDNS mode is `MulticastDNSModeQueryOnly` ([`agent.go`](https://github.com/pion/ice/blob/v4.4.8/agent.go), [`mdns.go`](https://github.com/pion/ice/blob/v4.4.8/mdns.go)). A browser's `.local` candidate can't be resolved from inside a pod. That is fine as long as the browser can reach the server's candidate: the browser's connectivity check arrives, and Pion learns the browser's address from it as a peer-reflexive candidate. All the options below are designed around browser→pod reachability, so this holds for every one of them.

### Kubernetes

- **Services have no port ranges.** `ServicePort` has a single `Port`, `TargetPort` and `NodePort` ([`core/v1/types.go`](https://github.com/kubernetes/api/blob/master/core/v1/types.go)). Exposing a range through a Service means listing every port.
- **NodePorts** are allocated from `--service-node-port-range`, which defaults to 30000–32767 ([Service docs](https://kubernetes.io/docs/concepts/services-networking/service/)).
- **LoadBalancer and UDP.** `MixedProtocolLBService` has been on by default since v1.24, so one LB Service can carry TCP and UDP ports. However, "the set of protocols that can be used for load balanced Services is defined by your cloud provider", meaning the LB implementation. `.spec.loadBalancerIP` has been deprecated since v1.24. Pinning an IP is done through an implementation-specific annotation or `loadBalancerClass` ([Service docs](https://kubernetes.io/docs/concepts/services-networking/service/)).
- **Source IP.** With `externalTrafficPolicy: Cluster`, a node may forward to a pod on another node and SNAT the packet to its own IP. With `Local`, the client source IP is preserved and only nodes with a local endpoint take traffic ([Using Source IP](https://kubernetes.io/docs/tutorials/services/source-ip/)). ICE works under SNAT because responses follow conntrack, but `Local` removes a hop and keeps the browser's real address visible to Pion's mux.
- **hostNetwork** puts the pod in the node's network namespace. When it is set, `hostPort` must equal `containerPort` ([`core/v1/types.go`](https://github.com/kubernetes/api/blob/master/core/v1/types.go)). The Pod Security Standards' Baseline profile forbids `spec.hostNetwork` and restricts `hostPort` ([Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)).

### Gateway API

- **WebSocket** through an HTTPRoute is the `HTTPRouteBackendProtocolWebSocket` feature. It is in the standard channel but is *extended* conformance, so implementations may not support it. It is selected by setting the backend Service port's `appProtocol: kubernetes.io/ws` ([`pkg/features/httproute.go`](https://github.com/kubernetes-sigs/gateway-api/blob/v1.6.2/pkg/features/httproute.go), [`core/v1/types.go`](https://github.com/kubernetes/api/blob/master/core/v1/types.go)).
- **UDPRoute** has a `v1` version and is in Gateway API v1.6.2's `standard-install.yaml` (`v1` served and stored, `v1alpha2` not served; [`pkg/features/udproute.go`](https://github.com/kubernetes-sigs/gateway-api/blob/v1.6.2/pkg/features/udproute.go)). Whether thecluster's Gateway implementation supports it is a separate question. It would also put a proxy in the media path for no benefit over a Service.

### Tailscale

These facts come from the source, because `tailscale.com/kb` was not reachable from this session.

- **Operator ingress.** The operator proxies a Service that has `loadBalancerClass: tailscale`, or the `tailscale.com/expose: "true"` annotation ([`cmd/k8s-operator/svc.go`](https://github.com/tailscale/tailscale/blob/v1.104.2/cmd/k8s-operator/svc.go), [`operator.go`](https://github.com/tailscale/tailscale/blob/v1.104.2/cmd/k8s-operator/operator.go)). The proxy pod runs with `TS_USERSPACE=false` and a privileged init container that turns on IP forwarding ([`deploy/manifests/proxy.yaml`](https://github.com/tailscale/tailscale/blob/v1.104.2/cmd/k8s-operator/deploy/manifests/proxy.yaml)). It is configured with `TS_DEST_IP` set to the Service's cluster IP ([`sts.go`](https://github.com/tailscale/tailscale/blob/v1.104.2/cmd/k8s-operator/sts.go)). containerboot then installs `iptables -t nat -I PREROUTING -d <tailnet IP> -j DNAT --to-destination <cluster IP>`. The rule has no `-p` match, so **UDP is forwarded as well as TCP** ([`util/linuxfw/iptables_runner.go`](https://github.com/tailscale/tailscale/blob/v1.104.2/util/linuxfw/iptables_runner.go), [`cmd/containerboot/forwarding.go`](https://github.com/tailscale/tailscale/blob/v1.104.2/cmd/containerboot/forwarding.go)).
- **Sidecar.** containerboot defaults to `TS_USERSPACE=true` ([`cmd/containerboot/settings.go`](https://github.com/tailscale/tailscale/blob/v1.104.2/cmd/containerboot/settings.go)). In userspace mode the pod gets no `tailscale0` interface, so Pion cannot bind a candidate on the tailnet IP. A sidecar is only useful in kernel mode, which needs `NET_ADMIN` and `/dev/net/tun`. In that mode Pion's ordinary interface enumeration finds the 100.x address with no rewrite needed.

### Browser

`getUserMedia` (mic uplink) only works in a secure context, meaning HTTPS or `localhost`. In an insecure context `navigator.mediaDevices` is `undefined` ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia)). The signaling page therefore has to be served over HTTPS if the mic is ever used, so the HTTPRoute's Gateway listener needs TLS for the LAN or tailnet hostname.

## Options compared

### 1. `hostNetwork: true`

- **Reachable IP.** Pion enumerates the node's interfaces directly: the LAN IP, plus the node's `tailscale0` IP if Tailscale runs on the node. No rewrite is needed, but an `InterfaceFilter` or `IPFilter` is advisable to drop CNI and bridge interfaces.
- **Pod restarts.** Candidates are gathered at runtime, so they are always correct. The pod may be rescheduled to a different node, which changes the IP unless it is pinned with a `nodeSelector`. The fixed UDP port can clash with other host-network pods or node services.
- **Signaling.** Use a normal ClusterIP Service and HTTPRoute. This works because the pod still answers on its (host) IP.
- **Cost.** Violates PSS Baseline and needs `dnsPolicy: ClusterFirstWithHostNet`. It is also the least "cloud-native" option. It is the simplest thing that works regardless of the LB implementation.

### 2. Fixed UDP port with UDP mux behind a Service (recommended)

- **Reachable IP.**
  - **LoadBalancer:** the Service's external IP, given to Pion with `ICEAddressRewriteRule{External: [lbIP], AsCandidateType: host}`. The mode defaults to replace, so the pod IP is never advertised. With `externalTrafficPolicy: Local`, the browser's real address reaches the mux.
  - **NodePort:** a node IP plus `NewPort: <nodePort>`. This is worse, because the advertised node is fixed but the pod may not be on it, unless the policy is `Cluster` (which SNATs) or the pod is pinned.
- **Pod restarts.** The Service and its IP outlive the pod, so the advertised address is stable. Setting the IP through the LB implementation's own annotation makes it stable across re-creation of the Service too. Any in-flight PeerConnection dies with the pod regardless, and the browser has to re-signal (the ICE agent state is gone).
- **Signaling.** Use a separate ClusterIP Service for HTTP and attach it to an HTTPRoute, exactly like the existing charts. Alternatively, put a TCP port on the same LB Service (mixed protocol) for direct access.
- **Cost.** Depends on thecluster having a LoadBalancer implementation that does UDP (see the questions below). The address has to be configured in Pion. That means a Helm value, or reading the Service's `status.loadBalancer.ingress` at startup, which needs RBAC to `get` Services.

### 3. UDP port range

- **Reachable IP.** Same as option 2, with `SetEphemeralUDPPortRange`.
- **Pod restarts.** Same as option 2.
- **Signaling.** Same as option 2.
- **Cost.** A Service can't express a range, so every port must be listed (or the pod must use `hostNetwork`, which is option 1 in disguise). A range buys nothing for a single session that a mux on one port doesn't already provide. **Rejected.**

### 4. Tailscale

- **Operator ingress on the same UDP Service.** Annotate the Service, or make a second one with `loadBalancerClass: tailscale`. The proxy gets a stable tailnet IP, with its state kept in a Secret. It DNATs all protocols, UDP included, to the cluster IP. Add that 100.x IP to the rewrite rule's `External` list, so Pion advertises both the LAN IP and the tailnet IP as host candidates, and ICE picks whichever the browser can reach. The cost is one extra hop through the proxy pod, which may be on another node, and the latency that adds. The proxy is a separate pod, so mangomu pod restarts don't touch it. Signaling over the tailnet can go through the same operator, either as a Tailscale Ingress or as a TCP port on the exposed Service.
- **Kernel-mode sidecar.** Pion sees `tailscale0` directly, giving a direct path with no extra hop. The cost is that the mangomu pod needs `NET_ADMIN`, the TUN device and an auth key, which couples the app pod to Tailscale.
- **Subnet router: no Kubernetes change.** If a Tailscale subnet router already advertises the LAN, tailnet clients reach the LAN LB IP directly, and option 2 alone covers both cases.

## Recommendation

1. **Media.** Pion `NewMultiUDPMuxFromPort(<fixed port>)` sits behind a single `type: LoadBalancer` Service with `protocol: UDP`, `externalTrafficPolicy: Local`, and a pinned IP through the LB implementation's annotation. Pion advertises only that IP as a host candidate, via `SetICEAddressRewriteRules`. The `External` list comes from a Helm value, so a tailnet IP can be added.
2. **Tailscale.** Prefer an existing subnet router covering the LAN, which needs no work. If there isn't one, expose the same Service through the Tailscale operator and append its tailnet IP to `External`. Avoid the sidecar.
3. **Signaling.** Exchange the offer and answer over HTTP POST with no trickle ICE, WHEP-style. Gathering from a mux with a fixed rewrite is effectively instant. Serve it through a ClusterIP Service and an HTTPRoute (`parentRefs` and `hostnames` as values, like `deemix` and `filebrowser`). This avoids depending on the extended-conformance WebSocket feature. Serve over HTTPS so mic capture works.
4. **Fallback.** If thecluster has no UDP-capable LoadBalancer, use `hostNetwork: true` pinned to one node with a `nodeSelector`, filtering interfaces in Pion. No rewrite is needed.

## Facts about thecluster found in readable repos

- Its LAN domain is `thecluster.lan`. CI pulls from `https://ncps.thecluster.lan` (`unmango/charts` `.github/workflows/ci.yml`). The source repo is `UnstoppableMango/the-cluster` (`vcs/github/index.ts`), which this research did not read.
- `unmango/charts` installs the Gateway API v1.6.2 standard CRDs, and its charts expose HTTP only through an `HTTPRoute` with user-supplied `parentRefs`. No Gateway or GatewayClass name for thecluster appears anywhere.
- Several charts (`deluge`, `qbittorrent`, `unifi`) offer `type: LoadBalancer` Services with UDP ports and a `loadBalancerIP` value. This hints that UDP LoadBalancers are used on thecluster, but no implementation is named.
- Nothing in `unmango/charts`, `unmango/containers` or the `vcs` Pulumi repo names the CNI, the LB implementation (MetalLB, Cilium LB-IPAM/L2, kube-vip, k3s ServiceLB) or a Tailscale setup.

## To ask Erik

1. Which LoadBalancer implementation does thecluster run, does it handle UDP, and how is a Service's IP pinned (annotation or pool)? Does it honour `externalTrafficPolicy: Local` (for example, an L2 announcement from the node that hosts the pod)?
2. Which CNI is it, and does it replace kube-proxy? This affects SNAT and source-IP behaviour.
3. How does Tailscale reach thecluster today: a subnet router advertising the LAN, the Kubernetes operator, or Tailscale on the nodes?
4. What is the Gateway's name and namespace for `parentRefs`, which implementation is behind it, and does it terminate TLS for `*.thecluster.lan` (or a tailnet name)?
5. Does Pod Security admission block `hostNetwork` in the target namespace, in case the fallback is needed?

## Sources that could not be read

- `https://kubernetes.io/docs/concepts/services-networking/service/` could not be fetched. The same page was read from its source in `kubernetes/website`.
- `tailscale.com/kb/*` was refused by the egress proxy. The Tailscale claims rest on the `tailscale.com` v1.104.2 source instead.
