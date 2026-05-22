# Baseline A - Fase 0: congelare baseline A

## Contesto

Questa nota contiene lo stato del cluster prima della creazione della baseline B.

- Host di accesso: `192.168.100.10`
- Sistema operativo: Alpine Linux
- Cluster: k3s
- Obiettivo: fotografare lo stato iniziale di nodi, pod, servizi e ingress.

## Accesso al nodo

```text
root@192.168.100.10's password:
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.
```

## 1. Stato dei nodi

Comando:

```bash
kubectl get nodes -o wide
```

Output:

```text
NAME            STATUS   ROLES           AGE   VERSION        INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
alpine-node-2   Ready    <none>          22d   v1.34.6+k3s1   192.168.100.30   <none>        Alpine Linux v3.23   6.18.22-0-lts    containerd://2.2.2-bd1.34
alpinelinux     Ready    control-plane   34d   v1.34.6+k3s1   192.168.100.10   <none>        Alpine Linux v3.23   6.18.22-0-lts    containerd://2.2.2-bd1.34
```

### Osservazioni

- Il cluster ha 2 nodi.
- `alpinelinux` è il nodo control-plane.
- Entrambi i nodi sono in stato `Ready`.

## 2. Stato dei pod

Comando:

```bash
kubectl get pods -A -o wide
```

Output:

```text
NAMESPACE             NAME                                        READY   STATUS      RESTARTS        AGE     IP            NODE            NOMINATED NODE   READINESS GATES
cattle-fleet-system   fleet-agent-75f5868dff-rtl42                1/1     Running     127 (21s ago)   23d     10.42.0.201   alpinelinux     <none>           <none>
cattle-system         cattle-cluster-agent-66b7cf66db-4fzfb       0/1     Error       144 (15s ago)   23d     10.42.0.194   alpinelinux     <none>           <none>
cattle-system         cattle-cluster-agent-66b7cf66db-rzxjf       0/1     Error       59 (15s ago)    22d     10.42.0.196   alpinelinux     <none>           <none>
cattle-system         rancher-webhook-db77694d7-4qgsg             0/1     Running     153 (21s ago)   23d     10.42.0.197   alpinelinux     <none>           <none>
cattle-system         system-upgrade-controller-65d9b4b8b-pnk2c   1/1     Running     131 (21s ago)   23d     10.42.0.205   alpinelinux     <none>           <none>
kube-system           coredns-76c974cb66-tvkls                    1/1     Running     196 (22s ago)   34d     10.42.0.193   alpinelinux     <none>           <none>
kube-system           helm-install-traefik-6zh9p                  0/1     Completed   1               34d     <none>        alpinelinux     <none>           <none>
kube-system           helm-install-traefik-crd-b56z6              0/1     Completed   0               34d     <none>        alpinelinux     <none>           <none>
kube-system           local-path-provisioner-8686667995-947p4     1/1     Running     22 (21s ago)    34d     10.42.0.199   alpinelinux     <none>           <none>
kube-system           metrics-server-c8774f4f4-fgcdx              0/1     Running     197 (21s ago)   34d     10.42.0.200   alpinelinux     <none>           <none>
kube-system           svclb-traefik-6d0a4775-kb9nk                2/2     Running     44 (21s ago)    34d     10.42.0.192   alpinelinux     <none>           <none>
kube-system           svclb-traefik-6d0a4775-lph22                2/2     Running     50 (7s ago)     22d     10.42.1.133   alpine-node-2   <none>           <none>
kube-system           traefik-c5c8bf4ff-4qwmf                     1/1     Running     210 (21s ago)   34d     10.42.0.198   alpinelinux     <none>           <none>
prototype             api-backend-787cf69f8c-89plf                1/1     Running     8 (21s ago)     4d17h   10.42.0.195   alpinelinux     <none>           <none>
prototype             api-backend-787cf69f8c-nxkqb                1/1     Running     10 (7s ago)     4d17h   10.42.1.134   alpine-node-2   <none>           <none>
prototype             data-generator-7bbc645ff8-glqmp             1/1     Running     9 (21s ago)     5d18h   10.42.0.204   alpinelinux     <none>           <none>
prototype             logic-simulator-7bc8997f99-q2vhn            1/1     Running     9 (21s ago)     5d18h   10.42.0.203   alpinelinux     <none>           <none>
prototype             web-ui-5c66f4b87b-ftdb2                     1/1     Running     10 (7s ago)     4d17h   10.42.1.135   alpine-node-2   <none>           <none>
prototype             web-ui-5c66f4b87b-lbx58                     1/1     Running     8 (21s ago)     4d17h   10.42.0.202   alpinelinux     <none>           <none>
```

### Osservazioni

- I pod applicativi principali sono nel namespace `prototype`.
- `api-backend` e `web-ui` hanno repliche distribuite su entrambi i nodi.
- `data-generator` e `logic-simulator` risultano sul nodo `alpinelinux`.
- Alcuni componenti Rancher risultano non sani o instabili:
  - `cattle-cluster-agent`: `Error`
  - `rancher-webhook`: `0/1`
  - `metrics-server`: `0/1`
- Sono presenti molti restart nei namespace `cattle-system` e `kube-system`.

## 3. Stato dei servizi

Comando:

```bash
kubectl get svc -A
```

Output:

```text
NAMESPACE       NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP                     PORT(S)                      AGE
cattle-system   cattle-cluster-agent   ClusterIP      10.43.3.32      <none>                          80/TCP,443/TCP               23d
cattle-system   rancher-webhook        ClusterIP      10.43.198.129   <none>                          443/TCP                      23d
default         kubernetes             ClusterIP      10.43.0.1       <none>                          443/TCP                      34d
kube-system     kube-dns               ClusterIP      10.43.0.10      <none>                          53/UDP,53/TCP,9153/TCP       34d
kube-system     metrics-server         ClusterIP      10.43.189.48    <none>                          443/TCP                      34d
kube-system     traefik                LoadBalancer   10.43.33.113    192.168.100.10,192.168.100.30   80:30437/TCP,443:30142/TCP   34d
prototype       api-backend            ClusterIP      10.43.198.200   <none>                          5000/TCP                     34d
prototype       data-generator         ClusterIP      10.43.58.91     <none>                          5000/TCP                     29d
prototype       logic-simulator        ClusterIP      10.43.181.209   <none>                          5000/TCP                     34d
prototype       web-ui                 ClusterIP      10.43.33.31     <none>                          80/TCP                       34d
```

### Osservazioni

- Traefik è esposto come `LoadBalancer`.
- Gli external IP di Traefik sono:
  - `192.168.100.10`
  - `192.168.100.30`
- I servizi applicativi nel namespace `prototype` sono `ClusterIP`.

## 4. Stato degli ingress

Comando:

```bash
kubectl get ingress -A
```

Output non ancora riportato nella nota originale.

## Riassunto baseline A

| Elemento | Stato |
| --- | --- |
| Numero nodi | 2 |
| Control-plane | `alpinelinux` / `192.168.100.10` |
| Worker | `alpine-node-2` / `192.168.100.30` |
| Stato nodi | Entrambi `Ready` |
| Ingress controller | Traefik |
| Traefik external IP | `192.168.100.10`, `192.168.100.30` |
| Namespace applicativo | `prototype` |
| Pod distribuiti su entrambi i nodi | `api-backend`, `web-ui` |
| Pod solo su control-plane | `data-generator`, `logic-simulator` |
| Problemi visibili | Rancher agent/webhook e metrics-server non completamente sani |

