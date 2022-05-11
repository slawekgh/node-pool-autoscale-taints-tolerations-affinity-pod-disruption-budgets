# node-pool-autoscale-taints-tolerations-affinity-pod-disruption-budgets

jak w nazwie repo - omawiamy tu:
- node-pools z opcją autoskalowania
- taints
- toleration na taints 
- affinity, POD affinity, AntiAffinity, NodeAffinity
- eviction 
- PodDisruptionBudgets



# TYT1

aaa


## TYT2

aaaa

### TYT3 

```
terminal
block
```
# GKE-Node-Pools z autoskalowaniem 

powołanie auto-scale pool z taint (extranodes=mlops) via gcloud :

```

gcloud container node-pools create pool-6 --cluster central --node-taints=extranodes=mlops:NoSchedule --enable-autoscaling --max-nodes=3 --min-nodes=0

```

chwile po powołaniu nowej dodatkowej auto-scaled-node-pool pojawiają się na niej pody (te Pody żyją przez jakiś czas, potem znikają i znikają nody) 

```

slawek_wolak@cs-470434381902-default:~$ kubectl get po -A -o wide | grep pool-6
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

slawek_wolak@cs-470434381902-default:~/AUTOSCALER-GKE$ kk get daemonsets -A
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


