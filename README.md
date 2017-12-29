## Day 11 - Service (1)

### 本日共賞

* 如何存取 Pod
* Service 物件

### 希望你知道
* [建構組件](https://ithelp.ithome.com.tw/articles/10193513)

<br/>

#### 如何存取 Pod

在 [Day 10 - 建構物件](https://ithelp.ithome.com.tw/articles/10193513) 一文中，我們成功的

* 將 Deployment 物件部署到 k8s 叢集中
* 運行了三個 Pod 

查看一下 Pod 的運行狀態

```
$ kubectl get pods
NAME                    READY     STATUS    RESTARTS   AGE
nginx-75f4785b7-6fbd4   1/1       Running   0          1m
nginx-75f4785b7-g284n   1/1       Running   0          1m
nginx-75f4785b7-vd4tt   1/1       Running   0          1m
```

接下來就是討論如何存取 Pod。由於每個 Pod 被建立的時候 k8s 會動態的分配 IP 給每個 Pod，最簡單的方式當然就是直接透過 IP 存取。

![](https://ithelp.ithome.com.tw/upload/images/20171224/20107062pfHyNYs2c0.png)

但由於 Pod 有臨時的特性，因此當 Pod 發生錯誤需要重新被產生時 IP 也會被重新分配，如果透過上述方式，存取 Pod 將會相當的不方便，因為使用者要一直更新 IP 才知道該如何正確的存取 Pod。

![](https://ithelp.ithome.com.tw/upload/images/20171224/2010706249Dniks10n.png)

> 為什麼其中一個 Pod 壞掉 k8s 會重新幫我們建立一個 Pod 呢？還記得我們說過，k8s 會盡力去滿足我們想要物件的狀態嗎？當 Pod 發生問題無法存取的時候，k8s 會察覺到系統的變化也會發現運行三個 Pod 是使用者想要的狀態，因此 k8s 就會再重新幫我們建立一個新的 Pod 以符合使用者想要的狀態。

因此 k8s 引用了一個更高階的物件 Service 來處理連接到 Pod。

>如果想查看 Pod 被分配的 IP 可以用下列指令，請記得把 Pod 名稱 (nginx-75f4785b7-6fbd4) 置換成正確名字
>
> ```bash
> $ kubectl get pods nginx-75f4785b7-6fbd4 -o jsonpath --template={.status.podIP}
> ```

#### Service

Service 物件利用 Label 與 Selector 來決定如何存取 (policy) 以及可以存取哪些 Pod (logical group)。

![](https://ithelp.ithome.com.tw/upload/images/20171224/20107062EhIddBl3hS.png)

如上圖所示，我們在使用者跟 Pod 中間加了一個 Service 物件。透過 Selector (app==nginx) ，而使用者可以透過 Service 存取兩個 Pod。當其中一個 Pod 故障需要被重新產生時，新產生的 Pod 也能夠透過 Service 正確存取。

![](https://ithelp.ithome.com.tw/upload/images/20171224/20107062kaRDqF4wQl.png)

> 忘記 Label 跟 Selector 了嗎？看看 [Day 10 - 建構物件](https://ithelp.ithome.com.tw/articles/10193513) 吧


#### Service Type

Service 物件根據使用方式有三個不同的形態

* 僅供叢集內部存取：ClusterIP
* 供叢集及外部存取：NodePort, LoadBalancer
* 對應叢集外其他資源：ExternalName

由於我們需要透過 Service 存取 Pod，因此，NodePort 形態的 Service 正符合我們的需求。

> LoadBalancer 一般需搭配雲端服務使用 (AWS, GCP, ...)。
> 
> ExternalName 是透過 CNAME 與外部連接

首先，建立 service.yaml

```
# service.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
```

* `kind: Service`：指定為 Service 物件
* `spec.type: NodePort`：指定 Service 型態。
* `spec.selector.app: nginx`：指定 Service 綁定 Label 有 app: nginx 的 Pod

>請注意，如果沒有指定則預設會是 ClusterIP 型態則僅供叢集內部存取，外部無法存取

接著，部署到 k8s

```bash
$ kubectl apply -f service.yaml
service "web" created
```

再透過指令查看 Service

```bash
$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP        1d
web          NodePort    10.0.0.221   <none>        80:31063/TCP   6s
```

這裡會看到兩個 Service 物件，其中 web 就是我們新增加的物件。

> 無特別指定下，k8s 會自動配置 30000 ~ 32767 區間的 port 供 Service 使用

這裡有點複雜，我們用一個圖來說明 k8s 叢集中的狀態

![](https://ithelp.ithome.com.tw/upload/images/20171224/20107062CHkWwjajI2.png)

k8s 叢集中，可能包含了很多台主機 (Worker Node)。由於我們是使用 minikube 來操作 k8s 故可以把 minikube 想像成 k8s 叢集中的一個 Node 即圖中的 Worker Node 2。

當我們部署了一個名為 web 的 Service 物件時，在 k8s 內部就會建立一個對應的 web: 10.0.0.242 (IP 會自動配置不用擔心)，並且將所有的 Node 的 32080 port 對應到 web 這個 Service。因此無論使用者從哪一個 Node 存取 32080 port 都會被指到 web 並透過 web 找到對應的 Pod (app: nginx)。

> Pod 可能會分散在不同的 Node，如同上圖中 other-svc 對應的 Pod，但是不用擔心存取問題 K8s 會幫你處理。這也正是抽象化的好處，存取將透過單一入口而不需要知道各個主機的設定。


關於更多的 Service 說明，可以參考 [官方文件](https://kubernetes.io/docs/concepts/services-networking/service/)

本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day11/](https://jlptf.github.io/ironman2018-day11/)