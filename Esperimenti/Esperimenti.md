# Table of contents

```table-of-contents
```
---

## Lista esperimenti/idee proposte con chat 

Guarda la lista su questo file --> [[Lista di esperimenti proposti da chat]]

Domande venute fuori:
1. Quali fault sono realistici per il nostro ambiente aziendale?
2. Quali fault sono facili da iniettare senza rompere troppo il cluster?
3. Quali fault danno risultati misurabili?
4. Quali fault sono già coperti da k3s?
5. Quali fault richiedono configurazione aggiuntiva?
6. Quali fault sono fuori scope ma vanno citati come limitation/future work?

Gli esperimenti possono essere classificati così alla fine:

| Categoria                        | Significato                                                   |
| -------------------------------- | ------------------------------------------------------------- |
| k3s-native recovery              | k3s recupera automaticamente, esempio pod deletion            |
| configuration-dependent recovery | k3s recupera solo se configurato bene, esempio probes         |
| operational limitation           | k3s vuole recuperare ma manca qualcosa, esempio image missing |
| architectural limitation         | k3s da solo non basta, esempio DB HA/control-plane HA         |

Quello che ho capito: è importante definire 5/7 esperimenti fondamentali, tesatrli e trarre delle conlusioni. Dopo si trova una soluzione e si testa quella soluzione. Poi si fa un comparison.

---
## Elaborazione idee/esperimenti dal meeting con Peter

### 1. Salvare la baseline #tag

I dati di come la baseline è gestita sono stati già salvati prima di fare ogni esperimento. E’ stata definita una baseline da cui partire e ora si procede con quella.

![Diagram_v4_completed.drawio.png](attachment:b3f1e48f-d02e-4613-8df8-70aef147b098:Diagram_v4_completed.drawio.png)

Informazioni in più da Chat — tabella di controllo baseline

| Cosa fare                                  | Perché                                             |
| ------------------------------------------ | -------------------------------------------------- |
| Test accesso da browser a `192.168.100.10` | baseline                                           |
| Test accesso da browser a `192.168.100.30` | capire se entrambi i nodi possono servire ingresso |
| Eseguire HTTP monitor su entrambi          | confrontare availability                           |
| `kubectl get pods -o wide`                 | vedere dove sono i pod                             |
| `kubectl get ingress -n prototype`         | vedere indirizzi ingress                           |


### Exp2 - crash worker VM2 “100.30”

| Client usa               | Node crashato    | Cosa osservare                 |
| ------------------------ | ---------------- | ------------------------------ |
| `192.168.100.10`         | `192.168.100.30` | il browser resta su main node? |
| `192.168.100.30`         | `192.168.100.30` | ovviamente perde accesso?      |
| hostname/VIP se presente | `192.168.100.30` | failover?                      |

Al 16 Maggio 2026 sto procedendo con l’esperimento.

### Note importanti tratte dai run:

#### Run01

- _Run-01:_
    
    se un pod è deloyato tutto solo su VM2, allora al crush del nodo il pod non risulterà più disponibile e non verrà ricreato.
    
    I.e. → WebUI deployato 2/2 su alpine-node-2 → crash nodo 2 → WebUI non disponibile e non replicata
    
    **Soluzione?** Modifica dello YAML del parametro
    
    ```powershell
    whenUnsatisfiable: ScheduleAnyway
      							↓
    whenUnsatisfiable: DoNotSchedule
    ```
    
    Se k3s non riesce a redistribuire il pod su entrambi i nodi allora lui non lo redistribuisce. Per evitare di avere un pod su un solo nodo durante l’assenza dell’altro nodo (i.e. 1/2 solo su VM1), allora su YAML si può intervenire aggiungendo:
    
    ```powershell
     nodeTaintsPolicy: Honor
    ```
    
    Questo fa si che quando un nodo è “NotReady” k3s lo marca come “unreacheable/not-ready” così non viene considerato come se fosse ancora schedulabile.
    
    Tuttavia, questo non permette a k3s di reschedulare i pod una volta che il nodo assente ritorna ad essere attivo. Se WebUI viene schedulato 1 su VM1 e 1 su VM2, quando VM2 va down allora il pod viene rishcedulato su VM1. Quando VM2 ritorna “online” i due pod 2/2 di WebUI sono entrambi su VM1 e il secondo pod non viene rischedulato su VM2.
    
    **Finding:** k3s can recreate pods after failure, but id does not automatically redistribute already running pods when a failed node returns. This can be adressed trhough controlled rollout restarts or architecturally through an additional component such as **Kuberneted Descheduler.**
    

#### Run02

- _Run-02:_
    
    Prima di iniziare, le configurazioni dettate nella sezione precedente (Run01) sono state applicate:
    
    - YAML modificato
        - whenUnsatisfiable: DoNotSchedule
        - nodeTaintsPolicy: Honor
- **Da verificare:** se i pod vengono schedulati su entrambi i nodi disponibili. Se un nodo va in crash o non è disponibile allora il secondo nodo verrà rischedulato sul nodo principale.
    
- **To Keep In Mind:** se abbiamo due user, tutti e due accedono allo stesso pod o a due pod diversi? → Entrambi i pod sono attivi. Le richieste vengono distribuite tra i pod disponibili, ma non è garantito che due utenti diversi vadano sempre su pod diversi. In questo modo: più pod possono servire traffico contemporaneamente.
    
    _esperimento in corso…_
    

#### Run03, 04, 05 e 06

Run eseguiti con successo e file .md fatti e segnati sulla repo.

---

### Exp3 - crash main node “100.10”

|Client usa|Node crashato|Expected|
|---|---|---|
|`192.168.100.10`|`192.168.100.10`|accesso perso|
|`192.168.100.30`|`192.168.100.10`|forse accesso parziale se workload/Traefik girano su VM3|
|Rancher|VM1 down|cluster probabilmente non gestibile via API|

[Primo esperimento di prova (da considerare out)](https://www.notion.so/Primo-esperimento-di-prova-da-considerare-out-35d71db0c4eb80b48a2ae744cfd6482b?pvs=21)

#### Run01

done

#### Run02

done

#### Run03

in progress

### Analisi al problema “VM1 crush e tutto cluster down” → qui sotto “.md” file di analisi

[K3s_Main_Node_Failure_HA_Solution_Note.md](attachment:7a84b14d-e2da-43f9-b206-7410243f9d64:K3s_Main_Node_Failure_HA_Solution_Note.md)

- **Le considerazioni:** _2 Nodi non bastano → servono 3 nodi_

---

### Exp4 - Traefik pod failure

|Fault|Cosa osserva|
|---|---|
|delete/kill Traefik pod|se k3s lo ricrea e quanto dura l’interruzione esterna|

Comandi:

```powershell
kubectl get pods -n kube-system | grep traefik
kubectl delete pod <traefik-pod-name> -n kube-system
```

Metriche:

```powershell
HTTP failures
downtime
time until Traefik pod Running/Ready
```

### Exp5 - CPU overload / no response

Due varianti:

- Variante semplice: overload pod
    
    - Stressi api-backend o un container apposito e osservi:
        
        ```powershell
        latency aumenta?
        timeout?
        liveness/readiness reagiscono?
        altri componenti vengono impattati?
        ```
        
- Variante forte: overload del nodo
    
    - Stressi CPU su VM1 o VM3
        
        ```powershell
        tutto il nodo degrada?
        Traefik rallenta?
        UI risponde male?
        scheduler/cluster resta reattivo
        ```
        

### Exp6 - replica distribution

|Domanda|Comando|
|---|---|
|Dove sono distribuite le repliche?|`kubectl get pods -n prototype -o wide`|
|Due repliche stanno su nodi diversi?|controllare `NODE`|
|Se un nodo cade, l’altra replica basta?|test HTTP|
|Kubernetes le distribuisce automaticamente?|non sempre|
|Serve anti-affinity/topology spread?|probabilmente sì|

Esperimento:

1. osserva distribuzione attuale;
2. forza rollout/restart;
3. vedi se Kubernetes distribuisce sempre bene;
4. aggiungi eventualmente anti-affinity/topologySpread;
5. confronta prima/dopo.

Conclusione possibile:

> Replicas improve availability only if they are distributed across failure domains. Without placement rules, Kubernetes may schedule replicas in a way that is not ideal for node failure tolerance.

## Tabella esperimenti aggiornata

|ID|Esperimento|Fault|Domanda principale|Categoria|
|---|---|---|---|---|
|EXP2|Worker node crash|VM2 down|servizio resta disponibile da `.10`?|node failure|
|EXP3|Main node crash|VM1 down|browser su `.10` come fa failover?|entrypoint/control-plane|
|EXP4|Traefik failure|Traefik pod deleted|entrypoint si autoripristina?|ingress|
|EXP5|CPU overload|pod/nodo sotto stress|cosa succede quando il servizio non risponde?|degradation|
|EXP6|Replica distribution|pod placement|le repliche sono davvero distribuite?|scheduling/HA|
|EXP7|Health failure with livenessProbe|`/health` 500|k3s riavvia container unhealthy?|probes|
|EXP8|Image missing|wrong tag/local image missing|k3s non può creare pod senza immagine|operational limit|

