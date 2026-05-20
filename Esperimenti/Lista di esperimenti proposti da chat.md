Questa lista è collegata al fle [[Esperimenti]]

| ID  | Esperimento                    | Cosa osserva                                                      | Stato |
| --- | ------------------------------ | ----------------------------------------------------------------- | ----- |
| E1  | Delete `web-ui` pod            | k3s ricrea il pod tramite Deployment/ReplicaSet                   | fatto |
| E2  | Worker node shutdown / VM3 off | k3s rischedula i pod sul nodo disponibile, con possibile downtime | fatto |

---

|ID|Esperimento|Fault iniettato|Cosa dimostra|Soluzione/meccanismo / Note|
|---|---|---|---|---|
|E3|`api-backend` unhealthy with livenessProbe|`/health` restituisce errore 500|k3s può riavviare un container unhealthy se la livenessProbe è configurata|`livenessProbe`|
|E4|`api-backend` not ready with readinessProbe|`/ready` o `/health` fallisce temporaneamente|k3s può togliere il pod dal traffico senza ucciderlo|`readinessProbe`|
|E5|Image missing / wrong image tag|Deployment usa immagine inesistente o non disponibile sul nodo|k3s vuole creare il pod, ma non può farlo senza immagine|Registry locale / image distribution|
|E6|Insufficient resources|Richieste CPU/RAM troppo alte|k3s non può schedulare pod se non c’è capacità|Capacity planning, resource requests/limits|
|E7|Traefik / Ingress pod failure|Kill del pod Traefik|Verifica se l’entrypoint esterno recupera automaticamente|k3s system Deployment/DaemonSet + Traefik|
|E8|Wrong Ingress backend/port|Ingress punta al service o porta sbagliata|Pod sani ma accesso esterno rotto|Validazione manifest / config review|
|E9|Wrong Service selector|Service selector non matcha i label dei pod|I pod sono Running ma il Service non ha endpoint|Ottimo per configuration fault|
|E10|All replicas on same node|Assenza/rimozione di anti-affinity, poi node failure|Le repliche aiutano solo se distribuite bene|Utile per parlare di placement|
|E11|Anti-affinity/topology spread|Aggiungere regole di distribuzione|Confronta distribuzione prima/dopo|Più configurazione, ma molto sensato|
|E12|Memory limit exceeded / OOMKilled|Container consuma più memoria del limite|Kubernetes riavvia o termina il container|Richiede modo per stressare memoria|
|E13|CPU stress|Container o nodo sotto carico CPU|Misura degrado/latency, non solo crash|Più difficile da misurare bene|
|E14|CoreDNS failure|Scale down/delete CoreDNS|Rompe service discovery interno|Avanzato, attenzione perché impatta tutto|
|E15|Rancher VM down|Spegnere VM2/Rancher|Rancher non è nel runtime critical path|Utile come nota, non esperimento centrale|