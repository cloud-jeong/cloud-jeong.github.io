

```
curl -O -L  https://github.com/projectcalico/calicoctl/releases/download/v3.16.0/calicoctl
chmod +x calicoctl

export DATASTORE_TYPE=kubernetes
export KUBECONFIG=/path/to/kubeconfig

calicoctl node status
calicoctl get ippool -o wide
NAME                  CIDR           NAT    IPIPMODE   VXLANMODE   DISABLED   SELECTOR   
default-ipv4-ippool   10.32.0.0/12   true   Always     Never       false      all()   

calicoctl get wep --all-namespaces
```



```
calicoctl get ippool default-ipv4-pool -o yaml > ipv4-pool.yaml

vi ipv4-pool.yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ipv4-ippool
spec:
  cidr: 192.168.0.0/16
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Always
calicoctl apply -f ipv4-pool.yaml

calicoctl delete ippool default-ipv4-pool

calicoctl get ippool -o wide
```



```
kubectl get nodes node1 -o yam1 > node1.yaml
kubectl get nodes node2 -o yam1 > node2.yaml
kubectl get nodes node3 -o yam1 > node3.yaml

vi node1.yaml (node2.yaml, node3.yaml)
apiVersion: v1
kind: Node
metadata:
  name: node1
spec:
  podCIDR: 192.168.0.0/16
  podCIDRs:
  - 192.168.0.0/16

kubectl delete node node1; k apply -f node1.yaml
kubectl delete node node2; k apply -f node1.yaml
kubectl delete node node3; k apply -f node1.yaml
```



```
< change sericeSubnet >
https://www.devops.buzz/public/kubeadm/change-servicesubnet-cidr

전체 노드에서 백업
cp -R /etc/kubernetes /etc/kubernetes_bak
cd /prod/cri/work

mkdir work
cd work
kubectl get cm -o yaml -n kube-system kubeadm-config > kubeadm.yaml
vi kubeadm.yaml
...
serviceSubnet: 172.20.0.0/16
...

## master1,2,3에서에서 10.96.0.1 -> 172.20.0.1로 변경한 후 kube-apiserver 인증서 재생성
vi /opt/kubernetes/pki/common-openssl.conf
...
IP.3 = 172.20.0.1
...

openssl req -new -key /etc/kubernetes/pki/apiserver.key -subj '/CN=kube-apiserver' |
openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out ./apiserver.crt -days 29012 -extensions v3_req_apiserver -extfile /opt/kubernetes/pki/common-openssl.conf

cp ./apiserver.crt /etc/kubernetes/pki/apiserver.crt

ps -ef | grep kube-apiserver
kill {pid}

## 전체 노드에서 kubelet 재시작
systemctl daemon-reload; systemctl restart kubelet

kubectl -n kube-system delete service kube-dns
kubeadm upgrade apply --config ./kubeadm.yaml --certificate-renewal=false

## kubernetes 서비스 삭제
kubectl -n default delete service kubernetes

[root@node1 work]# k get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   172.20.0.1   <none>        443/TCP   2s

전체 노드에서 /etc/sysconfig/kubelet에 —cluster-dns=172.20.0.10 항목 추가 후 restart
systemctl restart kubelet

## 이후 각 service에서 uid, clusterIP 삭제 저장 후 재생성
kubectl get svc -n cocktail-system -o yaml > cocktail-service.yaml
kubectl delete svc -n cocktail-system
kubectl apply -f cocktail-service.yaml
```

