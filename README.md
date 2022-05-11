
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





### PROBLEMY Z BLOKOWANIEM WĘZŁÓW, jak powstają i skąd się biorą , SAFE TO EVICTION



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

i tak dalej (do czasu aż nam węzłów w tej dodatkowej puli wystarczy)









# TAINT na Nodach - czyli jak zabronić wjazdu na nody 



Taint to blokada wejścia dla wszystkich którzy nie tolerują takiego tainta (a że nody same powstają i się same gaszą to oczywiście nie biegamy za nodami z pieczątką ale nakładamy taint nie na node ale na całą node-pool) 



UWAGA ! do puli z autoskalowaniem nie da się dorzucić TAINT : 

```

Updates for 'taints' are not supported in node pools with autoscaling enabled (as a workaround, consider temporarily disabling autoscaling or recreating the node pool with the updated values.).

```



jak sie nałoży taki TAINT to po wymuszeniu deployentem biznesowym powołania extra-nodes Autoscaler czasem rozkręci dodatkowy node a czasami wogóle zrezygnuje z tego - ale niezależnie czy rozkręci czy nie to zawsze zadziała tak że nie wpuści PODa : 



```

Events:
  Type     Reason             Age                    From                Message
  ----     ------             ----                   ----                -------
  Warning  FailedScheduling   19m                    default-scheduler   0/5 nodes are available: 2 node(s) didn't match pod affinity/anti-affinity rules, 2 node(s) didn't match pod anti-affinity rules, 3 node(s) had taint {extranodes: mlops}, that the pod didn't tolerate.
```

LUB (w przypadku gdy nawet nie chce rozkręcić auto-scale-node-pooli:) :



```

   Warning  FailedScheduling   93s               default-scheduler   0/2 nodes are available: 2 node(s) didn't match pod affinity/anti-affinity rules, 2 node(s) didn't match pod anti-affinity rules.

  Warning  FailedScheduling   3s (x1 over 91s)  default-scheduler   0/2 nodes are available: 2 node(s) didn't match pod affinity/anti-affinity rules, 2 node(s) didn't match pod anti-affinity rules.
  Normal   NotTriggerScaleUp  91s               cluster-autoscaler  pod didn't trigger scale-up: 1 node(s) had taint {extranodes: mlops}, that the pod didn't tolerate

```



# TOLERATION



aby Deployment wszedł na nody z TAINTS potrzebna jest w deploymencie wstawiona toleration:



```

      tolerations:
      - effect: NoSchedule
        key: extranodes
        operator: Equal
        value: mlops
      dnsPolicy: ClusterFirst

```



plik deploy-consumer-anti-affinity-TOLERATION.yaml





PRZY OKAZJI PATENT NA TOLERATION "NA WSZYSTKO" - sprytne ale niebezpieczne :

```
#      tolerations:
#      - effect: NoSchedule
#        operator: Exists

``` 



# AUTOMATIC k8s TAINTS i CONDITION TAINTS 



idąc za: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/



nie tylko admin moze robić taint na węzłach, robi to również sam k8s (a konkretnie node-controller): 



```

The node controller automatically taints a Node when certain conditions are true. The following taints are built in:

node.kubernetes.io/not-ready: Node is not ready. This corresponds to the NodeCondition Ready being "False".
node.kubernetes.io/unreachable: Node is unreachable from the node controller. This corresponds to the NodeCondition Ready being "Unknown".
node.kubernetes.io/memory-pressure: Node has memory pressure.
node.kubernetes.io/disk-pressure: Node has disk pressure.
node.kubernetes.io/pid-pressure: Node has PID pressure.
node.kubernetes.io/network-unavailable: Node's network is unavailable.
node.kubernetes.io/unschedulable: Node is unschedulable.
node.cloudprovider.kubernetes.io/uninitialized: When the kubelet is started with "external" cloud provider, this taint is set on a node to mark it as unusable. After a controller from the cloud-controller-manager initializes this node, the kubelet removes this taint.

```



możemy zatem np. uzbroić nasz deployment w toleration dla któregoś z tych stanów - np POD nieprzeganialny przez node.kubernetes.io/unreachable musi zawierać definicję: 



```

tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000

```



są też Tainty nakładane przez conditions:





```

node.kubernetes.io/memory-pressure
node.kubernetes.io/disk-pressure
node.kubernetes.io/pid-pressure (1.14 or later)
node.kubernetes.io/unschedulable (1.10 or later)
node.kubernetes.io/network-unavailable (host network only)

```





# LABEL na nodach i nakaz deployu tylko na nody z takim labelem (tzw. NodeAffinity)



zostawmy jeszcze na chwile starą auto-skalowaną-node-pool (ale bez labelu - czyli pool-6) i wdrażamy deploy który ma dodatkowo:



```     
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: machine
                operator: In
                values:
                - big

```

plik deploy-consumer-anti-affinity-TOLERATION-NodeAffinity.yaml





Deploy wisi (PODy w stanie Pending) zaś Autoskaler nie rozkręci extra-nodes gdyż nie ma spełnionego warunku:



```

$ kk get po
NAME                        READY   STATUS    RESTARTS   AGE
consumer-6dcb6986f5-2b6cx   0/1     Pending   0          21s
consumer-6dcb6986f5-5pdp2   0/1     Pending   0          21s



  Warning  FailedScheduling   13s   default-scheduler   0/2 nodes are available: 2 node(s) didn't match Pod's node affinity/selector.
  Warning  FailedScheduling   12s   default-scheduler   0/2 nodes are available: 2 node(s) didn't match Pod's node affinity/selector.
  Normal   NotTriggerScaleUp  11s   cluster-autoscaler  pod didn't trigger scale-up: 1 node(s) didn't match Pod's node affinity/selector

```



potrzebna jest zatem nowa node-poola - ale tym razem z Labelem (machine=big):



```

gcloud container node-pools create pool-7 --cluster central --node-taints=extranodes=mlops:NoSchedule --enable-autoscaling --max-nodes=3 --min-nodes=0 --node-labels=machine=big

```

i już jest ok, Deploy wymaga aby jego PODy uruchomiły się tylko na nodach z labelem "machine=big" , Autoskaler uruchamia zatem taką node-pool (pool-7) a na niej uruchamiają się PODy:



```

$ kk get po -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP           NODE                               NOMINATED NODE   READINESS GATES
consumer-6dcb6986f5-2b6cx   1/1     Running   0          4m54s   10.104.3.2   gke-central-pool-7-cd3343f3-n9fn   <none>           <none>
consumer-6dcb6986f5-5pdp2   1/1     Running   0          4m54s   10.104.2.2   gke-central-pool-7-cd3343f3-4x27   <none>           <none>

```









# PodDisruptionBudget



https://kubernetes.io/docs/tasks/run-application/configure-pdb/



```

$ cat pdb.yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: sample-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: consumer



$ kk apply -f  pdb.yaml
Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
poddisruptionbudget.policy/app-pdb created

```



## Specyfikacjja PodDisruptionBudget 

3 pola: 

```
A label selector .spec.selector to specify the set of pods to which it applies. This field is required.
.spec.minAvailable which is a description of the number of pods from that set that must still be available after the eviction, even in the absence of the evicted pod. minAvailable can be either an absolute number or a percentage.
.spec.maxUnavailable (available in Kubernetes 1.7 and higher) which is a description of the number of pods from that set that can be unavailable after the eviction. It can be either an absolute number or a percentage.

```



skoro PDB "ochrania" PODy podczas eviction należy wywołać takową ewikcję - wcześniej chroniąc PODy 



dotychczasowe zachowanie po usunięciu PODów biznesowych było takie że node-pool-autoskalowana zwijała się do zera (mimo że były na niej PODy systemowych daemon-setów) 



najprawdopodobniej te daemon-sety nie były zatem chronione 



## TEST1 

należy wdrożyć deploy-consumer-anti-affinity-TOLERATION-NodeAffinity.yaml (ustawić w środku replicas na 1) , zaczekać na powołanie noda z node-pool-autoskalowanej, zaobserwować na tym węźle :



a) PODy systemowe (wynikające z daemon-setów)

b) POD biznesowy 



```

$ kk get po -A -o wide | grep pool-7
default       consumer-6dcb6986f5-d5zfb                           1/1     Running   0          10m     10.104.2.2    gke-central-pool-7-cd3343f3-9lwp         <none>           <none>
kube-system   fluentbit-gke-4xbjr                                 2/2     Running   0          9m54s   10.128.0.22   gke-central-pool-7-cd3343f3-9lwp         <none>           <none>
kube-system   gke-metadata-server-s7hg6                           1/1     Running   0          9m54s   10.128.0.22   gke-central-pool-7-cd3343f3-9lwp         <none>           <none>
kube-system   gke-metrics-agent-8nclc                             1/1     Running   0          9m55s   10.128.0.22   gke-central-pool-7-cd3343f3-9lwp         <none>           <none>
kube-system   kube-proxy-gke-central-pool-7-cd3343f3-9lwp         1/1     Running   0          9m55s   10.128.0.22   gke-central-pool-7-cd3343f3-9lwp         <none>           <none>
kube-system   netd-cq8v7                                          1/1     Running   0          9m54s   10.128.0.22   gke-central-pool-7-cd3343f3-9lwp         <none>           <none>
kube-system   pdcsi-node-d7t2q                                    2/2     Running   0          9m55s   10.128.0.22   gke-central-pool-7-cd3343f3-9lwp         <none>           <none>

```



jeden z daemon setów dozbrajamy w PDB chroniące go przed eviction - wcześniej sprawdzając jakie ma labele aby jedną z nich wybrać i na niej wygodnie (dla nas) oprzeć ochronę via PDB :



```

$ kk get daemonset fluentbit-gke -n kube-system --show-labels
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE   LABELS
fluentbit-gke   3         3         3       3            3           kubernetes.io/os=linux   33h   addonmanager.kubernetes.io/mode=Reconcile,k8s-app=fluentbit-gke,kubernetes.io/cluster-service=true

```



najwygodniejszy LABEL  to najlepiej ten: k8s-app=fluentbit-gke



na bazie tego labela definiujemy PDB:



```

$ cat pdb-fluentbit.yaml                                                                                                  
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: fluentbit-pdb
spec:
  minAvailable: 3
  selector:
    matchLabels:
      k8s-app: fluentbit-gke



$ kk apply -f pdb-fluentbit.yaml
Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
poddisruptionbudget.policy/fluentbit-pdb created
```



w gruncie rzeczy od tej pory eviction na tym daemon-secie powinno być AWYKONALNE:



```

$ kk get pdb -A -o wide
NAMESPACE     NAME            MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
default       app-pdb         2               N/A               0                     16m
kube-system   fluentbit-pdb   3               N/A               0                     9s

```



skalujemy biznesowy deploy do 0 co powinno (tak jak wcześniej) spowodować po pewnym czasie likwidację node-pooli-autoskalowanej (poprzedzoną oczywiście likwidacją chwilę wczesniej PODów zw. z systemowymi daemonSetami)



```

$ kubectl scale deploy consumer --replicas 0

```



TAK SIE JEDNAK NIE DZIEJE - PDB chroni (jako-tako ale o tym za chwilę) DS=fluentbit-gke, dodatkowy NODE żyje nadal i być zlikwidowanym (przynajmniej na razie) nie ma zamiaru





w LOGS klastra (w sekcji AUTOSCALER LOGS) widnieje tylko w kółko: 



```

{
  "noDecisionStatus": {
    "noScaleDown": {
      "nodesTotalCount": 1,
      "nodes": [
        {
          "reason": {
            "parameters": [
              "consumer-6dcb6986f5-d5zfb"
            ],
            "messageId": "no.scale.down.node.pod.not.enough.pdb"
          },
          "node": {
            "name": "gke-central-pool-7-cd3343f3-9lwp",
            "mig": {
              "zone": "us-central1-b",
              "name": "gke-central-pool-7-cd3343f3-grp",
              "nodepool": "pool-7"
            }
          }
        }
      ]
    },
    "measureTime": "1652214019"
  }
}

```





## TEST2  - skalujemy Deploy aplikacyjny do 2, czekamy na powołanie 2 węzłów z autoskalującej-node-pooli , obserwujemy że PDB fluentbita widzi teraz "3 +1 na allowed disruptions" 



```

$ kk scale deploy consumer --replicas=2
deployment.apps/consumer scaled
$ kk get po
NAME                        READY   STATUS    RESTARTS   AGE
consumer-6dcb6986f5-78cz2   0/1     Pending   0          5s
consumer-6dcb6986f5-jss77   1/1     Running   0          10m

```

po chwili pojawia się nowy kolejny węzeł na ktorym rozgaszcza się POD DaemonSetu Fluentbita , a zatem PDB wskazuje już co innnego , mamy rezerwę na śmierć jednego PODA:



```

$ kk get pdb -A
NAMESPACE     NAME            MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
kube-system   fluentbit-pdb   3               N/A               1                     2m35s

```

powtarzamy test ze skalowaniem deployu app z 2 do 0, powinno to spowodować likwidację nie 2 węzłów (jak dotychczas) ale tylko jednego z racji wytycznych jakich pilnuje PDB



innymi słowy powinno być tak że jeden node zniknął ale drugi jest trzymany via PDB "chroniące" systemowy daemon-set fluentbita 



oczywiście początkowo (zanim node nie zniknął) PDB raportuje nadal 3+1:

```

NAMESPACE     NAME            MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
kube-system   fluentbit-pdb   3               N/A               1                     7m4s

```

 

ale po chwili jeden z węzłów znika i zostaje tylko ten jeden trzymany przy życiu przez PDB (pdb wskazuje teraz juz tylko 3+0) 



```

NAME                                     STATUS   ROLES    AGE   VERSION
gke-central-default-pool-b0d6c80c-9z88   Ready    <none>   43h   v1.21.10-gke.2000
gke-central-default-pool-b0d6c80c-lpmb   Ready    <none>   43h   v1.21.10-gke.2000
gke-central-pool-7-498fa03d-1tgm         Ready    <none>   43m   v1.21.10-gke.2000
---
kube-system   fluentbit-gke-65sjh                                 2/2     Terminating   0          17m    10.128.0.29   gke-central-pool-7-498fa03d-7r4g         <none>           <none>
kube-system   fluentbit-gke-q7mz9                                 2/2     Running       0          43m    10.128.0.26   gke-central-pool-7-498fa03d-1tgm         <none>           <none>
kube-system   gke-metadata-server-w288l                           0/1     Terminating   0          17m    10.128.0.29   gke-central-pool-7-498fa03d-7r4g         <none>           <none>
kube-system   gke-metadata-server-wwc2z                           1/1     Running       0          43m    10.128.0.26   gke-central-pool-7-498fa03d-1tgm         <none>           <none>
kube-system   gke-metrics-agent-fz2wv                             1/1     Running       0          43m    10.128.0.26   gke-central-pool-7-498fa03d-1tgm         <none>           <none>
kube-system   gke-metrics-agent-lcq68                             1/1     Terminating   0          17m    10.128.0.29   gke-central-pool-7-498fa03d-7r4g         <none>           <none>
kube-system   kube-proxy-gke-central-pool-7-498fa03d-1tgm         1/1     Running       0          43m    10.128.0.26   gke-central-pool-7-498fa03d-1tgm         <none>           <none>
kube-system   kube-proxy-gke-central-pool-7-498fa03d-7r4g         1/1     Terminating   0          17m    10.128.0.29   gke-central-pool-7-498fa03d-7r4g         <none>           <none>
kube-system   netd-rm2cz                                          0/1     Init:0/1      0          119s   10.128.0.29   gke-central-pool-7-498fa03d-7r4g         <none>           <none>
kube-system   netd-zxl97                                          1/1     Running       0          43m    10.128.0.26   gke-central-pool-7-498fa03d-1tgm         <none>           <none>
kube-system   pdcsi-node-849jk                                    2/2     Running       0          43m    10.128.0.26   gke-central-pool-7-498fa03d-1tgm         <none>           <none>
kube-system   pdcsi-node-fxhhq                                    0/2     Terminating   0          17m    10.128.0.29   gke-central-pool-7-498fa03d-7r4g         <none>           <none>
---
NAMESPACE     NAME            MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
kube-system   fluentbit-pdb   3               N/A               0                     19m  

```



CO CIEKAWE PDB NIE DZIAŁA TU NA STAŁE  - po chwili k8s i tak likwiduje nody - pytanie dlaczego?



być może dzieje się tak dlatego że DaemonSet nie jest wspieranym kontrolerem dla PDB



idąc za https://kubernetes.io/docs/tasks/run-application/configure-pdb/ wskazano wspierane kontrolery:



```

The most common use case when you want to protect an application specified by one of the built-in Kubernetes controllers:
Deployment
ReplicationController
ReplicaSet
StatefulSet

```



trzeba zatem zrobić DEPLOY - a konkretnie taki który wskoczy na extra-nodes ale będzie tez nadawał się do ewikcji (początkowo bez PDB a potem z PDB ktory nie pozwoli na ewikcję (zanik) nawet jednego PODa) 



sterowac ewikcją będziemy oczywiście via ilość extra-podów - czyli via ilość replik app=consumer 



sprzątamy i rozkręcamy app=consumer do replik=3 co spowoduje pojawienie sie 3 extra-węzłów:



```

$ kk delete  pdb fluentbit-pdb -n kube-system

poddisruptionbudget.policy "fluentbit-pdb" deleted
$ kk get pdb -A
No resources found

$ kk get nodes

NAME                                     STATUS   ROLES    AGE   VERSION
gke-central-default-pool-b0d6c80c-9z88   Ready    <none>   46h   v1.21.10-gke.2000
gke-central-default-pool-b0d6c80c-lpmb   Ready    <none>   46h   v1.21.10-gke.2000
$ kk scale deploy consumer --replicas=3
deployment.apps/consumer scaled
$ kk get po
NAME                        READY   STATUS    RESTARTS   AGE
consumer-6dcb6986f5-gz65h   0/1     Pending   0          3s
consumer-6dcb6986f5-l6cjm   0/1     Pending   0          3s
consumer-6dcb6986f5-xndjx   0/1     Pending   0          3s

```

po chwili:



```

$ kk get po
NAME                        READY   STATUS    RESTARTS   AGE
consumer-6dcb6986f5-gz65h   1/1     Running   0          2m9s
consumer-6dcb6986f5-l6cjm   1/1     Running   0          2m9s
consumer-6dcb6986f5-xndjx   1/1     Running   0          2m9s
$ kk get nodes
NAME                                     STATUS   ROLES    AGE   VERSION
gke-central-default-pool-b0d6c80c-9z88   Ready    <none>   46h   v1.21.10-gke.2000
gke-central-default-pool-b0d6c80c-lpmb   Ready    <none>   46h   v1.21.10-gke.2000
gke-central-pool-7-498fa03d-7br4         Ready    <none>   90s   v1.21.10-gke.2000
gke-central-pool-7-498fa03d-8cwv         Ready    <none>   89s   v1.21.10-gke.2000
gke-central-pool-7-498fa03d-x6z2         Ready    <none>   90s   v1.21.10-gke.2000

```



wgrywamy drugi deploy (replicas=5) ktory umie się rozmieścić na wszystkich węzłach (ma więc toleration, jednak lepiej żeby nie "na wszystko" ale tylko na ten jeden taint oryginalny ) 



```

$ kk apply -f  deploy-consumer2.yaml

```



część z PODów trafiła na węzły dodatkowej node-pool



```

$ kk get po -A -o wide | grep pool-7 | grep consumer2
default       consumer2-5c8c67488c-9s9cp                          1/1     Running   0          9m43s   10.104.4.3    gke-central-pool-7-498fa03d-x6z2         <none>           <none>
default       consumer2-5c8c67488c-fzfsd                          1/1     Running   0          9m43s   10.104.3.3    gke-central-pool-7-498fa03d-7br4         <none>           <none>
default       consumer2-5c8c67488c-wtwsl                          1/1     Running   0          9m43s   10.104.2.3    gke-central-pool-7-498fa03d-8cwv         <none>           <none>

```



wywołujemy likwidacje extra-node-pool przez scale do zera pierwszego deployu (czyli tego który ma NodeAffinity na extra-poole) 



```

$ kk get po  -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP            NODE                                     NOMINATED NODE   READINESS GATES
consumer-6dcb6986f5-gz65h    1/1     Running   0          21m   10.104.4.2    gke-central-pool-7-498fa03d-x6z2         <none>           <none>
consumer-6dcb6986f5-l6cjm    1/1     Running   0          21m   10.104.2.2    gke-central-pool-7-498fa03d-8cwv         <none>           <none>
consumer-6dcb6986f5-xndjx    1/1     Running   0          21m   10.104.3.2    gke-central-pool-7-498fa03d-7br4         <none>           <none>
consumer2-5c8c67488c-9s9cp   1/1     Running   0          16m   10.104.4.3    gke-central-pool-7-498fa03d-x6z2         <none>           <none>
consumer2-5c8c67488c-dr7mz   1/1     Running   0          16m   10.104.0.33   gke-central-default-pool-b0d6c80c-lpmb   <none>           <none>
consumer2-5c8c67488c-fzfsd   1/1     Running   0          16m   10.104.3.3    gke-central-pool-7-498fa03d-7br4         <none>           <none>
consumer2-5c8c67488c-kmwzc   1/1     Running   0          16m   10.104.0.32   gke-central-default-pool-b0d6c80c-lpmb   <none>           <none>
consumer2-5c8c67488c-wtwsl   1/1     Running   0          16m   10.104.2.3    gke-central-pool-7-498fa03d-8cwv         <none>           <none>
$ kk get nodes
NAME                                     STATUS   ROLES    AGE   VERSION
gke-central-default-pool-b0d6c80c-9z88   Ready    <none>   47h   v1.21.10-gke.2000
gke-central-default-pool-b0d6c80c-lpmb   Ready    <none>   47h   v1.21.10-gke.2000
gke-central-pool-7-498fa03d-7br4         Ready    <none>   21m   v1.21.10-gke.2000
gke-central-pool-7-498fa03d-8cwv         Ready    <none>   21m   v1.21.10-gke.2000
gke-central-pool-7-498fa03d-x6z2         Ready    <none>   21m   v1.21.10-gke.2000


$ kk scale deploy consumer --replicas=0
deployment.apps/consumer scaled

```



po (dłuższej) chwili extra-nodes zniknęły a PODy consumera2 zostały przemigrowane:



```

NAME                         READY   STATUS    RESTARTS   AGE     IP            NODE                                     NOMINATED NODE   READINESS GATES
consumer2-5c8c67488c-dr7mz   1/1     Running   0          36m     10.104.0.33   gke-central-default-pool-b0d6c80c-lpmb   <none>           <none>
consumer2-5c8c67488c-fkqtk   1/1     Running   0          5m2s    10.104.1.20   gke-central-default-pool-b0d6c80c-9z88   <none>           <none>
consumer2-5c8c67488c-h7jng   1/1     Running   0          4m11s   10.104.0.34   gke-central-default-pool-b0d6c80c-lpmb   <none>           <none>
consumer2-5c8c67488c-kmwzc   1/1     Running   0          36m     10.104.0.32   gke-central-default-pool-b0d6c80c-lpmb   <none>           <none>
consumer2-5c8c67488c-xchvc   1/1     Running   0          3m21s   10.104.1.21   gke-central-default-pool-b0d6c80c-9z88   <none>           <none>

```



nałóżmy zatem PDB 



```

$ cat   pdb-consumer2.yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: consumer2-pdb
spec:
  minAvailable: 5
  selector:
    matchLabels:
      app: consumer2
$ kk apply -f pdb-consumer2.yaml
Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
poddisruptionbudget.policy/consumer2-pdb created
$ kk get pdb
NAME            MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
consumer2-pdb   5               N/A               0                     3s

```



wymuszamy rozkręcenie się 3 extra-węzłów:

```

$ kk scale deploy consumer --replicas=3

```

jak już się pojawią przekręcamy drugi deploy (na tyle duży żeby pody pojawiły sie wszędzie) 



```

$ kk delete -f deploy-consumer2.yaml
$ kk apply -f deploy-consumer2.yaml

```



część podów consumer2 trafiła na extra-pool, PDB jest 5+0 



```

$ kk get po -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP            NODE                                     NOMINATED NODE   READINESS GATES
consumer-6dcb6986f5-ltq8x    1/1     Running   0          2m51s   10.104.4.2    gke-central-pool-7-498fa03d-q8xj         <none>           <none>
consumer-6dcb6986f5-rch96    1/1     Running   0          2m51s   10.104.2.2    gke-central-pool-7-498fa03d-pqdj         <none>           <none>
consumer-6dcb6986f5-vtbjc    1/1     Running   0          2m51s   10.104.3.2    gke-central-pool-7-498fa03d-7b9l         <none>           <none>
consumer2-6bdb546c89-7nz86   1/1     Running   0          4s      10.104.3.3    gke-central-pool-7-498fa03d-7b9l         <none>           <none>
consumer2-6bdb546c89-m6ph6   1/1     Running   0          4s      10.104.0.37   gke-central-default-pool-b0d6c80c-lpmb   <none>           <none>
consumer2-6bdb546c89-m7dnw   1/1     Running   0          4s      10.104.0.38   gke-central-default-pool-b0d6c80c-lpmb   <none>           <none>
consumer2-6bdb546c89-rhphw   1/1     Running   0          4s      10.104.2.3    gke-central-pool-7-498fa03d-pqdj         <none>           <none>
consumer2-6bdb546c89-s8ls8   1/1     Running   0          4s      10.104.4.3    gke-central-pool-7-498fa03d-q8xj         <none>           <none>
$ kk get pdb
NAME            MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
consumer2-pdb   5               N/A               0                     3m39s
```



wywołujemy eviction poprzez likwidacje extra-node (poprzez scale do zera) 



```

$ kk scale deploy consumer --replicas=0

```



otrzymujemy stan gdzie część PODów niby blokuje extra-nodes, a zatem extra-nodes żyją nadal zaś PDB niby pilnuje aby ta sytuacja trwała 



```

NAME                         READY   STATUS    RESTARTS   AGE     IP            NODE                                     NOMINATED NODE   READINESS GATES

consumer2-6bdb546c89-7nz86   1/1     Running   0          2m50s   10.104.3.3    gke-central-pool-7-498fa03d-7b9l         <none>           <none>
consumer2-6bdb546c89-m6ph6   1/1     Running   0          2m50s   10.104.0.37   gke-central-default-pool-b0d6c80c-lpmb   <none>           <none>
consumer2-6bdb546c89-m7dnw   1/1     Running   0          2m50s   10.104.0.38   gke-central-default-pool-b0d6c80c-lpmb   <none>           <none>
consumer2-6bdb546c89-rhphw   1/1     Running   0          2m50s   10.104.2.3    gke-central-pool-7-498fa03d-pqdj         <none>           <none>
consumer2-6bdb546c89-s8ls8   1/1     Running   0          2m50s   10.104.4.3    gke-central-pool-7-498fa03d-q8xj         <none>           <none>
---
NAME                                     STATUS   ROLES    AGE     VERSION
gke-central-default-pool-b0d6c80c-9z88   Ready    <none>   47h     v1.21.10-gke.2000
gke-central-default-pool-b0d6c80c-lpmb   Ready    <none>   47h     v1.21.10-gke.2000
gke-central-pool-7-498fa03d-7b9l         Ready    <none>   4m54s   v1.21.10-gke.2000
gke-central-pool-7-498fa03d-pqdj         Ready    <none>   4m53s   v1.21.10-gke.2000
gke-central-pool-7-498fa03d-q8xj         Ready    <none>   4m55s   v1.21.10-gke.2000
---
NAME            MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
consumer2-pdb   5               N/A               0                     6m14s
```



w logu zaczyna sie pojawiać 



```

noDecisionStatus": {
    "noScaleDown": {
      "nodes": [
        {
          "reason": {
            "messageId": "no.scale.down.node.pod.not.enough.pdb",
            "parameters": [
              "consumer2-6bdb546c89-s8ls8"



```



nawet po godzinie oczekiwania sytuacja jest nadal stabilna (nikt nie rusza PODów i nie próbuje ich przegonić) , jedynie w logu k8s pojawia sie cyklicznie:



```

},
          "reason": {
            "messageId": "no.scale.down.node.pod.not.enough.pdb",
            "parameters": [
              "consumer2-6bdb546c89-rhphw"
            ]
          }



```



jak widać Deployment chroniony via PDB to skuteczna metoda zapobiegająca migracji (eviction) 



## kolejny TEST - jeśli zmienimy PDB z 5 na 4 to teoretycznie nastąpi migracja  (po jednym czyli bardzo powoli) i finalnie pozbycie się extra node-pooli



```

$ kk get pdb
NAME            MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
consumer2-pdb   5               N/A               0                     72m


$ cat pdb-consumer2.yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: consumer2-pdb
spec:
  minAvailable: 4
  selector:
    matchLabels:
      app: consumer2
$ kk apply -f pdb-consumer2.yaml

poddisruptionbudget.policy/consumer2-pdb configured
$ kk get pdb
NAME            MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
consumer2-pdb   4               N/A               1                     74m

```



i faktycznie powoli bo powoli ale finalnie wszystko się przemigrowało i extra-node-poola została zwolniona a więc i zlikwidowana 



```

NAME                         READY   STATUS    RESTARTS   AGE    IP            NODE                                     NOMINATED NODE   READINESS GATES
consumer2-6bdb546c89-bg9x9   1/1     Running   0          48m    10.104.0.39   gke-central-default-pool-b0d6c80c-lpmb   <none>           <none>
consumer2-6bdb546c89-m6ph6   1/1     Running   0          132m   10.104.0.37   gke-central-default-pool-b0d6c80c-lpmb   <none>           <none>
consumer2-6bdb546c89-m7dnw   1/1     Running   0          132m   10.104.0.38   gke-central-default-pool-b0d6c80c-lpmb   <none>           <none>
consumer2-6bdb546c89-pjmxc   1/1     Running   0          47m    10.104.1.23   gke-central-default-pool-b0d6c80c-9z88   <none>           <none>
consumer2-6bdb546c89-x5wh7   1/1     Running   0          37m    10.104.0.40   gke-central-default-pool-b0d6c80c-lpmb   <none>           <none>
---
NAME                                     STATUS   ROLES    AGE    VERSION
gke-central-default-pool-b0d6c80c-9z88   Ready    <none>   2d1h   v1.21.10-gke.2000
gke-central-default-pool-b0d6c80c-lpmb   Ready    <none>   2d1h   v1.21.10-gke.2000
---
---
NAME            MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
consumer2-pdb   4               N/A               1                     136m

```



wszystkie PODy są teraz na węzłach puli podstawowej, nody są tylko 2 (podstawowe), PDB 4+1 , jak widac powoli ale przepuściło po jednym 





# PRZYDATNE LINKI:



bardzo dobry link: https://stackoverflow.com/questions/66642406/gke-node-pool-with-autoscaling-does-not-scale-down

https://cloud.google.com/sdk/gcloud/reference/container/node-pools/create#--reservation

https://stackoverflow.com/questions/71176242/how-to-solve-pod-is-blocking-scale-down-because-its-a-non-daemonset-in-gke

https://stackoverflow.com/questions/57059552/prevent-kube-system-pods-from-running-on-a-specific-node

https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

https://cloud.google.com/kubernetes-engine/docs/how-to/node-taints#creating_a_node_pool_with_node_taints

https://spot.io/resources/kubernetes-autoscaling-3-methods-and-how-to-make-them-great/

https://stackoverflow.com/questions/58049333/how-can-i-manually-force-kubernetes-to-scale-up-the-nodes

https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-autoscaler

https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity

https://stackoverflow.com/questions/66642406/gke-node-pool-with-autoscaling-does-not-scale-down

https://stackoverflow.com/questions/65788593/gke-autoscaler-not-scaling-down-the-nodes














