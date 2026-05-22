Nel meeting di oggi è importante capire se quello che sto facendo torna utile a loro in qualche maniera o no.
Di cosa dovrei discutere:
- <font color="#ff0000">diagramma quindi architettura</font> (VM1,VM2 ecc...)
- tabella esperimenti:

| Experiment | Fault                           | Result / Expected result                                                                                                           |
| ---------- | ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| E1         | `web-ui` pod deletion           | k3s automatically recreates the pod.<br>Esperimento eseguito                                                                       |
| E2         | Worker node failure / VM3 off   | k3s reschedules the pods, but there is a short instability window and possible cleanup issues.                                     |
| E3         | Main node crash / VM1 off       | Both `.10` and `.30` endpoints timed out for approximately 91–92 seconds. This shows that the main node is still a critical point. |
| E4         | Traefik / Ingress pod failure   | To observe whether external access to the Web UI/API is interrupted and whether k3s recreates Traefik automatically.               |
| E5         | Replica distribution analysis   | To check whether `web-ui`, `api-backend`, and `logic-simulator` replicas are really distributed across VM1 and VM3.                |
| E6         | Application health failure      | To test whether k3s restarts `api-backend` when `/health` fails and a livenessProbe is configured.                                 |
| E7         | CPU overload / no response      | To observe what happens when a pod or node becomes overloaded and requests start timing out.                                       |
| E8         | Image missing / wrong image tag | To show that k3s cannot start a pod if the required image is missing or unavailable on the target node.                            |

## Per l'E3 è importante notare la seguente cosa:

1.  Con 2 server nodes il <font color="#00b050">quorum</font> ( *serve una maggioranza di nodi d'accrodo per prendere decisioni valide* ) è 2, quindi se ne perdi 1 non hai più maggioranza; con 3 server nodes il quorum è 2, quindi puoi perdere 1 nodo e gli altri 2 mantengono il cluster vivo.
	- 3 nodi non "garantiscono" che tutto funzioni sempre,
	- però,
	- 3 nodi permettono al control-plane di sopravvivere alla perdita di 1 nodo.

2. Anche se il control-plane resta vivo, devi avere anche un endpoint stabile tipo <font color="#00b050">VIP</font>( *indirizzo IP "finto/stabile" che non appartiene rgidamente a una sola macchina* ) /load balancer, altrimenti se continuo ad usare direttamente 192.168.100.10 e VM1 cade, il client perde comunque quell'indirizzo. Il file infatti distingue tra control-plane HA e application/service-entry HA.
	- Sì, 3 nodi sono la soluzione corretta per un k3s HA cluster con embedded etcd. 
	  2 nodi non bastano davvero, perché se uno cade perdi il quorum. 
	  Però per avere vera disponibilità lato utente servono anche VIP/load balancer e pod distribuiti bene.

## Discussione con Peter e test di errori a loro sensati - cose importanti da portare

Un altro punto importante è testare feature che a loro possano andare bene. 

- Quella cosa del Browser per esempio è importante (la quale soluzione può essere il vIP).
-


Ridondanza sul DB

Un node crasha

C'è un servizio che gioca con i vIP


# Appunti del meeting

- 3 nodi invece che 2
- CPU overload
- Node crash - cosa succede