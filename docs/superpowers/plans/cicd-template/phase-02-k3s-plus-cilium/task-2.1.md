# Task 2.1: k3s 설치 (flannel/traefik 비활성화)

**Phase**: 2 (k3s + Cilium)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 2.1' 섹션

---


**Files:** (호스트 — git에 들어가지 않음)
- VM 안: `/etc/rancher/k3s/config.yaml`

- [ ] **Step 1: VM 안에서 k3s install (Cilium 위해 flannel/traefik 끄기)**

```bash
orb shell macmini
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--flannel-backend=none --disable-network-policy --disable=traefik --disable=servicelb --write-kubeconfig-mode=644" sh -
```

- [ ] **Step 2: kubeconfig를 호스트로 복사**

```bash
# VM 안에서:
sudo cat /etc/rancher/k3s/k3s.yaml > /tmp/k3s.yaml
sudo chmod 644 /tmp/k3s.yaml
exit

# 호스트(macOS)에서:
mkdir -p ~/.kube
orb cp macmini:/tmp/k3s.yaml ~/.kube/macmini.kubeconfig
# 서버 주소를 VM IP로 (orb info macmini로 확인)
VM_IP=$(orb info macmini --format json | jq -r '.ip4')
sed -i '' "s/127.0.0.1/${VM_IP}/" ~/.kube/macmini.kubeconfig
export KUBECONFIG=~/.kube/macmini.kubeconfig
echo "export KUBECONFIG=~/.kube/macmini.kubeconfig" >> ~/.zshrc
```

- [ ] **Step 3: 노드 NotReady 확인 (CNI 미설치)**

```bash
kubectl get nodes
```

기대: `NotReady` (CNI 없음, 의도된 상태). 다음 단계에서 Cilium 설치하면 Ready.
