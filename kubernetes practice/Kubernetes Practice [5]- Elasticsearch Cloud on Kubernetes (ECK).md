## Giới thiệu
Chào mọi người đến với series practice về kubernetes. Ở bài này chúng ta sẽ tìm hiểu về cách triển khai Elasticsearch cluster lên trên môi trường cloud, và cách backup và restore Elasticsearch thông qua snapshot của Elasticsearch và S3 bucket.

Để triển khai Elasticsearch thì có rất nhiều cách (ở đây mình bàn về triển khai Elasticsearch trên AWS):
+ Xài AWS Opensearch (Elasticsearch service), thằng này thì AWS nó quản lý scale, HA cho mình, dễ quản lý, nhưng tiền khá cao, hơn $400/month.
+ Deploy Elasticsearch lên trên kubernetes, giá tiền thì chắc chắn sẽ ít hơn khi xài của AWS Opensearch. Dễ scale và quản lý, tuy nhiên yêu cầu cần phải biết kubernetes.
+ Deploy Elasticsearch lên thẳng EC2, này thì sẽ không có HA sẵn, cần mình tự settup, bitnami nó có một ELK AMI sẵn (https://aws.amazon.com/marketplace/pp/prodview-tlbc33skxwrm6), mình chỉ cần tạo EC2 instance từ AMI này. Cách này thì giá tiền là ít nhất, chỉ cần con EC2 t3a.medium với 30GB là mình đủ chạy => khoảng $0.06/hr. Nhưng về HA, scale, backup, recovery thì mình tự làm khá nhiều.

Tùy thuộc vào điều kiện thì ta sẽ chọn cách phù hợp, ở đây mình sẽ hướng dẫn cách triển khai dùng kubernetes.

![image.png](./images/e3241479-2cb2-48fa-befb-3e101294a0a0.png)

## Elastic Cloud on Kubernetes
Bình thường để deploy Elasticsearch, thì ta sẽ viết file config StatefulSet cho Elasticsearch, hoặc dùng helm chart để deploy Elasticsearch lên trên K8S, tuy việc đó không có gì sai cả, nhưng để quản lý scale và cấu hình security này nọ cho cụm Elasticsearch đó thì khá khó khăn. Nên Elasticsearch official có cung cấp cho ta sẵn một Kubernetes Operator dùng cho Elasticsearch, tên là **Elastic Cloud on Kubernetes** viết tắt là ECK, sử dụng thằng này sẽ giúp ta deploy Elasticsearch lên trên K8S một cách rất chi là dễ dàng.

Nếu các bạn chưa biết Kubernetes Operator là gì thì đọc bài này nhé https://viblo.asia/p/kubernetes-series-bai-19-adding-custom-resource-to-kubernetes-eW65GB8alDO.

Để sử dụng ECK, đầu tiên ta phải cài CRDs và RBAC rules của nó.

### Install ECK
Cài CRDs và RBAC rules:

```
kubectl create -f https://download.elastic.co/downloads/eck/1.9.1/crds.yaml
```

Cài operator controller cho ECK:

```
kubectl apply -f https://download.elastic.co/downloads/eck/1.9.1/operator.yaml
```

Kiểm tra operator đã hoạt động chưa:

```
$ kubectl get sts elastic-operator

NAME               READY   AGE
elastic-operator   1/1     99s
```

Chắc chắn ta thấy cột READY là 1/1 thì ECK của ta đã cài thành công nhé.

### Deploy Elasticsearch with ECK
Để triển khai Elasticsearch cluster với ECK, ta sẽ dùng apiVersion là `elasticsearch.k8s.elastic.co/v1` với kind là Elasticsearch. Tạo một file tên là elasticsearch.yaml với config như sau:

```elasticsearch.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
spec:
  version: 7.16.3
  nodeSets:
  - name: default
    count: 1
```

```
kubectl apply -f elasticsearch.yaml
```

Thuộc tính version để ta chọn version của Elasticsearch mà ta muốn dùng, thuộc tính nodeSets dùng để định nghĩa cụm node cho elasticsearch, ở trên ta chỉ định cụm node có tên là default với số lượng node là 1.

Sau khi chạy câu lệnh apply xong, ta kiểm tra thử Elasticsearch của ta chạy chưa.

```
$ kubectl get elasticsearch

NAME            HEALTH   NODES   VERSION   PHASE   AGE
elasticsearch   yellow   1       7.16.3    Ready   4m53s
```

Khi ta tạo một Elasticsearch resource bằng ECK, thì ECK sẽ tự động tạo StatefulSet ở bên dưới cho ta. Ta kiểm tra như sau.

```
$ kubectl get sts

NAME                       READY   AGE
elasticsearch-es-default   1/1     4m53s
```

Oke, như bạn thấy, sử dụng ECK ta deploy một cụm Elasticsearch cực kì dễ, có cái là nó chỉ xài được trên cloud chứ không chạy dưới on premise thôi 😂.

### Access Elasticsearch
Để access cụm elasticsearch, ta có thể dùng port-forward `kubectl port-forward service/elasticsearch-es-http 9200`, sử dụng curl để request tới nó.

```
$ curl -k https://localhost:9200 ; echo

{"error":{"root_cause":[{"type":"security_exception","reason":"missing authentication credentials for REST request [/]","header":{"WWW-Authenticate":["Basic realm=\"security\" charset=\"UTF-8\"","Bearer realm=\"security\"","ApiKey"]}}],"type":"security_exception","reason":"missing authentication credentials for REST request [/]","header":{"WWW-Authenticate":["Basic realm=\"security\" charset=\"UTF-8\"","Bearer realm=\"security\"","ApiKey"]}},"status":401}
```

Ta sẽ thấy nó báo lỗi là 401, vì khi ta tạo Elasticsearch dùng ECK, thì mặc định nó sẽ bật mode security lên cho ta, nên để truy cập tới nó thì ta cần phải có password. Ta lấy password của cụm Elasticsearch ở trong Secret resource bằng câu lệnh.

```
kubectl get secret elasticsearch-es-elastic-user -o jsonpath={.data.elastic} | base64 --decode ; echo
```

Nó sẽ in ra password cho ta. Ok gọi lại elastic mà có truyền thêm password.

```
PASSWORD=$(kubectl get secret elasticsearch-es-elastic-user -n persistent -o jsonpath={.data.elastic} | base64 --decode)
curl -u "elastic:$PASSWORD" https://localhost:9200
```

```
{
  "name" : "elasticsearch-es-default-0",
  "cluster_name" : "default",
  "cluster_uuid" : "XqWg0xIiRmmEBg4NMhnYPg",
  "version" : {...},
  "tagline" : "You Know, for Search"
}
```

Yep, vậy là ta đã triển khai Elasticsearch lên trên K8S thành công.

### Custom persistent volume
Mặc định ECK sẽ tạo ra PVC với size là 1GB cho mỗi pod, nếu ta muốn thay đổi số đó thì ta phải chỉ định thông qua trường **volumeClaimTemplates**, bạn có thể kiểm tra thông qua câu lệnh kubectl.

```
$ kubectl get pvc
NAME                                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-data-elasticsearch-es-default-0   Bound                                               1Gi        RWO            gp2            4m53s
```

Cập nhật lại PVC của file elasticsearch.yaml

```elasticsearch.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
spec:
  version: 7.16.3
  nodeSets:
  - name: default
    count: 1
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # Do not change this name unless you set up a volume mount for the data path.
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
```

```
kubectl apply -f elasticsearch.yaml
```

Kiểm tra PVC.

```
$ kubectl get pvc
NAME                                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-data-elasticsearch-es-default-0   Bound                                               10Gi       RWO            gp2            1m52s
```

Lúc này bạn sẽ thấy storage của ta đã tăng lên 10GB.

### Public Elasticsearch
Để public elasticsearch cho dev bên ngoài có thể truy cập được, ta sẽ dùng Ingress loại AWS Load Balancer. Để cài Ingress, các bạn xem hướng dẫn sau đây https://aws.amazon.com/premiumsupport/knowledge-center/eks-alb-ingress-controller-fargate/.

ECK sẽ tạo ra cho elasticsearch cluster của ta 3 service.

```
$ kubectl get svc
NAME                         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
elasticsearch-es-default     ClusterIP   None           <none>        9200/TCP   7m52s
elasticsearch-es-http        ClusterIP   172.20.86.41   <none>        9200/TCP   7m52s
elasticsearch-es-transport   ClusterIP   None           <none>        9300/TCP   7m52s
```

Service mà expose request cho ta là elasticsearch-es-http, ta sẽ tạo một ingress chỉa tới thằng này. Tạo một file tên là ingress-elasticsearch.yaml với config sau:

```ingress-elasticsearch.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: elasticsearch
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTPS
    alb.ingress.kubernetes.io/success-codes: '401'
    alb.ingress.kubernetes.io/backend-protocol: HTTPS
    alb.ingress.kubernetes.io/certificate-arn: <acm-arn>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
spec:
  rules:
  - host: elastic.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ssl-redirect
            port:
              name: use-annotation
      - path: /
        pathType: Prefix
        backend:
          service:
            name: elasticsearch-es-http
            port:
              number: 9200
```

+ `alb.ingress.kubernetes.io/healthcheck-protocol: HTTPS` sẽ cấu hình phương thức AWS Load Balancer dùng để healthcheck là HTTPS.
+ `alb.ingress.kubernetes.io/success-codes: '401'` vì khi ta dùng ECK để deploy elasticsearch, mặc định nó sẽ bật security mode lên, nên tất cả các đường dẫn đều cần authenticate mới gọi được, vì hiện tại ALB không cho cấu hình truyền body vào trong healthcheck, nên ta phải thay đổi success code thành 401.
+ `alb.ingress.kubernetes.io/backend-protocol: HTTPS` protocol ALB gọi tới Pod sẽ là HTTPS.
+ `alb.ingress.kubernetes.io/certificate-arn` là ACM ARN, nhớ nhập chính xác nhé, ở trên chỉ là giá trị ví dụ.
+ `alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'` port ALB listen là 80 và 443.

Truy cập elastic thông qua url `elastic.example.com`.

## Backup and Restore
Ở đây để thực hiện backup và restore, ta sẽ sử dụng cơ chế **Snapshot and Restore** của elasticsearch. Elasticsearch cho phép ta tạo **snapshot** từ dữ liệu hiện tại, và có thể restore dữ liệu từ snapshot đó, snapshot này ta có thể tự tạo bằng tay hoặc cấu hình cho elasticsearch tự động tạo snapshot trong một khoảng thời gian nhất định. Snapshot này ta có thể lưu ở local server, hoặc ở trên cloud đều được.

### Backup
Vì ta triển khai ECK ở trên AWS, nên ta sẽ lưu snapshot ở trên S3. Để thực hiện tạo snapshot và lưu nó trên S3, ta cần làm các bước sau đây:
+ Cài elasticsearch plugin cho S3.
+ Thêm credentials vào elasticsearch để nó quyền lưu dữ liệu vào s3.
+ Đăng ký snapshot repository bằng Elasticsearch API. 
+ Tạo snapshot bằng tay hoặc cấu hình cho elasticsearch tự động tạo snapshot.

### Install S3 plugin
Để install plugin bằng ECK khá đơn giản, ta chỉ cần thêm vào file cấu hình elasticsearch một **initContainers** và cho nó thực hiện câu lệnh install plugin là được. Cập nhật lại file elasticsearch.yaml như sau:

```elasticsearch.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
spec:
  version: 7.16.3
  nodeSets:
  - name: default
    count: 1
    podTemplate:
      spec:
        initContainers:
        - name: install-plugins
          command:
          - sh
          - -c
          - |
            bin/elasticsearch-plugin install --batch repository-s3
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
```

Đây là đoạn ta dùng để install plugin.

```yaml
initContainers:
- name: install-plugins
  command:
  - sh
  - -c
  - |
    bin/elasticsearch-plugin install --batch repository-s3
```

### Add S3 Credentials into Elasticsearch
Để thêm credentials vào elasticsearch, đầu tiên ta cần truy cập vào IAM console, tạo một user và cấp quyền cho nó  có toàn quyền trên S3, sau đó lấy access_key và secret_key của user đó và import vào elasticsearch. Ta làm như sau:
+ Truy cập IAM console https://console.aws.amazon.com/iam.
+ Bấm vào menu Users chọn Add users.
+ Nhập User name và ở mục **Select AWS credential type** check vào **Access key - Programmatic access**, bấm Next Permission.
+ Chọn Attach existing policies directly, kiếm AmazonS3FullAccess và chọn nó, bấm Next Tags và bấm Next Review, sau đó tạo user, nhớ tải xuống hoặc copy lại **ACCESS_KEY** và **SECRET_KEY**.

Sau đó, tạo hai file với với tên là `s3.client.default.access_key` và `s3.client.default.secret_key`, copy **ACCESS_KEY** dán vào file s3.client.default.access_key và **SECRET_KEY** dán vào file s3.client.default.secret_key, chạy câu lệnh sau để tạo K8S Secret resource và gán nó vào elasticsearch.

```
kubectl create secret generic s3-credentials --from-file=s3.client.default.access_key --from-file=s3.client.default.secret_key
```

Cập nhật lại file elasticsearch.yaml, thêm vào đoạn cấu hình sau:

```
...
spec:
  version: 7.16.3
  secureSettings:
  - secretName: s3-credentials
    entries:
    - key: s3.client.default.access_key
    - key: s3.client.default.secret_key
...
```

```
kubectl apply -f elasticsearch.yaml
```

Oke, vậy là ta đã thêm credentials thành công.

### Register the snapshot repository
Tiếp theo là ta sẽ tạo một repository để chứa toàn bộ snapshot. Kiểm tra toàn bộ repository hiện tại.

```
$ curl -u "elastic:$PASSWORD" https://localhost:9200/_snapshot/_all

{}
```

Tạo repository.

```
$ curl -XPUT -u "elastic:$PASSWORD" https://localhost:9200/_snapshot/s3_snapshot -d '{
  "type": "s3",
  "settings": {
    "client": "default",
    "bucket": "<your-bucket>",
    "region": "<region>"
  }
}'

{
  "acknowledged": true
}
```

Lúc này khi liệt kệ repository ta sẽ thấy repository ta vừa tạo.

```
$ curl -u "elastic:$PASSWORD" https://localhost:9200/_snapshot/_all

{
  "s3_snapshot" : {
    "type" : "s3",
    "settings" : {
      "bucket" : "<your-bucket>",
      "region" : "<region>",
      "client" : "default"
    }
  }
}
```

Tạo snapshot bằng tay.

```
curl -XPUT -u "elastic:$PASSWORD" https://localhost:9200/_snapshot/s3_snapshot/<snapshot-name>
```

Với snapshot-name là tên snapshot của ta. Liệt kê toàn bộ snapshot.

```
curl -u "elastic:$PASSWORD" https://localhost:9200/_snapshot/s3_snapshot/_all
```

### Restore
Nếu ta có lỡ xóa đi dữ liệu, hoặc server ta bị disaster, để restore dữ liệu từ snapshot thì khá đơn giản, ta chỉ việc gọi API như sau:

```
curl -XPOST -u "elastic:$PASSWORD" https://localhost:9200/_snapshot/s3_snapshot/<snapshot-name>/_restore
```

Elastic sẽ lấy lại dữ liệu của ta từ snapshot.

### Automated snapshots
Để tự động tạo snapshot, ta cần sử dụng Kibana. Sau đó truy cập vào Kibana, di chuyển tới phần Management -> Snapshot and Restore -> Policies, sau đó bấm tạo Policies, cấu hình thời gian tạo snapshot theo ý của ta.

![image.png](./images/076e9edf-b63b-4add-a48d-99ea6c3d8dc6.png)

## Custom self-signed certificate using cert-manager
Phần này cũng khá quan trọng nếu bạn xài JAVA hoặc .NET core, vì khi ta dùng ECK thì nó sẽ tự tạo self-signed certificate để config TLS cho elastic cluster, nên khi ở trên K8S lúc ta gọi tới elastic bằng phương thức HTTPS thì ta cần phải cung cấp thêm file CA thì nó mới gọi được. Nếu không thì nó sẽ báo lỗi, ví dụ lỗi ở JAVA khi ta gọi tới elasticsearch.

```
PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
```

Để fix lỗi này, ta sẽ tự tạo self-signed certificate rồi gắn nó vào elasticsearch, sau đó gắn tiếp vào Pod nào mà cần connect tới cụm elasticsearch này. Ta sẽ dùng cert-manager để tạo certificate. Trước tiên ta sẽ cần cài cert-manager trước.

Cài cert-manager.

```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
```

Sau khi cài xong, tạo một file tên là elasticsearch-cert.yaml với config như sau:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: elasticsearch-es-cert
spec:
  isCA: true
  dnsNames:
    - elasticsearch-es-http
    - elasticsearch-es-http.default.svc
    - elasticsearch-es-http.default.svc.cluster.local
  issuerRef:
    kind: Issuer
    name: selfsigned-issuer
  secretName: elasticsearch-es-cert
  subject:
    organizations:
      - elasticsearch
```

```
kubectl apply -f elasticsearch-cert.yaml
```

Certificate được tạo ra sẽ được lưu ở Secret `elasticsearch-es-cert`. Ta gắn nó vào elasticsearch.

```yaml
...
spec:
  version: 7.16.3
  http:
    tls:
      certificate:
        secretName: elasticsearch-es-cert
  secureSettings:
  - secretName: s3-credentials
    entries:
    - key: s3.client.default.access_key
    - key: s3.client.default.secret_key
...
```

Giờ elasticsearch sẽ sử dụng certificate ta vừa tạo ra. Với Pod mà ta cần connect tới Elasticsearch trên, thì ta sẽ mount Secret elasticsearch-es-cert vào Pod. Ví dụ Pod chạy JAVA container, ta sẽ làm như sau.

```yaml
...
spec:
  containers:
    - name: <name>
      image: <image>
      lifecycle:
        postStart:
          exec:
            command:
            - sh
            - -c
            - |
              keytool -import -noprompt -trustcacerts -alias elasticsearch-es-http \
                -file /tmp/cert/ca.crt \
                -keystore /usr/lib/jvm/jre-11/lib/security/cacerts \
                -keypass changeit -storepass changeit
      volumeMounts:
        - name: cert
          mountPath: /tmp/cert
  volumes:
    - name: cert
      secret:
        secretName: elasticsearch-es-cert
...
```

Ta mount secret vào thư mục /tmp/cert của container, sau đó ta dùng câu lệnh keytool để import certificate vào file cacerts trong container. JAVA sẽ dùng file cacerts này để quản lý các CA.

## Kết luận
Vậy là ta đã tìm hiểu xong cách triển khai Elasticsearch lên trên K8S bằng cách sử dụng ECK, như các bạn thấy khi ta dùng ECK ta deploy Elasticsearch rất dễ dàng, các bạn xem full docs ở đây https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html, bạn sẽ thấy hướng dẫn của phần autoscale Elasticsearch cluster cũng khá dễ dàng. Nếu có thắc mắc hoặc cần giải thích rõ thêm chỗ nào thì các bạn có thể hỏi dưới phần comment.
