## Giới thiệu
Chào các bạn tới với series về kubernetes. Đây là bài thứ 9 trong series của mình, ở những bài trước chúng ta đã dùng [Pod], [ReplicaSet], [Deployment] để deploy một ứng dụng hoặc nhiều ứng dụng mà chạy chung một template để tăng performance. Ở bài này chúng ta sẽ nói về StatefulSets, một resource sẽ giúp ta deploy một stateful application.

> Stateless application là một ứng dụng mà không có lưu trữ trạng thái của chính nó, hoặc là không có lưu trữ dữ liệu mà cần persistent storage. Ví dụ là một web server API mà không có lưu trữ hình ảnh, hoặc session login của user, thì đó là một stateless apps, bởi vì cho dù ta có xóa ứng dụng của ta và tạo lại nó bao nhiêu lần thì cũng không ảnh hưởng tới dữ liệu của người dùng. Bởi vì dữ liệu của ta được lưu trữ thông qua database, web server API chỉ kết nối với database và lưu trữ dữ liệu, chứ nó không có dữ liệu của chính nó. Một ví dụ nữa là command line app, nó không cần phải lưu trữ dữ liệu gì cả, tất cả những gì nó cần là xuất ra kết quả và không cần phải lưu lại kết quả đó. Những đặc tính của stateless app sẽ giúp nó dễ dàng scale hơn.

> Stateful application thì yêu cầu có trạng thái (state) của chính nó, và cần lưu lại state đó, hoặc yêu cầu phải có lưu trữ dữ liệu mà cần persistent storage, dữ liệu này sẽ được sử dụng bởi client và các app khác. Ví dụ như là database, nó sẽ có dữ liệu riêng của nó.

Ở trong kubernetes ta có thể deploy một stateful application bằng cách tạo Pod và config volume cho Pod, hoặc dùng PersistentVolumeClaim. Nhưng ta chỉ có thể tạo một single instance của Pod mà kết nối tới PersistentVolumeClaim đó. Vậy thì có thể dùng ReplicaSet để tạo replicated stateful app không? Cái ta muốn là sẽ tạo ra nhiều replicas của Pod, và mỗi Pod ta sẽ dùng một PersistentVolumeClaim riêng, để chạy được một ứng dụng distributed data store.

## Hạn chế của việc sử dụng ReplicaSet để tạo replicated stateful app
Vì ReplicaSet tạo nhiều pod replicas từ một Pod template, nên những Pod được replicated đó không khác với những Pod khác ngoại trừ tên và IP. Nếu ta config volume trong Pod template thì tất cả các Pod được replicated đều lưu trữ dữ liệu chung một storage.

![image.png](./images/470d3565-3bc7-4203-93fa-2064863ce49f.png)

Nên ta không thể sử dụng một ReplicaSet rồi set thuộc tính replicas của nó để chạy một ứng dụng  distributed data store được. Ta cần phải sử dụng cách khác.

### Tạo nhiều ReplicaSet chỉ có một Pod mỗi ReplicaSet
Ta tạo nhiều ReplicaSet và mỗi ReplicaSet đó sẽ có một template Pod khác nhau.

![image.png](./images/75226cb5-b7dd-456f-9baf-29a8dc218c0a.png)

Ta có thể dụng cách này để deploy một ứng dụng distributed data store. Nhưng đây không phải là cách tốt, ví dụ khí ta muốn scale ứng dụng của ta lên thì ta sẽ làm thế nào? Ta chỉ có cách là tạo thêm một ReplicaSet bằng tay, công việc này không tự động chút nào. Ta chọn kubernetes để chạy ứng dụng là vì muốn mọi thứ như scale đều được tự động và dễ dàng nhất.

### Cung cấp stable identity cho mỗi Pod
Đối với statefull application, ta cần định danh cho mỗi Pod, vì Pod có thể bị xóa và tạo lại bất cứ lúc nào, khi ReplicaSet thay thế một thằng Pod cũ bằng Pod mới thì Pod mới tạo ra sẽ có tên khác và IP khác. Cho dù dữ liệu ta vẫn còn đó và giống như Pod cũ, nhưng đối với một số ứng dụng, khi ta tạo Pod mới mà nó có một network identity mới (như địa chỉ IP) thì sẽ sinh ra nhiều vấn đề. Nên ta cần phải dùng Service để identity IP cho Pod, ta có bao nhiêu ReplicaSet thì ta sẽ cần tạo bấy nhiêu Service tương ứng.

![image.png](./images/e9bdf670-7117-45a9-852d-8778b4ff2016.png)

Khi ta có thêm Service, lúc này ta muốn scale lên, bên cạnh việc phải tạo một ReplicaSet mới, bây giờ ta cần phải tạo thêm một Service mới cho thằng ReplicaSet tương ứng nữa, gấp đôi công việc phải làm bằng tay.

Thì để giải quyết những vấn đề trên, có thể dễ dạng tạo nhiều replicated của Pod, mỗi thằng sẽ có một định danh riêng, và dễ dạng tự động scale mà không cần ta phải làm bằng tay quá nhiều, giúp ta dễ hơn trong việc tạo một ứng dụng distributed data store. Kubernetes cung cấp cho chúng ta một resource tên là StatefulSet.

## StatefulSets
Giống như ReplicaSet, StatefulSet là một resource giúp chúng ta chạy nhiều Pod mà cùng một template bằng cách set thuộc tính replicas, nhưng khác với ReplicaSet ở chỗ là Pod của StatefulSet sẽ được định danh chính xác và mỗi thằng sẽ có một stable network identity của riêng nó.

Mỗi Pod được tạo ra bởi StatefulSet sẽ được gán với một index, index này sẽ được sử dụng để định danh cho mỗi Pod. Và tên của Pod sẽ được đặt theo kiểu `<statefulset name>-<index>`, chứ không phải random như của ReplicaSet.

![image.png](./images/8a7216d3-039b-4a8a-b0cb-069d83961f88.png)

### Cách StatefulSets thay thế một Pod bị mất
Khi một Pod mà được quản lý bởi một StatefulSets bị mất (do bị ai đó xóa đi), thằng StatefulSets sẽ tạo ra một Pod mới để thay thế thằng cũ tương tự như cách làm của ReplicaSet, nhưng thằng Pod được tạo mới này sẽ có tên và hostname giống y như thằng cũ.

![image.png](./images/582b368f-8b9e-4c0f-8d45-79606f7986d4.png)

Còn thằng ReplicaSet thì sẽ tạo ra thằng Pod mới hoàn toàn khác với thằng cũ.

![image.png](./images/586533b0-2257-4edf-8f4b-2f8f486dcada.png)

### Cách StatefulSets scale Pod
Khi ta scale up Pod trong StatefulSets, nó sẽ tạo ra một Pod mới được đánh index là số tiếp theo của index hiện tại. Ví dụ StatefulSets đang có replicas bằng 2, sẽ có 2 Pod là `<pod-name>-0`,` <pod-name>-1`, khi ta scale up Pod lên bằng 3, Pod mới được tạo ra sẽ có tên là `<pod-name>-2`.

Tương tự như vậy với scale down, nó sẽ xóa Pod với index lớn nhất. Đối với StatefulSets thì khi ta scale up và scale down thì ta có thể biết chính xác tên của Pod sẽ được tạo ra hoặc xóa đi.

![image.png](./images/18c3953e-10eb-460a-aff9-37b0e4266837.png)

### Cung cấp storage riêng biệt cho mỗi Pod
Tới đây thì chúng ta đã biết cách StatefulSets định danh cho mỗi thằng Pod, vậy còn storage thì sao? Mỗi Pod của chúng ta cần có một storage của riêng nó, và khi ta scale down số lượng Pod và scale up lại thì thằng Pod tạo ra mà có index giống với thằng cũ thì vẫn giữ nguyên storage của nó như không phải tạo ra một thằng storage khác.

Thằng StatefulSets làm được việc đó bằng cách tách storage ra khỏi Pod bằng cách sử dụng [PersistentVolumeClaims](https://viblo.asia/p/kubernetes-series-bai-7-persistentvolumeclaims-tach-pod-ra-khoi-kien-truc-storage-ben-duoi-6J3Zgyeq5mB) mà ta đã nói ở bài 7. StatefulSets sẽ tạo ra PersistentVolumeClaims cho mỗi Pod và gắn nó vào cho từng Pod tương ứng. 

![image.png](./images/d25971b9-7df8-4bf2-adbe-7bfc9c8a6e3c.png)

Khi ta scale up Pod trong StatefulSets, thì sẽ có một Pod và một PersistentVolumeClaims mới được tạo ra, nhưng khi ta scale down, thì chỉ có thằng Pod bị xóa đi, thằng PersistentVolumeClaims vẫn giữ ở đó và không bị xóa. Để khi ta scale up lại thì thằng Pod vẫn được gắn đúng với PersistentVolumeClaims trước đó để dữ liệu của nó vẫn được giữ nguyên.

![image.png](./images/b0dcc259-13f3-4be8-84fd-ffb5bf1f8f20.png)

### Tạo một StatefulSets
Giờ ta sẽ tạo thử một StatefulSets. Tạo một file tên là kubia-statefulset.yaml với image **luksa/kubia-pet**, code của image như sau:

```javascript
const http = require('http');
const os = require('os');
const fs = require('fs');

const dataFile = "/var/data/kubia.txt";

function fileExists(file) {
  try {
    fs.statSync(file);
    return true;
  } catch (e) {
    return false;
  }
}

var handler = function(request, response) {
  if (request.method == 'POST') {
    var file = fs.createWriteStream(dataFile);
    file.on('open', function (fd) {
      request.pipe(file);
      console.log("New data has been received and stored.");
      response.writeHead(200);
      response.end("Data stored on pod " + os.hostname() + "\n");
    });
  } else {
    var data = fileExists(dataFile) ? fs.readFileSync(dataFile, 'utf8') : "No data posted yet";
    response.writeHead(200);
    response.write("You've hit " + os.hostname() + "\n");
    response.end("Data stored on this pod: " + data + "\n");
  }
};

var www = http.createServer(handler);
www.listen(8080);
```

Config của file kubia-statefulset.yaml:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  clusterIP: None
  selector:
    app: kubia
  ports:
    - name: http
      port: 80

---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: kubia
spec:
  serviceName: kubia # the name of service
  replicas: 2
  template: # pod template
    metadata:
      labels:
        app: kubia
    spec:
      containers:
        - name: kubia
          image: luksa/kubia-pet
          ports:
            - name: http
              containerPort: 8080
      volumeMounts:
        - name: data
          mountPath: /var/data
  volumeClaimTemplates: # pvc template
    - metadata:
        name: data
      spec:
        resources:
          requests:
            storage: 1Mi
        accessModes:
          - ReadWriteOnce
```

Trong file config này ta sẽ có một Headless Service và một StatefulSet tên là kubia. Trong config của StatefulSet thì ta phải chỉ định service name dùng để định danh network cho Pod, Pod template, và PersistentVolumeClaims template. Cái khác với config của ReplicaSet là ta cần khai báo thêm template cho PersistentVolumeClaims, StatefulSet sẽ dùng nó để tạo ra PVCs riêng cho từng Pod.

![image.png](./images/b150e2aa-647b-4589-8fef-23a5d662ffb9.png)

Tạo StatefulSet:

```
$ kubectl create -f kubia-statefulset.yaml
service "kubia" created
statefulset "kubia" created
```

Giờ ta list Pod xem thử:

```
$ kubectl get po
NAME     READY  STATUS             RESTARTS  AGE
kubia-0  1/1    Running            0         8s
kubia-1  0/1    ContainerCreating  0         2s
```

Ta thấy Pod chúng ta được tạo ra sẽ có tên được gán theo index, giờ ta list PVCs xem thử:

```
$ kubectl get pvc
NAME          STATUS  VOLUME  CAPACITY  ACCESSMODES  AGE
data-kubia-0  Bound   pv-0    0                      37s
data-kubia-1  Bound   pv-1    0                      37s
```

Tên của PersistentVolumeClaims sẽ được đặt theo tên mà ta chỉ định ở phần volumeClaimTemplate và nối với index. Ở đây thì ta đã thấy là mỗi Pod của chúng ta sẽ có định danh riêng và xài một PVCs riêng của nó. Đúng với thứ ta cần khi xây dựng một hệ thống lưu trữ phân tán.

### Tương tác với Pod bằng định danh của nó
Giờ ta sẽ thử tương tác với từng thằng Pod riêng lẻ. Đâu tiên ta dùng câu lệnh proxy:

```
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

Mở một terminal khác:

```
$ curl localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
You've hit kubia-0
Data stored on this pod: No data posted yet
$ curl -X POST -d "Hey there! This greeting was submitted to kubia-0." localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
Data stored on pod kubia-0
$ curl localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
You've hit kubia-0
Data stored on this pod: Hey there! This greeting was submitted to kubia-0.
```

Kết quả trả về in ra là ta đã kết nối với Pod kubia-0.

![image.png](./images/1ac11bcb-6af3-48d8-96bb-228596baf3b8.png)

Bây giờ ta xóa Pod để kiểm tra xem Pod mới được tạo ra có sử dụng đúng PVCs cũ hay không.

```
$ kubectl delete po kubia-0
pod "kubia-0" deleted
```

List Pod để xem thử quá trình nó bị xóa và tạo lại.

```
$ kubectl get po
NAME     READY  STATUS       RESTARTS  AGE
kubia-0  1/1    Terminating  0         3m
kubia-1  1/1    Running      0         3m
$  kubectl get po
NAME     READY  STATUS             RESTARTS  AGE
kubia-0  0/1    ContainerCreating  0         6s
kubia-1  1/1    Running            0         4m
$ kubectl get po
NAME     READY  STATUS   RESTARTS  AGE
kubia-0  1/1    Running  0         9s
kubia-1  1/1    Running  0         4m
```

![image.png](./images/723ad430-2afd-4404-a0db-0238a33be765.png)

Ta thấy rằng ở đây Pod mới của chúng ta tạo ra sẽ có định danh y như Pod cũ, đó là điều mà ta muốn, giờ ta thử xem dữ liệu trước đó của ta có còn ý nguyên trong Pod kubia-0 không.

```
$ curl localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
You've hit kubia-0
Data stored on this pod: Hey there! This greeting was submitted to kubia-0.
```

Dữ liệu của ta vẫn còn đây, đúng với thứ mà ta muốn. Như các bạn thấy thì dùng StatefulSet sẽ tạo ra cho các Pod được định danh và có PVCs cũng được định danh. Vậy còn network thì thế nào? Nghĩa là ta sẽ truy cập từng Pod cụ thể ra sao nếu không dùng proxy, ở các bài trước thì ta dùng Service để tương tác với Pod, mà thằng Service thì request của nó sẽ được gửi random tới những Pod phía sau chứ không phải chính xác một thằng Pod, ta muốn ta có thể request tới chính xác một thằng Pod.

Ở đây ta sẽ dùng một kĩ thuật gọi là Headless Service, như các bạn thấy trong file config ở trên ta có tạo một service mà chỉ định thuộc tính **clusterIP của nó là None**. Đây là cách ta sẽ tạo ra một Headless Service và định danh địa chỉ cho từng thằng Pod.

### Headless Service
Đối với thằng Service ClusterIP bình thường, thì khi ta tạo Service đó, nó sẽ tạo ra một thằng Virtual IP cho chính nó và một thằng DNS tương ứng cho VIP đó, và thằng VIP này sẽ mapping với những thằng Pod phía sau Service.

![image.png](./images/f8f0385d-7aae-4126-ab30-74f9b9d3b7dd.png)

Còn đối với thằng Headless Service, khi khai báo config ta sẽ chỉ định thuộc tính **clusterIP: None** cho nó, khi ta tạo một thằng Headless Service thì nó sẽ không tạo ra một Virtual IP cho chính nó, mà chỉ tạo ra một DNS. Sau đó, nó sẽ tạo ra DNS cho chính xác từng thằng Pod phía sau, và mapping DNS tới những thằng DNS của Pod phía sau nó. Ví dụ thằng kubia-0 thì sẽ có DNS tương ứng là **kubia-0.kubia.default.svc.cluster.local**.

![image.png](./images/ba91c942-982c-4a2f-bd11-67bd49c6edd0.png)

Và ta có thể truy cập thẳng tới Pod bên trong cluster bằng cách gọi tới DNS **kubia-0.kubia** và **kubia-1.kubia** nếu truy cập cùng namespace. Hoặc dùng DNS **kubia-0.kubia.default.svc.cluster.local** và **kubia-1.kubia.default.svc.cluster.local** nếu truy cập khác namespace.

Headless Service cho phép chúng ta có thể truy cập thẳng tới một Pod nhất định sử dụng DNS, thay vì truy cập qua DNS của Service rồi request của ta sẽ được dẫn tới một Pod random. Ta kết hợp Headless Service với StatefulSet để cung cấp cho Pod một stable network identity, và vì mỗi Pod có định danh bằng index riêng nên ta có thể biết chính xác Pod nào chúng ta cần gọi tới.

## Cách StatefulSet làm việc với node failures
Không giống như ReplicaSet, thằng StatefulSet sẽ đảm bảo việc không bao giờ tạo Pod mà có định danh giống nhau, nên khi một node failed. StatefulSet sẽ không tạo ra Pod mới trước khi chắc chắn rằng thằng cũ không còn nữa.

### Cách Pod bị gỡ đi khi node fail
Khi một node failed, thì các Pod trên node đó sẽ có status là Unknown.  Nếu sau một thời gian mà node không có sống lại, thì Pod trên node đó sẽ được tháo ra, status của nó sẽ được update thành Terminating.

Đối với ReplicaSet, vì các Pod của nó quản lý không có định danh nên tên của Pod sẽ không trùng nhau, nên khi một Pod của nó mà có status là Terminating, nó sẽ tạo ra một thằng mới, và khi thằng mới được tạo xong, bất kệ thằng cũ có bị Terminating hay chưa thì nó cũng bị xóa đi khỏi cluster.

Còn đối với StatefulSet, do Pod có định danh, nên khi Pod mới tạo ra sẽ có tên giống Pod cũ, nên khi một Pod nằm trên node failed và có status là Terminating, StatefulSet vẫn sẽ không tạo ra Pod mới cho tới khi chắc chắn ràng Pod đã bị xóa hoàn toàn. Nhưng vì node của ta đã die đi rồi, nên nó không thể báo lại cho kubernetes master là thằng Pod đó đã bị xóa đi thành công hay chưa được, nên Pod sẽ cứ ở trạng thái Terminating hoài, lúc này ta cần phải xóa Pod đi bằng tay.

### Mô phỏng việc một node bị die
Giờ ta sẽ xem qua ví dụ cho dễ hiểu hơn. Ở ví dụ này ta sẽ có 3 con VM nằm trên GCP, ta sẽ ssh vào một con và mô phỏng thằng VM đó bị die:

```
$ gcloud compute ssh gke-kubia-default-pool-32a2cac8-m0g1
$ sudo ifconfig eth0 down
```

Lúc này ta kiểm tra status của 3 worker:

```
$ kubectl get node
NAME                                  STATUS    AGE  VERSION
gke-kubia-default-pool-32a2cac8-596v  Ready     16m  v1.6.2
gke-kubia-default-pool-32a2cac8-m0g1  NotReady  16m  v1.6.2
gke-kubia-default-pool-32a2cac8-sgl7  Ready     16m  v1.6.2
```

Ta sẽ thấy có một node chuyển status thành NotReady. Lúc này ta list pod ra thử:

```
$ kubectl get po
NAME     READY  STATUS   RESTARTS AGE
kubia-0  1/1    Unknown  0        15m
kubia-1  1/1    Running  0        14m
kubia-2  1/1    Running  0        13m
```

Như các bạn thấy thì status của Pod lúc này là Unknown. Sau một vài phút ta describe pod  thì sẽ thấy status của Pod chuyển thành Terminating.

```
$ kubectl describe po kubia-0
Name:      kubia-0
Namespace: default
Node:      gke-kubia-default-pool-32a2cac8-m0g1/10.132.0.2
...
Status:    Terminating (expires Tue, 23 May 2017 15:06:09 +0200)
Reason:    NodeLost
Message:   Node gke-kubia-default-pool-32a2cac8-m0g1 which was running pod kubia-0 is unresponsive
```

Lúc này để Pod có thể thay thế được, ta phải xóa Pod bằng tay, nhưng vì node ta đã die, nên câu lệnh delete bình thường sẽ không chạy được, mà ta cần dùng:

```
$ kubectl delete po kubia-0 --force --grace-period 0
```

Sau đó ta list pod ra sẽ thấy là pod mới sẽ được tạo ra:

```
$ kubectl get po
NAME     READY  STATUS             RESTARTS  AGE
kubia-0  1/1    ContainerCreating  0         21m
kubia-1  1/1    Running            0         20m
kubia-2  1/1    Running            0         19m
```

## Kết luận
Vậy là ta đã tìm hiểu xong về StatefulSet, dùng StatefulSet cho stateful application và database, nên dùng StatefulSet để chạy database thay vì dùng ReplicaSet hoặc Deployment. Nếu có thắc mắc hoặc cần giải thích rõ thêm chỗ nào thì các bạn có thể hỏi dưới phần comment. Tới bài này thì chúng ta đã tìm hiểu hết về các resource cơ bản để deploy một container trong kubernetes.  Ở những bài tiếp theo chúng ta sẽ đi sâu về cấu trúc bên trong của Kubernetes Cluster. Đầu tiên ta sẽ nói về cách để container có thể truy cập metadata của chính nó bằng cách dùng **Downward API**.