# node-pool-autoscale-taints-tolerations-affinity-pod-disruption-budgets

jak w nazwie repo - omawiamy tu:
- node-pools z opcją autoskalowania
- taints
- toleration na taints 
- affinity, POD affinity, AntiAffinity, NodeAffinity
- eviction 
- PodDisruptionBudgets


# GKE-Node-Pools z autoskalowaniem 

powołanie auto-scale pool z taint (extranodes=mlops) via gcloud :

```

gcloud container node-pools create pool-6 --cluster central --node-taints=extranodes=mlops:NoSchedule --enable-autoscaling --max-nodes=3 --min-nodes=0

```

chwile po powołaniu nowej dodatkowej auto-scaled-node-pool pojawiają się na niej pody (te Pody żyją przez jakiś czas, potem znikają i znikają nody) 

```

$ kubectl get po -A -o wide | grep pool-6
kube-system   fluentbit-gke-485js                                 2/2     Terminating         0          16m   10.128.0.4    gke-central-pool-1-0b68fd7f-v4q6         <none>           <none>
kube-system   fluentbit-gke-ck9lk                                 2/2     Terminating         0          16m   10.128.0.5    gke-central-pool-1-0b68fd7f-gwkf         <none>           <none>
kube-system   fluentbit-gke-jzwzs                                 2/2     Terminating         0          16m   10.128.0.6    gke-central-pool-1-0b68fd7f-j4hm         <none>           <none>
kube-system   gke-metadata-server-4hs4d                           0/1     ContainerCreating   0          91s   10.128.0.6    gke-central-pool-1-0b68fd7f-j4hm         <none>           <none>
kube-system   gke-metadata-server-8vnvk                           0/1     Pending             0          96s   <none>        gke-central-pool-1-0b68fd7f-gwkf         <none>           <none>
kube-system   gke-metadata-server-x5fzm                           0/1     Running             0          95s   10.128.0.4    gke-central-pool-1-0b68fd7f-v4q6         <none>           <none>
kube-system   gke-metrics-agent-7mf4x                             0/1     Pending             0          91s   <none>        gke-central-pool-1-0b68fd7f-j4hm         <none>           <none>
kube-system   gke-metrics-agent-9xvzf                             0/1     Terminating         0          16m   10.128.0.5    gke-central-pool-1-0b68fd7f-gwkf         <none>           <none>
kube-system   gke-metrics-agent-tmtzr                             1/1     Running             0          95s   10.128.0.4    gke-central-pool-1-0b68fd7f-v4q6         <none>           <none>
kube-system   kube-proxy-gke-central-pool-1-0b68fd7f-gwkf         1/1     Running             0          16m   10.128.0.5    gke-central-pool-1-0b68fd7f-gwkf         <none>           <none>
kube-system   kube-proxy-gke-central-pool-1-0b68fd7f-j4hm         1/1     Running             0          16m   10.128.0.6    gke-central-pool-1-0b68fd7f-j4hm         <none>           <none>
kube-system   kube-proxy-gke-central-pool-1-0b68fd7f-v4q6         1/1     Running             0          16m   10.128.0.4    gke-central-pool-1-0b68fd7f-v4q6         <none>           <none>
kube-system   netd-5qfkr                                          1/1     Running             0          98s   10.128.0.6    gke-central-pool-1-0b68fd7f-j4hm         <none>           <none>
kube-system   netd-hx9p4                                          0/1     Init:0/1            0          99s   10.128.0.5    gke-central-pool-1-0b68fd7f-gwkf         <none>           <none>
kube-system   netd-m5kcx                                          1/1     Running             0          98s   10.128.0.4    gke-central-pool-1-0b68fd7f-v4q6         <none>           <none>
kube-system   pdcsi-node-2b7wg                                    0/2     Pending             0          91s   <none>        gke-central-pool-1-0b68fd7f-v4q6         <none>           <none>
kube-system   pdcsi-node-kmffw                                    0/2     Pending             0          91s   <none>        gke-central-pool-1-0b68fd7f-j4hm         <none>           <none>
kube-system   pdcsi-node-w8q9g                                    0/2     Terminating         0          16m   10.128.0.5    gke-central-pool-1-0b68fd7f-gwkf         <none>           <none>

```

Są one wynikami Daemon-setów (oczywiście tych gdzie nie ma ZERO/ZERO):

```

$ kk get daemonsets -A
NAMESPACE     NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                                                             AGE
kube-system   fluentbit-gke               5         5         5       5            5           kubernetes.io/os=linux                                                    172m
kube-system   gke-metadata-server         5         5         5       5            5           beta.kubernetes.io/os=linux,iam.gke.io/gke-metadata-server-enabled=true   172m
kube-system   gke-metrics-agent           5         5         5       5            5           kubernetes.io/os=linux                                                    172m
kube-system   gke-metrics-agent-windows   0         0         0       0            0           kubernetes.io/os=windows                                                  172m
kube-system   kube-proxy                  0         0         0       0            0           kubernetes.io/os=linux,node.kubernetes.io/kube-proxy-ds-ready=true        172m
kube-system   metadata-proxy-v0.1         0         0         0       0            0           cloud.google.com/metadata-proxy-ready=true,kubernetes.io/os=linux         172m
kube-system   netd                        5         5         5       5            5           cloud.google.com/gke-netd-ready=true,kubernetes.io/os=linux               172m
kube-system   nvidia-gpu-device-plugin    0         0         0       0            0           <none>                                                                    172m
kube-system   pdcsi-node                  5         5         5       5            5           kubernetes.io/os=linux                                                    172m
kube-system   pdcsi-node-windows          0         0         0       0            0           kubernetes.io/os=windows                                                  172m

```



co ważne - jak się powoła node-poole z TAINT to powyższe i tak na chwile się pojawiają - dlatego że po pierwsze są DaemonSetami a po drugie DS-systemowe mają tolerację na wszystko



natomiast raczej tak zaraz po dodaniu do GKE  dodatkowej node-pooli nie powinno się na niej na tzw. "chwilę" pojawić nic co jest deployem czy STS - wyłącznie DaemonSety (chociaż różnie bywa)





## Jak wygląda praca z nodami z puli która ma autoskalowanie - czyli WYMUSZENIA SCALE-UP (żeby te dodatkowe nody się obudziły):



### a) Wymuszenie ilościowe

przykładowo scale deployu do 200 (198 max na default-pool, powołuje 1-extra-node) , scale do 400 (powołuje 2gi extra-node), scale do 450 - powołuje 3ci extra node

po takim scale-up poza app-pods na extra-nodes pojawia się składowe-śmieciowe związane z Daemon-setami z kube-system (fluentbit-gke,gke-metadata-server,gke-metrics-agent,netd i pdcsi-node) oraz czasem może zdarzyć się że związane z deployami (konnectivity-agent)

po scale down dla businnes-PODS do ZERO znika wszystko , te systemowe śmiecie też i na końcu oczywiście te dodatkowe nody też są wyłączane 



### b) Wymuszenie via requesty przekraczające rozmiar Linuxów w default-pool



skoro te nody w default-pool mają każdy: CPU requested￼ CPU allocatable￼ Memory requested￼ Memory allocatable￼: 371 mCPU 1.93 CPU 529.53 MB 5.89 GB to żeby przekroczyć zróbmy na CPU w definicji YAML-deployu:

   spec:
      containers:
      - image: gimboo/nginx_nonroot
        resources:

          requests:
            cpu: "0.5"

[...]



powyższe jest obecne w pliku deploy-consumer.yaml





### PROBLEMY Z BLOKOWANIEM WĘZŁÓW, jak powstają i skąd się biorą 



jak wiemy śmieci z kube-system pojawiają na auto-scale-node-pool ((po zwykłym obudzeniu tej puli loadem biznesowym) LUB (zaraz po utworzeniu tej puli po czym znikają))  - te śmieci to oczywiście DaemonSety (fluentbit-gke,gke-metadata-server,gke-metrics-agent,kube-proxy,netd,pdcsi-node), czasami też śmieci z Deploymentu (tu konnectivity-agent)

po scale down dla businnes-PODS do ZERO znika wszystko , systemowe śmiecie też i nody też 



gdy jednak zdarzy sie że na extra-node-pool wskoczy POD związany z systemowym Deployem to sytuacja może być różna 



wystarczy spowodować przypadek (via N x "kubectl delete") że na extra-node wskoczy z deployów systemowych poza konnectivity-agent jeszcze np. kube-dns - wtedy pula sie zblokuje (mimo wyłączenia load-biznesowego nie nastąpi likwidacja nodów w puli) 



POD z deployu kube-dns trzyma węzeł i powoduje w logu GKE error: no.scale.down.node.pod.kube.system.unmovable 



oczywiście po "kubectl delete" na POD z kube-dns na extra-węźle zostaje tylko konnectivity-deploy-pod , po czym znika i pula sie zeruje 



obydwa te deploye (kube-dns i konnectivity-agent) są z kube-system, czym zatem się różnią ? jeden blokuje extra-węzły a drugi nie blokuje 



konnectivity-agent ma na pewno coś czego nie ma kube-dns: 

```

  template:
    metadata:
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: "true"

```



przestawić się mu na false nie da wiec można zrobić na odwrót i wstawić true dla kube-dns -  po ponownym teście wychodzi wtedy że kube-dns przestaje blokować i trzymać przy życiu węzły z puli atoskalowanej  i się migruje 



PODSUMOWUJĄC: wygląda na to że deploye systemowe jeśli mają safe-to-evict: "true" to nie blokują operacji Scale-down, jesli nie mają to trzymają węzeł do usranej śmierci 



### c) Wymuszanie via affinity 

żeby nie dociskać wyzwolenia węzłów prymitywnie nadmierną ilością podów (a) ani sztucznymi/fikcyjnymi cpu-reqests(b) można wyzwolić to via affinity - czyli deploy ma każdy swój POD deployować na inny node :



```

  template:
    metadata:
      labels:
        app: consumer
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - consumer
            topologyKey: "kubernetes.io/hostname"
      containers:
      - image: gimboo/nginx_nonroot
        imagePullPolicy: Always
        name: apacz-2

```

plik deploy-consumer-anti-affinity.yaml



powyższy mechanizm  (https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#more-practical-use-cases) separuje wszystkie PODy na różnych węzłach , a jak brakuje węzłów żeby zrobić stosowne replicas to dosztukuje węzły via autoskaler i Scale-UP: 





### Jak działa mechanizm via podAntiAffinity aby rozrzucać PODy na różne Nody - w praktyce 



mamy 2 węzły w default-pool:

```

$ kk get nodes
NAME                                     STATUS   ROLES    AGE   VERSION
gke-central-default-pool-4afc0907-c60x   Ready    <none>   10h   v1.21.10-gke.2000
gke-central-default-pool-4afc0907-g7ll   Ready    <none>   10h   v1.21.10-gke.2000
```

wgrywamy deploy z tym gizmem ale z R=1 

```

$ kk apply -f deploy-consumer-anti-affinity.yaml
deployment.apps/consumer created
$ kk get po -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP             NODE                                     NOMINATED NODE   READINESS GATES
consumer-bc4d56946-lfqhz   1/1     Running   0          5s    10.104.1.141   gke-central-default-pool-4afc0907-c60x   <none>           <none>
```



robimy scale do 2 - nic sie nie dzieje bo mamy 2 node w default-pool , więc nawet jeśli nie wolno im przebywać obok siebie to się jeszcze jakoś mieszczą 

```

$ kk scale deploy consumer --replicas=2
deployment.apps/consumer scaled
$ kk get po -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP             NODE                                     NOMINATED NODE   READINESS GATES
consumer-bc4d56946-52mn2   1/1     Running   0          3s    10.104.0.92    gke-central-default-pool-4afc0907-g7ll   <none>           <none>
consumer-bc4d56946-lfqhz   1/1     Running   0          30s   10.104.1.141   gke-central-default-pool-4afc0907-c60x   <none>           <none>
$ kk get nodes
NAME                                     STATUS   ROLES    AGE   VERSION
gke-central-default-pool-4afc0907-c60x   Ready    <none>   10h   v1.21.10-gke.2000
gke-central-default-pool-4afc0907-g7ll   Ready    <none>   10h   v1.21.10-gke.2000

```

robimy scale do 3 - autoskaler musi dorzucić 1 extra węzeł :

```

$ kk scale deploy consumer --replicas=3
deployment.apps/consumer scaled
$ kk get po -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP             NODE                                     NOMINATED NODE   READINESS GATES
consumer-bc4d56946-4wz5d   0/1     Pending   0          4s    <none>         <none>                                   <none>           <none>
consumer-bc4d56946-52mn2   1/1     Running   0          23s   10.104.0.92    gke-central-default-pool-4afc0907-g7ll   <none>           <none>
consumer-bc4d56946-lfqhz   1/1     Running   0          50s   10.104.1.141   gke-central-default-pool-4afc0907-c60x   <none>           <none>

```

po chwili POD ktory miał stan Pending rozkręca się (bo już ma gdzie):

```

$ kk get po -o wide
NAME                       READY   STATUS    RESTARTS   AGE    IP             NODE                                     NOMINATED NODE   READINESS GATES
consumer-bc4d56946-4wz5d   1/1     Running   0          76s    10.104.2.2     gke-central-pool-1-0b68fd7f-tnzw         <none>           <none>
consumer-bc4d56946-52mn2   1/1     Running   0          95s    10.104.0.92    gke-central-default-pool-4afc0907-g7ll   <none>           <none>
consumer-bc4d56946-lfqhz   1/1     Running   0          2m2s   10.104.1.141   gke-central-default-pool-4afc0907-c60x   <none>           <none>
$ kk get nodes
NAME                                     STATUS   ROLES    AGE   VERSION
gke-central-default-pool-4afc0907-c60x   Ready    <none>   10h   v1.21.10-gke.2000
gke-central-default-pool-4afc0907-g7ll   Ready    <none>   10h   v1.21.10-gke.2000
gke-central-pool-1-0b68fd7f-tnzw         Ready    <none>   36s   v1.21.10-gke.2000

```



robimy scale do 4 - autoskaler musi dorzucić jeszcze jeden (to już drugi) extra węzeł :

```

$ kk scale deploy consumer --replicas=4
deployment.apps/consumer scaled
$ kk get po -o wide
NAME                       READY   STATUS    RESTARTS   AGE     IP             NODE                                     NOMINATED NODE   READINESS GATES
consumer-bc4d56946-4wz5d   1/1     Running   0          110s    10.104.2.2     gke-central-pool-1-0b68fd7f-tnzw         <none>           <none>
consumer-bc4d56946-52mn2   1/1     Running   0          2m9s    10.104.0.92    gke-central-default-pool-4afc0907-g7ll   <none>           <none>
consumer-bc4d56946-lfqhz   1/1     Running   0          2m36s   10.104.1.141   gke-central-default-pool-4afc0907-c60x   <none>           <none>
consumer-bc4d56946-pv6l6   0/1     Pending   0          8s      <none>         <none>                                   <none>           <none>

```

i tak dalej (do czasu aż nam węzłów wystarczy)




