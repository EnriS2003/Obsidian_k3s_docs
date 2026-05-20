
> Dettaglio tecnico importante: un nodo sarà a **valle** e uno a **monte**. Bisogna capire come k3s gestisce le repliche e prendere alcuni use-case:
- come la gestione di una replica di logik,
- oppure la gestione di una replica di IO-Networks - _vedi i punti 2a e 2b sotto_

1. _Gestione interfaccia browser al kill del Nodo 1:_ capire come passare alla 100.30 se la 100.10 va in crash.
        1. Problema: se io ho la WebUi che va e ad un certo punto crasha il nodo principale, come fa il browser dell’operatore a spostarsi in automatico alla 100.30 senza che l’operatore lo noti troppo?
        2. Soluzione: vIP → virtual IP Tutto sta sotto un unico IP 100.100 → 100.10 e 100.30. Da vedere quale applicativo/componente/SW può fare questo.
    2. Sarebbe da capire anche i vari collegamenti tra i componenti come sono gestiti da k3s, cioè se si potrebbero creare problemi di vario genere. Tipo: _senza Api-backend, webUi e logic-simulator non potrebbero comunicare._ **Com’è gestita sta cosa qui?**
2. Altre cose da tenere in considerazione:
    1. Capire se Logic necessita di un’altra replica o no
    2. Come IO-Networks (che riceve dati da componenti esterni, dai PLC) viene gestito se questo va in crash.
    3. Testare come reagisce k3s a una mancanza di risorse.
3. Come ho installato k3s: packages e cose varie che a loro potrebbero tornare utili.

## Elaborazione idee/esperimenti dal meeting con Peter

### 1. Salvare la baseline

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

> Guarda ciò che è stato scritto in [[Esperimenti]] #tag 