---
title: Hands-on
weight: 2
---

## kubeadm으로 클러스터 부트스트랩

{{< tabs >}}
{{< tab name="Control Plane" >}}
```bash
# control plane 노드에서
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version=v1.30.0 \
  --upload-certs

# kubeconfig 적용
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# CNI 설치 (예: Calico)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```
{{< /tab >}}
{{< tab name="Worker Join" >}}
```bash
# kubeadm init 출력에 표시된 join 명령어를 워커 노드에서 실행
sudo kubeadm join 10.0.0.1:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# 노드가 정상 조인되었는지 확인
kubectl get nodes -o wide
```
{{< /tab >}}
{{< /tabs >}}

## Cluster API로 워크로드 클러스터 생성

```bash
# management cluster에 CAPI 컴포넌트 설치 (AWS provider 예시)
clusterctl init --infrastructure aws

# 워크로드 클러스터 매니페스트 생성
clusterctl generate cluster my-cluster \
  --kubernetes-version v1.30.0 \
  --control-plane-machine-count=3 \
  --worker-machine-count=5 \
  > my-cluster.yaml

kubectl apply -f my-cluster.yaml

# 클러스터 프로비저닝 상태 확인
clusterctl describe cluster my-cluster
```

## 안전한 업그레이드 절차

```bash
# 1. etcd 백업 (가장 먼저, 항상)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 2. 업그레이드 가능 여부 사전 점검
sudo kubeadm upgrade plan

# 3. 첫 control plane 노드 업그레이드
sudo apt-mark unhold kubeadm && sudo apt-get update && sudo apt-get install -y kubeadm=1.30.0-1.1
sudo kubeadm upgrade apply v1.30.0

# 4. 나머지 control plane 노드
sudo kubeadm upgrade node

# 5. kubelet/kubectl 업그레이드 (각 노드, 한 노드씩 cordon → drain → 업그레이드 → uncordon)
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
sudo apt-get install -y kubelet=1.30.0-1.1 kubectl=1.30.0-1.1
sudo systemctl daemon-reload && sudo systemctl restart kubelet
kubectl uncordon <node>
```

{{< callout type="info" >}}
워커 노드는 한 번에 1개씩 drain → 업그레이드 → uncordon 한다. 여러 노드를 동시에 drain하면 PodDisruptionBudget이 있어도 가용 capacity가 급격히 줄어 워크로드 전체가 영향을 받을 수 있다.
{{< /callout >}}

## 트러블슈팅: 업그레이드 후 일부 Pod가 Pending에서 멈춤

**현상**: control plane을 1.29 → 1.30으로 업그레이드한 직후, 신규 배포된 Pod 일부가 `Pending` 상태에서 스케줄링되지 않는다.

**의심되는 원인(가설)**: 워커 노드의 kubelet이 아직 1.28에 머물러 있는데, 새로 배포한 워크로드가 1.30에서만 지원하는 API 필드(예: 최신 `resources.claims`)를 사용하고 있어 구버전 kubelet이 Pod spec을 거부하고 있을 가능성.

**확인할 로그/명령어**:
```bash
# 노드별 kubelet 버전과 apiserver 버전 스큐 확인
kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion

# 스케줄링 실패 사유 확인
kubectl describe pod <pending-pod> | grep -A 10 Events

# kubelet 자체 로그 (대상 노드에서)
sudo journalctl -u kubelet -n 100 --no-pager
```

**조치**: 워커 노드 kubelet을 control plane과 같은 마이너 버전(또는 최대 -2 이내)으로 롤링 업그레이드한다. 동시에 deprecated/신규 API 필드를 사용하는 워크로드는 모든 노드의 kubelet이 해당 버전으로 올라간 뒤에 배포하도록 배포 순서를 조정한다.

## 운영 체크리스트

- [ ] etcd 백업이 업그레이드 전에 자동/수동으로 수행되는가
- [ ] 업그레이드는 control plane → 애드온(CNI/CoreDNS) → 워커 노드 순서를 지키는가
- [ ] 워커 노드는 한 번에 1개씩 drain하는가 (PDB와 함께 검증)
- [ ] `kubectl deprecations` 또는 `pluto`/`kubent` 같은 도구로 제거 예정 API 사용 여부를 사전 점검했는가
- [ ] 클러스터 자체(IaC) 변경과 클러스터 내부 리소스(GitOps) 변경의 책임 경계가 명확한가
