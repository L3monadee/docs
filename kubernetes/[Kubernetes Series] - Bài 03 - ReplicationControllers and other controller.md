## Giới thiệu
Chào các bạn tới với series về kubernetes. Đây là bài thứ ba trong series của mình, trong bài này mình sẽ nói về Kubernetes ReplicationControllers, ReplicaSets, DaemonSet. Ở bài trước chúng ta đã nói về pod, như chúng ta đã biết pod là thành phần cơ bản nhất để deploy application, nhưng trong thực tế ta sẽ không chạy pod trực tiếp, vì nó sẽ gặp nhiều hạn chế, mà chúng ta sẽ tạo những resource khác như ReplicationControllers hoặc ReplicaSets, và nó sẽ tự động tạo và quản lý pod

## ReplicationControllers là gì?
ReplicationControllers là một resource mà sẽ tạo và quản lý pod, và chắc chắn là số lượng pod nó quản lý không thay đổi và kept running. ReplicationControllers sẽ tạo số lượng pod bằng với số ta chỉ định ở thuộc tính replicas và quản lý pod thông qua labels của pod
![image.png](./images/18bf20c6-7160-4987-9fae-66ec6edc6d33.png)

### Tại sao ta nên dùng ReplicationControllers để chạy pod?
Chúng ta đã biết pod nó sẽ giám sát container và tự động restart lại container khi nó fail 
![image.png](./images/5c1b6cdb-626b-41d3-ad56-3d0a0df59651.png)

Vậy trong trường hợp toàn bộ worker node của chúng ta fail thì sẽ thế nào? pod nó sẽ không thể chạy nữa, và application của chúng ta sẽ downtime với người dùng
![image.png](./images/cfb88987-43ef-49d7-9835-050482618b2b.png)

Nếu chúng ta chạy cluster với hơn 1 worker node, RC sẽ giúp chúng ta giải quyết vấn đề này. Vì RC sẽ chắc chắn rằng số lượng pod mà nó tạo ra không thay đổi, nên ví dụ khi ta tạo một thằng RC với số lượng replicas = 1, RC sẽ tạo 1 pod và giám sát nó, khi một thằng worker node die, nếu pod của thằng RC quản lý có nằm trong worker node đó, thì lúc này thằng RC sẽ phát hiện ra là số lượng pod của nó bằng 0, và nó sẽ tạo ra thằng pod ở một worker node khác để đạt lại được số lượng 1
![image.png](./images/e47d1df7-93cb-48fd-abd1-775f200665c1.png)

Dưới đây là hình minh họa cách hoạt động của RC
![image.png](./images/365e0eb8-8cad-430d-aeaa-efd87db31c61.png)

Sử dụng ReplicationControllers để chạy pod sẽ giúp ứng dụng của chúng ta luôn luôn availability nhất có thể. Ngoài ra ta có thể tăng performance của ứng dụng bằng cách chỉ định số lượng replicas trong RC để RC tạo ra nhiều pod chạy cùng một version của ứng dụng.

Ví dụ ta có một webservice, nếu ta chỉ deploy một pod để chạy ứng dụng, thì ta chỉ có 1 container để xử lý request của user, nhưng nếu ta dùng RC và chỉ định replicas = 3, ta sẽ có 3 pod chạy 3 container của ứng dụng, và request của user sẽ được gửi tới 1 trong 3 pod này, giúp quá trình xử lý của chúng ta tăng gấp 3 lần
![image.png](./images/a6db7abf-1452-4463-8345-43f9b640b307.png)

## Tạo một ReplicationControllers
Vậy là xong phần lý thuyết, chúng ta bắt tay vào phần thực hành nào. Tạo một file tên là hello-rc.yaml và copy phần config sau vào:
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: hello-rc
spec:
  replicas: 2 # number of the pod
  selector: # The pod selector determining what pods the RC is operating on
    app: hello-kube # label value
  template: # pod template
    metadata:
      labels:
        app: hello-kube # label value
    spec:
      containers:
      - image: 080196/hello-kube # image used to run container
        name: hello-kube # name of the container
        ports:
          - containerPort: 3000 # pod of the container
```
Cấu trúc của một file config RC sẽ gồm 3 phần chính như sau:
+ label selector: sẽ chỉ định pod nào sẽ được RC giám sát
+ replica count: số lượng pod sẽ được tạo
+ pod template: config của pod sẽ được tạo

![image.png](./images/3604e4cd-1a95-4b60-8266-1b592272533e.png)

Bây giờ ta tạo RC nào

`kubectl apply -f hello-rc.yaml`

![image.png](./images/a61bd091-6bd2-4260-8fef-7dbe7d8772f5.png)

Kiểm tra xem rc của chúng ta đã chạy thành công hay chưa

`kubectl get rc`

![image.png](./images/e84ae2b2-dff6-4f59-8349-a59ada5257ff.png)

Nếu số lượng ở cột READY bằng với số lượng DESIRED thì chúng ta đã chạy RC thành công. Bây giờ ta kiểm tra số lượng pod được tạo ra bởi RC có đúng với số lượng chỉ định ở replicas như lý thuyết hay không

`kubectl get pod`

![image.png](./images/4ff9929a-2d9b-4013-ab65-ad15a4c5e102.png)

Ở đây thì tên pod của bạn hiển thị sẽ khác với trong hình nhé, nếu bạn thấy có 2 pod thì số lượng pod được tạo ra bởi RC đã chính xác, tên của pod được tạo ra bởi RC sẽ theo kiểu **`<replicationcontroller name>-<random>`**. Giờ ta sẽ xóa thử một thằng pod xem RC có tạo lại một thằng pod khác cho chúng ta như lý thuyết không. Nhớ chỉ định đúng tên pod của bạn

`kubectl delete pod hello-rc-c6l8k`

![image.png](./images/bc32dbc1-d757-4f26-85f3-b20c0e854551.png)

Mở cửa sổ terminal khác và gõ câu lệnh

`kubectl get pod`

![image.png](./images/e9f36658-5dc9-4a15-9cae-d397fab7b33d.png)

Bạn sẽ thấy là có một thằng cũ đang bị xóa đi, và cũng lúc đó, sẽ có một thằng pod mới được RC tạo ra, ở đây pod mới của mình tên là **hello-rc-mftfd**. Hoạt động của thằng RC được mình họa như hình sau

![image.png](./images/1a6dd461-ebf1-46de-a42c-b580531937d6.png)

### Thay đổi template của pod
Bạn có thể thay đổi template của pod và cập nhật lại RC, nhưng nó sẽ không apply cho những thằng pod hiện tại, muốn pod của bạn cập nhật template mới, bạn phải xóa hết pod để RC tạo ra pod mới, hoặc xóa RC và tạo lại

![image.png](./images/52e144c9-0c89-47f9-8a64-618556e02cb4.png)

Vậy là chúng ta đã chạy được RC, bây giờ ta xóa đi nhé, để xóa RC thì các bạn dùng câu lệnh

`kubectl delete rc hello-rc`

Khi bạn xóa RC thì những thằng pod nó quản lý cũng sẽ bị xóa theo

![image.png](./images/aa139690-dc3f-4357-8f8d-31cea4b59570.png)

Như các bạn thấy thì ReplicationController là một resource rất hữu ích để chúng ta deploy pod

## Sử dụng ReplicaSets thay thế RC
Đây là một resource tương tự như RC, nhưng nó là một phiên bản mới hơn của RC và sẽ được sử dụng để thay thế RC. Chúng ta sẽ dùng ReplicaSets (RS) để deploy pod thay vì dùng RC, ở bài này mình nói về RC trước để chúng ta hiểu được nguồn gốc của nó, để đi phỏng vấn có bị hỏi vẫn biết trả lời :grin:.

Bây giờ ta sẽ tạo thử một thằng RS, config nó vẫn giống RC, chỉ khác một vài phần. Tạo file tên là hello-rs.yaml, copy config sau vào:
```yaml
apiVersion: apps/v1 # change version API
kind: ReplicaSet # change resource name
metadata:
  name: hello-rs
spec:
  replicas: 2
  selector:
    matchLabels: # change here 
      app: hello-kube
  template:
    metadata:
      labels:
        app: hello-kube
    spec:
      containers:
      - image: 080196/hello-kube
        name: hello-kube
        ports:
          - containerPort: 3000
```
`kubectl apply -f hello-rs.yaml`

Kiểm tra RS của chúng ta có chạy thành công hay không

`kubectl get rs`

![image.png](./images/00125b69-1c3b-40d4-b7f2-986bf65d12dd.png)

Nếu có 2 pod tạo ra là chúng ta đã chạy RS thành công. Đề xóa RS ta dùng câu lệnh

`kubectl delete rs hello-rs`

![image.png](./images/037587c2-0b14-4b68-8b83-1dc31468304e.png)

### So sánh ReplicaSets và ReplicationController
RS và RC sẽ hoạt động tương tự nhau. Nhưng RS linh hoạt hơn ở phần label selector, trong khi label selector thằng RC chỉ có thể chọn pod mà hoàn toàn giống với label nó chỉ định, thì thằng RS sẽ cho phép dùng một số expressions hoặc matching để chọn pod nó quản lý.

Ví dụ, thằng RC không thể nào match với pod mà có env=production và env=testing cùng lúc được, trong khi thằng RS có thể, bằng cách chỉ định label selector như **env=*** . Ngoài ra, ta có thể dùng **operators** với thuộc tính **matchExpressions** như sau:
```yaml
selector:
  matchExpressions:
    - key: app
      operator: In
      values:
        - kubia
```

Có 4 operators cơ bản là: In, NotIn, Exists, DoesNotExist

## Sử dụng DaemonSets để chạy chính xác một pod trên một worker node
Đây là một resource khác của kube, giống như RS, nó cũng sẽ giám xác và quản lý pod theo lables. Nhưng thằng RS thì pod có thể deploy ở bất cứ node nào, và trong một node có thể chạy mấy pod cũng được. Còn thằng DaemonSets này sẽ deploy tới mỗi thằng node một pod duy nhất, và chắc chắn có bao nhiêu node sẽ có mấy nhiêu pod, nó sẽ không có thuộc tính replicas

![image.png](./images/f3e48546-9b72-41bf-aa74-79cc354f06e6.png)

Ứng dụng của thằng DaemonSets này sẽ được dùng trong việc logging và monitoring. Lúc này thì chúng ta sẽ chỉ muốn có một pod monitoring ở mỗi node. Và ta cũng có thể đánh label vào trong một thằng woker node bằng cách sử dụng câu lệnh

`kubectl label nodes <your-node-name> disk=ssd`

Sau đó ta có thể chỉ định thêm vào config của DaemonSets ở cột nodeSelector với disk=ssd. Chỉ deploy thằng pod trên node có ổ đĩa ssd. Đây là config ví dụ

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:
        disk: ssd
      containers:
        - name: main
          image: luksa/ssd-monitor
```

![image.png](./images/7fc7135d-8a28-4375-8741-5e789b4fcbd4.png)

Ở đây thì chúng ta sẽ không thực hành tạo DaemonSets vì chúng ta cần môi trường có nhiều worker node để demo resource này. Các bạn có thể đọc nhiều hơn về DaemonSets [ở đây](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

## Kết luận
Vậy là chúng ta đã tìm hiểu xong về ReplicationController, ReplicaSets, và DaemonSets. Đây là những resource dùng để deploy pod và giúp cho ứng dụng của chúng ta powerful hơn, high availability hơn. Cảm ơn các bạn đã đọc bài của mình. Nếu có thắc mắc hoặc cần giải thích rõ thêm chỗ nào thì các bạn có thể hỏi dưới phần comment. Hẹn gặp lại các bạn ở bài tiếp theo mình sẽ nói về **Kubernetes Services**, đây là một resource quan trọng để expose traffic ra bên ngoài