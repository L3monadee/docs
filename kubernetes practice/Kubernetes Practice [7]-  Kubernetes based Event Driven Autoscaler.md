## Giới thiệu
Chào mọi người đến với series practice về kubernetes. Ở bài này chúng ta sẽ tìm hiểu về KEDA, một resource giúp ta trong việc scale các pod container trong Kubernetes một cách rất dễ dàng và cover rất nhiều use case phổ biến.

![image.png](./images/7e272554-6da9-4628-910b-0fb7d5622432.png)

Khi ta sử dụng kubernetes, để scale một pod container ta thường sử dụng [Horizontal Pod Autoscaling](https://viblo.asia/p/kubernetes-series-bai-16-automatic-scaling-pod-va-cluster-YWOZrGyRlQ0). Nhưng Horizontal Pod Autoscaling chỉ hỗ trợ ta scale container theo một vài metric đơn giản, ví dụ như là cpu và memory. Nếu ta muốn scale container theo các metric khác thì ta phải viết custom metrics khá phức tạp, ví dụ như là scale container dựa theo Kafka topic.

Thay vì phải tự viết thì ta có thể sử dụng KEDA để làm công việc này.

## KEDA
KEDA (Kubernetes-based Event Driven Autoscaler) là một Custom Resource Definition mà ta có thể thêm vào một kubernetes cluster có sẵn. KEDA sẽ tương tác với Horizontal Pod Autoscaling ở bên dưới để mở rộng thêm function của HPA.

Với KEDA ta có thể scale một ứng dụng dựa vào event-driven, ví dụ như số lượng message bên trong RabbitMQ đạt tới một số lượng nhất định.

### High-level architecture
Thì để hiểu rõ hơn về KEDA thì ta sẽ coi sơ qua kiến trúc high-level của nó.

![image.png](./images/599787b5-63f1-4258-9012-d9aaaf23667a.png)

KEDA bao gồm các thành phần cơ bản sau:
+ Metrics adapter: thành phần hành động tương tự như [kubernetes metrics server](https://github.com/kubernetes-sigs/metrics-server), mà sẽ expose các rich event data như queue length hoặc stream lag tới Horizontal Pod Autoscaler để xử lý việc scale.
+ Controller: thành phần thực hiện việc scale container về 0.
+ Scaler: đóng vai trò là một connector để kết nối tới external event source như kafka, rabbitmq và thu thập metric về.
+ ScaledObject: định nghĩa quan hệ giữa event source và workload resource (Deployment, StatefulSet).

### Event sources.
KEDA hỗ trợ rất nhiều event sources, hỗ trợ với cả các resource của AWS và AZURE.

![image.png](./images/19faae86-ee75-4844-9510-372fce4e408b.png)

Các bạn có thể xem tất cả các event sources mà KEDA hỗ trợ [ở đây](https://keda.sh/docs/2.6/concepts/#event-sources-and-scalers).

Oke, ta nói lý thuyết đủ rồi, tiếp theo ta sẽ install KEDA lên trên kubernetes cluster của ta và làm thử một ví dụ scale theo RabbitMQ.

## Install KEDA with Helm
Ta sẽ dùng [Helm](https://helm.sh/) để cài KEDA, nếu các bạn chưa install Helm thì install Helm như sau.

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

Sau đó ta chạy các lệnh sau.
1. Add Helm repo

```
helm repo add kedacore https://kedacore.github.io/charts
```

2. Update Helm repo

```
helm repo update
```

3. Install keda Helm chart

```
helm install keda kedacore/keda --namespace keda --create-namespace
```

```
NAME: keda
LAST DEPLOYED: Wed Apr 13 17:22:29 2022
NAMESPACE: keda
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Kiểm tra CRDs đã chạy chưa.

```
kubectl get all -n keda

NAME                                                   READY   STATUS    RESTARTS   AGE
pod/keda-operator-778cf49bcf-k8n2m                     1/1     Running   0          77s
pod/keda-operator-metrics-apiserver-5ccf7b74dd-f4c9m   1/1     Running   0          77s

NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/keda-operator-metrics-apiserver   ClusterIP   10.102.19.207   <none>        443/TCP,80/TCP   77s

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/keda-operator                     1/1     1            1           77s
deployment.apps/keda-operator-metrics-apiserver   1/1     1            1           77s

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/keda-operator-778cf49bcf                     1         1         1       77s
replicaset.apps/keda-operator-metrics-apiserver-5ccf7b74dd   1         1         1       77s
```

## Scale with RabbitMQ
Ok, giờ ta sẽ tạo rabbitmq + consumer, sau đó tạo một ScaledObject mà chỉ định thông số để consumer scale theo số lượng message.

Các bạn tải code ở github sau https://github.com/hoalongnatsu/kubernetes-practice.

Tạo RabbitMQ.

```
kubectl apply -f rabbitmq.yaml
```

```rabbitmq.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rabbitmq
  labels:
    component: rabbitmq
data:
  rabbitmq.conf: |
    default_user = k8s-practice
    default_pass = k8s-practice

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rabbitmq
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      component: rabbitmq
  template:
    metadata:
      labels:
        component: rabbitmq
    spec:
      containers:
        - image: rabbitmq
          name: rabbitmq
          ports:
            - containerPort: 5672
          env:
            - name: RABBITMQ_NODENAME
              value: k8s-practice
          volumeMounts:
            - name: config
              mountPath: /etc/rabbitmq
      volumes:
        - name: config
          configMap:
            name: rabbitmq

---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
spec:
  selector:
    component: rabbitmq
  ports:
    - port: 5672
```

Tạo consumer.
```
kubectl apply -f share-consume-queue.yaml
```

```share-consume-queue.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: share-consume-queue
data:
  RABBITMQ_HOST: rabbitmq
  RABBITMQ_PORT: "5672"
  RABBITMQ_USER: k8s-practice
  RABBITMQ_PASS: k8s-practice

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: share-consume-queue
  labels:
    component: share-consume-queue
spec:
  selector:
    matchLabels:
      component: share-consume-queue
  template:
    metadata:
      labels:
        component: share-consume-queue
    spec:
      containers:
        - name: share-consume-queue
          image: 080196/k8s-practice-keda-consumer
          envFrom:
            - configMapRef:
                name: share-consume-queue
```

Kiểm tra consumer mà ta đã tạo.

```
$ kubectl get deployment

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
rabbitmq              1/1     1            1           10m
share-consume-queue   1/1     1            1           12s
```

Sau đó ta tạo ScaledObject.

```
kubectl apply -f scaled-object.yaml
```

Config của ScaledObject như sau.

```scaled-object.yaml
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq-secret
data:
  MQ_HOST: YW1xcDovL2s4cy1wcmFjdGljZTprOHMtcHJhY3RpY2VAcmFiYml0bXEuZGVmYXVsdDo1NjcyL3Zob3N0 # base64 encoded value of format amqp://k8s-practice:k8s-practice@rabbitmq.default:5672/vhost

---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: rabbitmq-trigger-auth
spec:
  secretTargetRef:
    - parameter: host
      name: rabbitmq-secret # secret name
      key: MQ_HOST # secret key

---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: share-consume-queue
spec:
  scaleTargetRef:
    name: share-consume-queue
  triggers:
    - type: rabbitmq
      metadata:
        mode: QueueLength
        queueName: scale_out
        value: "5"
      authenticationRef:
        name: rabbitmq-trigger-auth
```

Các bạn có thể đọc full cấu hình ở đây https://keda.sh/docs/2.6/scalers/rabbitmq-queue. Mình sẽ giải thích sơ qua các resource cần thiết để ta tạo ScaledObject với RabbitMQ.

Đầu tiên là Secret dùng để lưu connection tới  rabbitmq, giá trị MQ_HOST là chuỗi base64 của connection string `amqp://k8s-practice:k8s-practice@rabbitmq.default:5672/vhost`, ta tạo ra nó bằng câu lệnh encode.

```
echo -n "amqp://k8s-practice:k8s-practice@rabbitmq.default:5672/vhost" | base64 -w 0
```

Thứ hai là TriggerAuthentication resource, được Scaler dùng để tạo connection tới rabbitmq, giá trị của **secretTargetRef** sẽ gồm 3 thuộc tính là parameter, name, key. Với giá trị name là tên của Secret, key là giá trị key trong trường data của Secret.

Cuối cùng là ScaledObject, thành phần chính để config scale cho consumer. Với giá trị của **scaleTargetRef** là resource ta chọn để scale, còn giá trị triggers dùng để định nghĩa event mà ta dùng để thực hiện scale.

```yaml
- type: rabbitmq
  metadata:
    mode: QueueLength
    queueName: scale_out
    value: "5"
    vhostName: /
  authenticationRef:
    name: rabbitmq-trigger-auth
```

Thuộc tính **type** ta chọn là rabbitmq, **metadata.mode** ta chỉ định giá trị là QueueLength, **metadata.queueName** là tên của queue ta theo dõi message của nó, **mode.value** là số lượng message tương ứng với một replica, số lượng replica sẽ được tính dựa theo số này. Ví dụ số lượng message là 30 thì ta sẽ scale lên 6 replica.

Giá trị của trường **authenticationRef** ta là tên của TriggerAuthentication. Sau khi ta tạo ScaledObject xong, lúc này nếu bạn kiểm tra lại Deployment thì bạn sẽ thấy share-consume-queue của ta đã scale xuống còn 0 replica.

```
$ kubectl get deployment

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
rabbitmq              1/1     1            1           19m
share-consume-queue   0/0     0            0           17m
```

**Một điểm mà KEDA khác với thằng HPA bình thường là nó cho phép ta scale một resource xuống còn 0 replica**.

Bây giờ ta sẽ tạo Job để nó push message vào queue, sau đó ta sẽ kiểm tra coi consumer của ta có scale như ta đã nói hay không.

```
kubectl apply -f publisher-job.yaml
```

```publisher-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: rabbitmq-publish
spec:
  backoffLimit: 4
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: rabbitmq-client
        image: 080196/k8s-practice-keda-send
        env:
          - name: QUEUE_LENGTH
            value: "50"
        envFrom:
          - configMapRef:
              name: share-consume-queue
```

Ở trên ta send 50 message vào queue. Sau đó bạn get pod lại, lúc này ta sẽ thấy số lượng pod của ta đã tăng lên.

```
$ kubectl get deploy

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
rabbitmq              1/1     1            1           47m
share-consume-queue   4/4     4            4           46m
```

```
$ kubectl get pod

NAME                                   READY   STATUS      RESTARTS   AGE
rabbitmq-568f45bff9-bbxkx              1/1     Running     0          47m
rabbitmq-publish-2stmd                 0/1     Completed   0          42s
share-consume-queue-68774b665c-7qg4f   1/1     Running     0          17s
share-consume-queue-68774b665c-gvmtn   1/1     Running     0          18s
share-consume-queue-68774b665c-khrxf   1/1     Running     0          17s
share-consume-queue-68774b665c-mc5tv   1/1     Running     0          17s
```

Kiểm tra resource HPA.

```
$ kubectl get hpa

NAME                           REFERENCE                        TARGETS      MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-share-consume-queue   Deployment/share-consume-queue   50/5 (avg)   1         100       1          28m
```

Lúc này thì pod của ta mới scale lên 4 pod, đợi một lúc bạn get lại Deployment thì sẽ thấy nó đã scale lên 10 replica.

```
$ kubectl get deploy

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
rabbitmq              1/1     1            1           49m
share-consume-queue   10/10   10           10          47m
```

Đợi một lát khi consumer tiêu thụ hết message trong queue, lúc này pod của ta sẽ scale xuống lại.

```
$ kubectl get deploy

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
rabbitmq              1/1     1            1           52m
share-consume-queue   0/0     0            0           50m
```

```
$ kubectl get hpa

NAME                           REFERENCE                        TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-share-consume-queue   Deployment/share-consume-queue   0/5 (avg)   1         100       10         32m
```

Oke, ta đã thực hiện thành công 😁.

## Kết luận
Vậy ta là ta đã tìm hiểu xong cơ bản về Kubernetes based Event Driven Autoscaler, như bạn thấy nó khá đơn giản mà rất hữu dụng. Nếu có thắc mắc hoặc cần giải thích rõ thêm chỗ nào thì các bạn có thể hỏi dưới phần comment.
