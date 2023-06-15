## Giới thiệu
Chào mọi người đến với series về kubernetes của mình, trong series này mình sẽ chia sẻ cho các bạn những kinh nghiệm của mình khi triển khai các ứng dụng thực tế trên môi trường kubernetes. Ở phần này mình sẽ nói về cách triển khai nodejs microservice trên môi trường kubernetes, phần một sẽ nói về cách viết file cấu hình từng thành phần cần thiết của microservice, phần hai sẽ nói về cách làm sao để sử dụng gitlab-ci để triển khai ứng dụng một cách tự động.

Về hệ thống, phần backend mình sẽ sử dụng [nodejs molecular](https://moleculer.services/docs/0.14/) framework, đây là một framework dùng để xây dựng một ứng dụng với kiến trúc microservice. 

## Kiến trúc của ứng dụng
![alt](https://github.com/hoalongnatsu/k8s-practice/blob/main/microservices/microservices.png?raw=true)

API Gateway, sẽ đóng vai trò là ngõ vào cho ứng dụng của ta, nó sẽ tạo một http endpoint và lắng nghe request từ client.

NATS, sẽ đóng vai trò là một transporter, nó sẽ như là một trạm chuyển tiếp để cho các service có thể giao tiếp được với nhau.

Categories service, News service, đây là service thực hiện công việc CRUD resource liên quan.

Redis, dùng để cache kết quả lấy ra từ database và kết quả trả về cho client, giúp giảm số lần thực hiện query vào DB và tăng tốc độ của ứng dụng.

Database thì ta sẽ xài postgres.

Mình đã nói sơ qua về kiến trúc mà ta sẽ triển khai lên trên kubernetes, bây giờ ta sẽ bắt đầu tiến hành viết config cho từng thành phần riêng biệt, bước đầu tiên là ta sẽ build image cho backend của chúng ta.

## Build image
Các bạn có thể sử dụng image mình đã build sẵn, tên là **080196/microservice**, hoặc nếu các bạn thích tự build image cho chính mình, thì các bạn tải source code từ đây xuống https://github.com/hoalongnatsu/microservices.

Sau khi tải được source code, các bạn nhảy vào thư mục microservices/code, và thực hiện build image với tên image sẽ là `<docker-hub-username>/microservice`, sau đó các bạn thực hiện câu lệnh docker login và push image lên docker hub của mình.

```bash
git clone https://github.com/hoalongnatsu/microservices.git && cd microservices/code
docker build . -t 080196/microservice
docker push 080196/microservice
```

Sau khi build image và push lên docker hub thành công, ta sẽ tiến hành viết file config cho ứng dụng.

## Deploy API Gateway
Đầu tiên ta sẽ viết file config cho api gateway, để tạo api gateway, ta sẽ dùng Deployment, Deployment là gì các bạn có thể xem [ở đây](https://viblo.asia/p/kubernetes-series-bai-5-deployment-cap-nhat-ung-dung-bang-deployment-RQqKL6q0l7z), tạo file tên là api-gateway-deployment.yaml với config như sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  labels:
    component: api-gateway
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: api-gateway
  template:
    metadata:
      labels:
        component: api-gateway
    spec:
      containers:
        - name: api-gateway
          image: 080196/microservice
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          env:
            - name: NODE_ENV
              value: testing
            - name: SERVICEDIR
              value: dist/services
            - name: SERVICES
              value: api
            - name: PORT
              value: "3000"
            - name: CACHER
              value: redis://redis:6379
```

Trong image **080196/microservice** chúng ta đã build ở trên, sẽ có 3 service ở trong image đó là api, categories, news. Ta chọn service ta cần chạy bằng cách truyền giá trị của service ta muốn chạy vào biến môi trường có tên là SERVICES, ở file config trên ta chạy api gateway, nên ta truyền vào giá trị là **api**.

Các bạn nhìn vào phần code ở file code/services/api.service.ts, ta sẽ thấy ở chỗ setting cho api gateway ở dòng 15

```
...
settings: {
  port: process.env.PORT || 3001,
...
```

Với biến môi trường PORT, ta sẽ chọn port mà api gateway lắng nghe, ở trên ta truyền vào giá trị là 3000. Vậy api gateway của ta sẽ chạy và lắng nghe ở port 3000. Biến môi trường CACHER là ta khai báo redis host mà các service của chúng ta sẽ sử dụng. Ta tạo deployment.

```
$ kubectl apply -f api-gateway-deployment.yaml
deployment.apps/api-gateway created

$ kubectl get deploy
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
api-gateway   0/1     1            0           100s
```

Ta đã tạo được api gateway, nhưng khi bạn get pod, bạn sẽ thấy nó không chạy thành công, mà sẽ bị restart đi restart lại.

```
$ kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE
api-gateway-79688cf6f5-g88f2   0/1     Running   2          93s
```

Ta logs pod để xem lý do tại sao nó không chạy được mà cứ bị restart.

```
$ kubectl logs api-gateway-79688cf6f5-g88f2
...
[2021-11-07T14:53:37.449Z] ERROR api-gateway-79688cf6f5-g88f2-28/CACHER: Error: getaddrinfo EAI_AGAIN redis
    at GetAddrInfoReqWrap.onlookup [as oncomplete] (dns.js:60:26) {
  errno: 'EAI_AGAIN',
  code: 'EAI_AGAIN',
  syscall: 'getaddrinfo',
  hostname: 'redis'
}
```

Lỗi được hiển thị ở đây là do ta không thể kết nối được tới redis, vì ta chưa tạo redis nào cả, tiếp theo ta sẽ tạo redis.

## Deploy Redis
Ta tạo một file redis-deployment.yaml với config như sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    component: redis
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      component: redis
  template:
    metadata:
      labels:
        component: redis
    spec:
      containers:
        - name: redis
          image: redis
          ports:
            - containerPort: 6379
```

```
$ kubectl apply -f redis-deployment.yaml
deployment.apps/redis created

$ kubectl get deploy
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
api-gateway   0/1     1            0           16m
redis         1/1     1            1           14s
```

Vậy là ta đã tạo được redis pod, tiếp theo, nếu muốn connect được tới redis pod này, ta cần phải tạo Service resource cho nó. Tạo file tên là redis-service.yaml với config như sau:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    component: redis
spec:
  selector:
    component: redis
  ports:
    - port: 6379
```

Ta tạo service:

```
$ kubectl apply -f redis-service.yaml
service/redis created
```

Ta Restart api gateway deployment lại để nó có thể kết nối được tới redis.

```
$ kubectl rollout restart deploy api-gateway
deployment.apps/api-gateway restarted
```

Bây giờ khi ta get pod ra, ta vẫn thấy nó vẫn chưa chạy được thành công, ta logs nó tiếp để xem tại sao.

```
$ kubectl get pod
NAME                           READY   STATUS             RESTARTS   AGE
api-gateway-79688cf6f5-g88f2   0/1     CrashLoopBackOff   8          13m
api-gateway-7f4d5f54f-lzgkd    0/1     Running            2          82s
redis-58c4799ccc-qhv2z         1/1     Running            0          5m41s

$ kubectl logs api-gateway-7f4d5f54f-lzgkd
...
[2021-11-07T15:05:10.388Z] INFO  api-gateway-7f4d5f54f-lzgkd-28/CACHER: Redis cacher connected.

Sequelize CLI [Node: 12.13.0, CLI: 6.2.0, ORM: 6.6.5]

Loaded configuration file "migrate/config.js".
Using environment "testing".

ERROR: connect ECONNREFUSED 127.0.0.1:5432

Error: Command failed: sequelize-cli db:migrate
ERROR: connect ECONNREFUSED 127.0.0.1:5432
    at ChildProcess.exithandler (child_process.js:295:12)
    at ChildProcess.emit (events.js:210:5)
    at maybeClose (internal/child_process.js:1021:16)
    at Process.ChildProcess._handle.onexit (internal/child_process.js:283:5) {
  killed: false,
  code: 1,
  signal: null,
  cmd: 'sequelize-cli db:migrate'
} 
Sequelize CLI [Node: 12.13.0, CLI: 6.2.0, ORM: 6.6.5]

Loaded configuration file "migrate/config.js".
Using environment "testing".

 ERROR: connect ECONNREFUSED 127.0.0.1:5432
...
```

Kết quả in ra ta thấy được là ta đã kết nối redis thành công, lỗi tiếp theo mà pod api gateway hiển thị là lỗi khi nó migrate database, nó không thể kết nối database được, vì ta chưa tạo database nào cả, bước tiếp theo là ta sẽ tạo database.

## Deploy database
Để deploy database, ta sẽ không dùng Deployment mà sẽ dùng StatefulSet, về lý do thì các bạn có thể xem [ở đây](https://viblo.asia/p/kubernetes-series-bai-9-statefulsets-deploying-replicated-stateful-applications-07LKXkXp5V4). Ta tạo một file tên là postgres-statefulset.yaml với config như sau:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  labels:
    component: postgres
spec:
  selector:
    matchLabels:
      component: postgres
  serviceName: postgres
  template:
    metadata:
      labels:
        component: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:11
          ports:
            - containerPort: 5432
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-data
          env:
            - name: POSTGRES_DB
              value: postgres
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              value: postgres
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: hostpath
        resources:
          requests:
            storage: 5Gi
```

Lưu ý phần storageClassName, này tùy thuộc vào kubernetes cluster của bạn thì bạn sẽ chỉ định trường storageClassName tương ứng nhé, để xem StorageClass thì bạn gõ câu lệnh sau `kubectl get sc`. Ta tạo statefulset:

```
$ kubectl apply -f postgres-statefulset.yaml
statefulset.apps/postgres created
$ kubectl get pod
NAME                           READY   STATUS             RESTARTS   AGE
api-gateway-79688cf6f5-g88f2   0/1     Running            16         32m
api-gateway-7f4d5f54f-lzgkd    0/1     CrashLoopBackOff   11         20m
postgres-0                     1/1     Running            0          55s
redis-58c4799ccc-qhv2z         1/1     Running            0          25m
```

Ta get pod thì ta sẽ thấy postgres pod đã được tạo thành công, tiếp theo muốn kết nối được tới DB, ta cần tạo Service cho nó, tạo file tên là postgres-service.yaml với config như sau:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    component: postgres
spec:
  selector:
    component: postgres
  ports:
    - port: 5432
```

Ta tạo service:

```
$ kubectl apply -f postgres-service.yaml
service/postgres created
```

Giờ ta sẽ update lại api gateway Deployment để kết nối với DB ta mới tạo, các bạn xem ở trong file code/src/db/connect.ts thì sẽ thấy các biến môi trường mà api gateway dùng để kết nối tới DB.

```javascript
import { Op, Sequelize } from "sequelize";

export const connect = () => new Sequelize(
		process.env.DB_NAME,
		process.env.DB_USER,
		process.env.DB_PASSWORD,
		{
			host: process.env.DB_HOST,
			port: +process.env.DB_PORT,
			operatorsAliases: {
				$like: Op.like,
				$nlike: Op.notLike,
				$eq: Op.eq,
				$ne: Op.ne,
				$in: Op.in,
				$nin: Op.notIn,
				$gt: Op.gt,
				$gte: Op.gte,
				$lt: Op.lt,
				$lte: Op.lte,
				$bet: Op.between,
				$contains: Op.contains,
			},
			dialect: "postgres",
			logging: (process.env.ENABLE_LOG_QUERY === "true"),
		}
	);
```

Ta update lại các biến env của file api-gateway-deployment.yaml, và tạo lại deployment.

```yaml
 ...
  env:
    - name: NODE_ENV
      value: testing
    - name: SERVICEDIR
      value: dist/services
    - name: SERVICES
      value: api
    - name: PORT
      value: "3000"
    - name: CACHER
      value: redis://redis:6379
    - name: DB_HOST
      value: postgres
    - name: DB_PORT
      value: "5432"
    - name: DB_NAME
      value: postgres
    - name: DB_USER
      value: postgres
    - name: DB_PASSWORD
      value: postgres
```

```
$ kubectl apply -f api-gateway-deployment.yaml
deployment.apps/api-gateway configured

$ kubectl get pod
NAME                         READY   STATUS    RESTARTS   AGE
api-gateway-544c7f84-6hv7z   1/1     Running   0          2m4s
nats-65687968fc-2drwp        1/1     Running   0          4m23s
postgres-0                   1/1     Running   0          31m
redis-58c4799ccc-qhv2z       1/1     Running   0          56m
```

Bây giờ khi ta get pod, thì ta thấy pod api gateway của ta đã chạy thành công.

## Deploy categories news service
Tiếp theo ta sẽ deploy 2 service còn lại của micoservice, tạo file tên là categories-news-deployment.yaml với config như sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: categories-service
  labels:
    component: categories-service
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: categories-service
  template:
    metadata:
      labels:
        component: categories-service
    spec:
      containers:
        - name: categories-service
          image: 080196/microservice
          env:
            - name: NODE_ENV
              value: testing
            - name: SERVICEDIR
              value: dist/services
            - name: SERVICES
              value: categories
            - name: CACHER
              value: redis://redis:6379
            - name: DB_HOST
              value: postgres
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: postgres
            - name: DB_USER
              value: postgres
            - name: DB_PASSWORD
              value: postgres

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: news-service
  labels:
    component: news-service
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: news-service
  template:
    metadata:
      labels:
        component: news-service
    spec:
      containers:
        - name: news-service
          image: 080196/microservice
          env:
            - name: NODE_ENV
              value: testing
            - name: SERVICEDIR
              value: dist/services
            - name: SERVICES
              value: news
            - name: CACHER
              value: redis://redis:6379
            - name: DB_HOST
              value: postgres
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: postgres
            - name: DB_USER
              value: postgres
            - name: DB_PASSWORD
              value: postgres

```

```
$ kubectl apply -f categories-news-deployment.yaml
deployment.apps/categories-service created
deployment.apps/news-service created
```

Sau khi tạo 2 service này xong, để chúng và api gateway có thể giao tiếp với nhau, ta cần tạo NATS.

## Deploy NATS
Tạo một file tên là nats-deployment.yaml với config như sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nats
  labels:
    component: nats
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      component: nats
  template:
    metadata:
      labels:
        component: nats
    spec:
      containers:
        - name: nats
          image: nats
          ports:
            - containerPort: 4222
```

```yaml
$ kubectl apply -f nats-deployment.yaml
deployment/nats created
```

Tiếp theo ta tạo service cho NATS, tạo file nats-service.yaml:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nats
  labels:
    component: nats
spec:
  selector:
    component: nats
  ports:
    - port: 4222
```

```
$ kubectl apply -f nats-service.yaml
service/nats created
```

Bước cuối cùng, ta update lại env của api gateway và categories với news service, và tạo lại chúng. Thêm giá trị này vào env để cập nhật TRANSPORTER cho các service:

```yaml
...
env:
...
    - name: TRANSPORTER
      value: nats://nats:4222
```

Cập nhật lại các deployment.

```
$ kubectl apply -f kubectl apply -f api-gateway-deployment.yaml
deployment.apps/api-gateway configured

$ kubectl apply -f categories-news-deployment.yaml
deployment.apps/categories-service configured
deployment.apps/news-service configured
```

Ta get pod và xem logs của api gateway, xem các service đã có thể giao tiếp với nhau được chưa:

```
$ kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
api-gateway-6cb4c6b657-tlkzq          1/1     Running   0          36s
categories-service-689cdb6c6d-gqtlb   1/1     Running   0          20s
nats-65687968fc-2drwp                 1/1     Running   0          20m
news-service-6b85f99987-dcplv         1/1     Running   0          20s
postgres-0                            1/1     Running   0          48m
redis-58c4799ccc-qhv2z                1/1     Running   0          72m

$ kubectl logs api-gateway-6cb4c6b657-tlkzq
...
[2021-11-07T16:14:54.181Z] INFO  api-gateway-6cb4c6b657-tlkzq-28/REGISTRY: Node 'news-service-6b85f99987-vcjzn-28' connected.
...
[2021-11-07T16:14:57.357Z] INFO  api-gateway-6cb4c6b657-tlkzq-28/REGISTRY: Node 'categories-service-689cdb6c6d-gjjjr-29' connected.
...
```

Bạn sẽ thấy logs là news-service và categories-service đã được kết nối với api-gateway. Vậy là ứng dụng của ta đã chạy thành công, nhưng bạn có để ý thấy những biến env mà ta khai báo thì hơi dài và lập lại ở các file deployment không? Ta có thể dùng ConfigMap để khai báo cấu hình ở chỗ và sử dụng lại cho nhiều nơi, giúp file config của ta gọn hơn.

## Khai báo config chung
Tạo một file tên là microservice-cm.yaml với config như sau:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: microservice-cm
  labels:
    component: microservice-cm
data:
  NODE_ENV: testing
  SERVICEDIR: dist/services
  TRANSPORTER: nats://nats:4222
  CACHER: redis://redis:6379
  DB_NAME: postgres
  DB_HOST: postgres
  DB_USER: postgres
  DB_PASSWORD: postgres
  DB_PORT: "5432"
```

```
$ kubectl apply -f microservice-cm.yaml
configmap/microservice-cm created
```

Ta update lại config của các file deployment như sau, file api-gateway-deployment.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  labels:
    component: api-gateway
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: api-gateway
  template:
    metadata:
      labels:
        component: api-gateway
    spec:
      containers:
        - name: api-gateway
          image: 080196/microservice
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          env:
            - name: SERVICES
              value: api
            - name: PORT
              value: "3000"
          envFrom:
            - configMapRef:
                name: microservice-cm
```

```
$ kubectl apply -f api-gateway-deployment.yaml
deployment.apps/api-gateway configured
```

File categories-news-deployment.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: categories-service
  labels:
    component: categories-service
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: categories-service
  template:
    metadata:
      labels:
        component: categories-service
    spec:
      containers:
        - name: categories-service
          image: 080196/microservice
          env:
            - name: SERVICES
              value: categories
          envFrom:
            - configMapRef:
                name: microservice-cm

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: news-service
  labels:
    component: news-service
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: news-service
  template:
    metadata:
      labels:
        component: news-service
    spec:
      containers:
        - name: news-service
          image: 080196/microservice
          env:
            - name: SERVICES
              value: news
          envFrom:
            - configMapRef:
                name: microservice-cm
```

```
$ kubectl apply -f categories-news-deployment.yaml
deployment.apps/categories-service configured
deployment.apps/news-service configured
```

Ta get pod lại xem mọi thứ vẫn ok không:

```
$ kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
api-gateway-86b67895fd-cphmv          1/1     Running   0          79s
categories-service-84c74cd87c-zjtd2   1/1     Running   0          53s
nats-65687968fc-2drwp                 1/1     Running   0          41m
news-service-69f45b8668-kv9dm         1/1     Running   0          52s
postgres-0                            1/1     Running   0          69m
redis-58c4799ccc-qhv2z                1/1     Running   0          93m
```

OK, tất cả các thành phần của ứng dụng ta vẫn chạy bình thường. Ta sử dụng ConfigMap để giúp file config của ta nhìn gọn hơn, và ta quản lý tất cả env ở một chỗ, dễ dàng thay đổi và cập nhật lại.

## Kết luận
Vậy là ta đã triển khai được mô hình microservice lên môi trường kubernetes, như các bạn thấy thì không khó lắm, chỉ cần ta triển khai những  thành phần trong đó từ từ, là ta sẽ triển khai được. Phần tiếp theo mình sẽ nói về cách làm sao để tự động được quá trình khi ta cập nhật lại file config nó sẽ tự update lên trên cluster cho chúng ta, không cần ta phải chạy lại câu lệnh kubectl. Và cách dùng gitlab ci để cập nhật lại ứng dụng khi ta sửa và update code trong ứng dụng của ta. Nếu các bạn có thắc mắc hoặc chưa hiểu rõ chỗ nào, các bạn có thể hỏi ở phần comment