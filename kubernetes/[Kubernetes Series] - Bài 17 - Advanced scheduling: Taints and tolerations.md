## Giới thiệu
Chào các bạn tới với series về kubernetes. Đây là bài thứ 17 trong series của mình, ở bài trước chúng ta đã nói về cách làm sao có thể để hệ thống của chúng ta tự động scale được. Ở bài này mình sẽ nói về cách làm sao ta có thể ngăn chặn một Pod được deploy tới node mà ta không mong muốn.

Khi ta chạy một cluster mà ta muốn triển khai nhiều môi trường trên đó, ví dụ như dev, testing, staging, production. Thì ta sẽ muốn server production của chúng ta sẽ là server mạnh với nhiều tài nguyên nhất, server dev thì chỉ ít tài nguyên thôi. Nên khi deploy thì ta sẽ phát hiện ra rằng ta có nhu cầu là ta không muốn những pod dev có thể được deploy lên worker node production và ngược lại.

Thì bên cạnh việc scheduler tự động chọn worker node nào để deploy pod tới đó, thì ta cũng có thể control được công việc đó. Những thao tác này được gọi là advanced scheduling, trong bài này chúng ta sẽ nói qua những kỹ thuật mà ta có thể chỉ định pod tới đúng worker node mà ta muốn. Phần đầu tiên đã xem qua hai thuộc tính sẽ giúp ta trong việc restrict pod được deploy tới node.

## Taints và tolerations
Hai đặc tính đầu tiên của advanced scheduling mà ta sẽ xem qua là taints và tolerations.

Taints được dùng để cấm một thằng pod được deploy tới worker node mà có taints. Ta sẽ dùng câu lệnh để đánh taints vào một worker node. Ví dụ ta có cụm worker node chạy môi trường production, ta sẽ đánh taints lên cụm worker node đó, và pod sẽ không thể được deploy lên cụm worker node này nữa. Cơ mà nếu vậy thì không có thằng pod nào có thể được deploy lên cụm này hết, vậy thì chẳng những worker node ta đánh taints sẽ thành thừa sao? Câu trả lời sẽ là không.

Tolerations được dùng để gán vào pod, và những pod nào mà được gán tolerations thỏa mãn với điều kiện taints thì sẽ được deploy lên những worker node mà có gán taints. Ví dụ như cụm worker node production ở trên ta gán taints cho nó, sau đó, ta sẽ chỉ định tolerations lên những pod production, thì bây giờ chỉ có những pod production có tolerations phù hợp mới có thể được deploy lên cụm worker node production, còn pod dev thì không.

Để dễ hiểu hơn thì ta sẽ xem qua một số taints và tolerations mặc định của kubernetes.

### Node's Taints
Trong kubernetes cluster, nếu các bạn để ý thì sẽ thấy là mặc định những pod của chúng ta sẽ không thể được deploy lên master node, mà chỉ có những system pod khi chúng ta khởi tạo một kubernetes cluster mới có thể được deploy tới master node. Lý do là vì master node đã được đánh taints, và những system pod được gán với những tolerations phù hợp với taints của master node, nên chỉ những system pod mới được deploy tới master node.

Ví dụ mình có một môi trường được triển khai bằng tool **kubeadm**.

```
$ kubectl get node
dev-node                      Ready    <none>                 246d   v1.22.3
production-node               Ready    <none>                 302d   v1.20.2
kube-master                   Ready    control-plane,master   304d   v1.20.1
```

Ta có thể kiểm tra taints của master node như sau:

```
$  kubectl describe node kube-master
Name:               kube-master
Roles:              control-plane,master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=kube-master
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node-role.kubernetes.io/master=
Annotations:        flannel.alpha.coreos.com/backend-data:
...
Taints:             node-role.kubernetes.io/master:NoSchedule
```

Ở trường Taints, bạn sẽ thấy nó có giá trị là `node-role.kubernetes.io/master:NoSchedule`, đây là taints mặc định được đánh vào master node. Taints sẽ có format như sau `<key>=<value>:<effect>`, với ví giá trị ở trên ta sẽ có key là  `role.kubernetes.io/master`, value là null, và effect là NoSchedule.

Giá trị taints này sẽ cấm những pod thông thường không thể deploy tới nó, và chỉ có system pod với giá trị tolerations là `role.kubernetes.io/master=:NoSchedule` mới có thể deploy lên nó được.

![image.png](./images/1fe5df8a-cee3-4537-9af8-a07606afc237.png)

### Pod's tolerations
Bạn liệt kê những pod nằm ở namespace là kube-system, và xem kiểm tra xem tolerations của nó có đúng như ta đã nói hay không.

```
$ kubectl get pod -n kube-system
configmapwatchers-controller-7d577784d-wl7wz   1/1     Running   5246       212d
coredns-5f4c5f68f9-mt2gz                       1/1     Running   2          246d
coredns-5f4c5f68f9-n4tc6                       1/1     Running   2          246d
etcd-kube-master                               1/1     Running   0          304d
kube-apiserver-kube-master                     1/1     Running   0          304d
...
```

Đây là những system pod được tạo ra khi mình dùng tool kubeadm, ta describe pod kube-apiserver-kube-master để xem thử tolerations của nó:

```
$ kubectl describe pod -n kube-system
...
Tolerations:  node-role.kubernetes.io/master=:NoSchedule
              node.alpha.kubernetes.io/notReady=:Exists:NoExecute
              node.alpha.kubernetes.io/unreachable=:Exists:NoExecute
...
```

Bạn sẽ thấy ở trường Tolerations nó có một giá trị là `node-role.kubernetes.io/master=:NoSchedule`, đây là giá trị tolerations để pod này có thể deploy được lên master node. Bạn để ý là ở tolerations ta có thêm dấu = nữa, thì taints và tolerations sẽ có cách hiển thị giá trị này khác nhau khi value = null.

### Taint effect
Trong taints và tolerations thì giá trị của key và value thì khá dễ hiểu, chúng chỉ là string mà thôi. Nhưng giá trị effect thì khác, ta sẽ có những giá trị effect sau đây:
+ NoSchedule: khi giá trị này được gán vào taints, nó sẽ không cho phép pod được deploy lên nó.
+ PreferNoSchedule: giá trị này cũng có tác dụng giống như NoSchedule, nhưng khác biệt một điểm là nếu Pod không thể được deploy tới bất kì node nào, mà node với effect là PreferNoSchedule có đủ tài nguyên để chạy pod, thì pod có thể được deploy lên nó.
+ NoExecute: không giống như NoSchedule và PreferNoSchedule chỉ có tác dụng khi pod được scheduling, thì giá trị effect này sẽ ảnh hưởng lên cả pod mà đang chạy trên node, nếu pod không có tolerations phù hợp với taints thì nó cũng sẽ bị gỡ khỏi node đó. Ví dụ ta có pod đang chạy trên một woker node, sau đó ta đánh taints node đó với effect là NoExecute, thì những pod nào đang chạy trên node đó mà không có tolerations phù hợp sẽ bị gỡ khỏi pod.

### Thêm Taints vào node
Như đã nói ở trên thì ta có thêm đánh taints vào một node, lấy ví dụ môi trường dev và production ở trên, cụm worker node production ta sẽ đánh taints cho nó như sau:

```
$ kubectl taint node production-node node-type=production:NoSchedule
node "production-node" tainted
```

Bây giờ khi ta tạo pod, ta sẽ thấy toàn bộ pod sẽ được deploy tới worker node dev.

```
$ kubectl run test --image busybox --replicas 5 -- sleep 99999
deployment/test created

$ kubectl get pod -o wide
NAME               READY  STATUS   RESTARTS  AGE  IP         NODE
test-196686-46ngl  1/1    Running  0         12s  10.47.0.1  dev-node
test-196686-73p89  1/1    Running  0         12s  10.47.0.7  dev-node
test-196686-77280  1/1    Running  0         12s  10.47.0.6  dev-node
test-196686-h9m8f  1/1    Running  0         12s  10.47.0.5  dev-node
test-196686-p85ll  1/1    Running  0         12s  10.47.0.4  dev-node
```

Để deploy pod tới production node, ta thêm tolerations vào nó, tạo một file tên là production-deployment.yaml với config như sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: prod
  template:
    metadata:
      labels:
        app: prod
    spec:
      containers:
        - image: luksa/kubia:v1
          name: nodejs
          resources:
            requests:
              cpu: 100m
      tolerations:
        - key: node-type
          Operator: Equal
          value: production
          effect: NoSchedule
```

Ta tạo deployment này và liệt kê pod của nó, ta sẽ thấy có pod sẽ được deploy lên production node.

```
$ kubectl apply -f production-deployment.yaml
deployment/prod created

$ kubectl get po -o wide
NAME               READY  STATUS   RESTARTS  AGE  IP          NODE
prod-350605-1ph5h  1/1    Running  0         12s  10.47.0.11  production-node
prod-350605-73p89  1/1    Running  0         12s  10.47.0.17  production-node
prod-350605-k7c8g  1/1    Running  0         12s  10.47.0.16  production-node
prod-350605-h9m8f  1/1    Running  0         12s  10.47.0.15  production-node
prod-350605-p85ll  1/1    Running  0         12s  10.47.0.14  production-node
```

### Hiều về cách sử dụng Taints và tolerations
Một node có thể có nhiều taints và một pod cũng có thể khai báo nhiều giá trị tolerations. Taints thì có giá trị key và effect là bắt buộc, còn value là optional. Ta chỉ định tolerations cho pod với Operator là Equal (mặc định) hoặc Exists.

Cách thứ nhất ta sử dụng taints và tolerations là ví dụ một kubernetes cluster  có nhiều môi trường ở trên.

Cách thứ hai ta có thể sử dụng taints và tolerations là cấu hình khi một node bị chết, thì bao lâu pod trên node đó sẽ được reschedule lại và được deploy ở một node khác, bằng cách dùng NoExecute effect. Ví dụ, khi ta describe một pod, ta có thể thấy nó có một số tolerations mặc định như sau:

```
...
Tolerations: node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
...
```

Giá trị này có nghĩa khi kubernetes Control Plane phát hiện một node bị chết, nó sẽ đánh taints cho node đó là `node.kubernetes.io/not-ready:NoExecute`, và sau đó tolerations của pod trên node bị chết này chỉ có giá trị trong vòng  300s, sau thời gian này tolerations sẽ bị xóa khỏi pod hiện tại, và do đó nó bị gỡ khỏi node vì không còn tolerations phù hợp.

![image.png](./images/78cc1db4-afb5-4724-a035-d15af7df91ad.png)

![image.png](./images/a8bdff17-e83a-490c-82c9-a6aefc061d5b.png)

## Kết luận
Vậy là ta đã tìm hiểu xong về ý nghĩa taints và tolerations, đây là cách đầu tiên trong advanced scheduling giúp ta ngăn chặn pod được deploy tới node cụ thể. Nếu có thắc mắc hoặc cần giải thích rõ thêm chỗ nào thì các bạn có thể hỏi dưới phần comment. Ở phần tiếp theo ta vẫn sẽ nói tiếp về advanced scheduling, các cách khác để giúp ta deploy pod lên node mà ta muốn.