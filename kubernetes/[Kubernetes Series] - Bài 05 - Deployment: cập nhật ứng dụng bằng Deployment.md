## Giới thiệu
Chào các bạn tới với series về kubernetes. Đây là bài thứ 5 trong series của mình, ở các bài trước ta đã nói về cách đóng gói application bằng container, nhóm chúng lại bằng Pod, rồi dùng ReplicaSetđể chạy Pod để application của chúng ta high availability nhất có thể, dùng Service  để mở kết nối của Pod ra bên ngoài. Ở bài này chúng ta sẽ nói về một resource giúp chúng ta trong quá trình CD, cập nhật một phiên bản mới của ứng dụng một cách dể dàng mà không gây ra ảnh hưởng nhiều tới client, là **Deployment**.

Trước khi nói về Deploymnet resource, ta sẽ xem qua nếu ta update một ứng dụng mà dùng những resource khác sẽ gặp những hạn chế gì.

## Cập nhật một ứng dụng đang chạy trong pods
Bắt đầu với một ví dụ là ta đang có một ứng dụng đang chạy, ta deploy nó bằng ReplicaSet, với replicas = 3, nó sẽ chạy 3 Pod đằng sau, và ta deploy một Service để expose traffic của nó ra cho client bên ngoài.

![image.png](./images/c6f50839-84e7-4a4d-bd3d-5b052f0d0caa.png)

Bây giờ các dev trong team đã viết xong tính năng mới, ta build lại image với code mới, và ta muốn update lại các Pod đang chạy này với image mới. Ta sẽ làm như thế nào? Thì trong quá trình deploy, sẽ có rất nhiều chiến lược để làm, nhưng có 2 cách thông dụng nhất là: **Recreate and RollingUpdate**

### Recreate
Ở cách deploy này, đầu tiên là sẽ xóa toàn bộ phiên bản (version) cũ của ứng dụng trước, sau đó ta sẽ deploy một version mới lên. Đối với kubernes thì đầu tiên ta sẽ cập nhật Pod template của ReplicaSet, sau đó ta xóa toàn bộ Pod hiện tại, để ReplicaSet tạo ra Pod với image mới.

![image.png](./images/2c76948f-3911-49d4-aa51-b1a7aa02eec1.png)

Với cách deploy này thì quá trình deploy quá dễ dàng, nhưng ta sẽ gặp một vấn đề rất lớn, đó là ứng dụng của chúng ta sẽ downtime với client, client không thể request tới ứng dụng của ta trong quá trình version mới được deploy lên.

Với những hệ thống ít client thì dù bạn downtime 1 phút 2 phút hoặc 1 tiếng cũng không ảnh hưởng nhiều lắm, nhưng với những hệ thống với số lượng request lớn tầm 1000-3000 request trên giây, đặt biệt là với hệ thống ngân hàng thì quá trình downtime dù chỉ 1s thì cũng không được. Nên mới sinh ra cách deploy thứ hai là RollingUpdate

### RollingUpdate
Ở cách này, ta sẽ deploy từng version mới của ứng dụng lên, chắc chắn rằng nó đã chạy, ta dẫn request tới version mới của ứng dụng này, lặp lại quá trình này cho tới khi toàn bộ version mới của ứng dụng được deploy và version cũ đã bị xóa. Đối với kubernetes, ta sẽ lần lượt xóa từng Pod và ReplicaSet sẽ tạo Pod mới cho ta.

![image.png](./images/6b723bd1-615e-4b65-93a3-4ebf9a3eac02.png)

Thì ở cách deploy này, điểm mạnh là ta sẽ giảm thời gian downtime của ứng dụng đối với client, còn điểm yếu là ta sẽ có version mới và version cũ của ứng dụng chạy chung một lúc. Và khó nhất là ta phải viết script để thực hiện quá trình này, test script này chạy đúng không, rất mất thời gian và công sức. Với các hệ thống lớn thì để viết một cái script test được quá trình deploy này thì không dễ chút nào.

Để chọn cách deploy nào cho ứng dụng thì còn tùy vào tình huống và nhu cầu, nếu ứng dụng ta có thể chạy nhiều version của ứng dụng cùng lúc không ảnh hưởng thì ta sẽ chọn RollingUpdate. Còn nếu ứng dụng chúng ta downtime không ảnh hưởng gì, thì ta sẽ cách Recreate để deploy. Còn cách một cách nữa là không có downtime và không có nhiều version của ứng dụng chạy cùng một lúc nữa là Bule-Green update. Thì ở đây mình không nói về cách deploy này. Các bạn có thể đọc thêm [ở đây](https://martinfowler.com/bliki/BlueGreenDeployment.html).

Nếu ta chỉ dùng những resource bình thường, thì vấn đề đầu tiên ta gặp phải là quá trình cập nhật version mới của ứng dụng và cần phải script để thực hiện một số thao tác như xóa Pod cũ. **Vấn đề thứ hai ta sẽ gặp là ta phát hiện version mới của chúng ta chạy sai hoặc có bug, ta muốn lùi lại phiên bản trước đó thì làm sao?**

## Lùi lại phiên bản trước của ứng dụng
Thì ở đây chúng ta sẽ có cách là fix bug, build image mới, rồi thực hiện lại cách depoy là Recreate hoặc RollingUpdate đối với những lỗi không cần gấp lắm và không lớn lắm. Còn đối với lỗi bự và ảnh hưởng nhiều tới client thì ta thường sẽ revert lại code trên git, sau đó ta build image, thực hiện deploy lại.

Đối với kubernetes, chúng ta dùng container chạy ứng dụng thì chỉ cần update lại version của image trong Pod template bằng image trước đó, rồi dùng cách Recreate hoặc RollingUpdate để deploy lại Pod là xong, không cần revert code hoặc fixbug ngay lập tức. Dù là cách nào thì cũng sẽ tốn nhiều thời gian để thực hiện và chạy CI/CD lại.

Để giải quyết những vấn đề trên thì kubernetes cung cấp một resource là **Deployment**.

## Deployment là gì?
Deployment là một resource của kubernetes giúp ta trong việc cập cập một version mới của úng dụng một cách dễ dàng, nó cung cấp sẵn 2 strategy để deploy là Recreate và RollingUpdate, tất cả đều được thực hiện tự động bên dưới, và các version được deploy sẽ có một histroy ở đằng sau, ta có thể rollback and rollout giữa các phiên bản bất cứ lúc nào mà không cần chạy lại CI/CD.

Khi ta tạo một Deployment, nó sẽ tạo ra một ReplicaSet bên dưới, và ReplicaSet sẽ tạo Pod. Luồn như sau **Deployment tạo và quản lý ReplicaSet -> ReplicaSet tạo và quản lý Pod -> Pod run container**.

![image.png](./images/218984a8-6c9d-466e-a6ac-53fe662c72d0.png)

Giờ ta tạo thử Deployment nào. File config của Deployment thì đa phần cũng giống như của ReplicaSet, chỉ cần thay đổi kind thành Deployment là được. Tạo một file tên là hello-deploy.yaml:
```yaml
apiVersion: apps/v1
kind: Deployment # change here
metadata:
  name: hello-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
      - image: 080196/hello-app:v1
        name: hello-app
        ports:
          - containerPort: 3000

---
apiVersion: v1
kind: Service
metadata:
  name: hello-app
spec:
  type: NodePort
  selector:
    app: hello-app
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 31000
```

Code của image **080196/hello-app:v1.**
```javascript
const http = require("http");

const server = http.createServer((req, res) => {
  res.end("Hello application v1\n")
});

server.listen(3000, () => {
  console.log("Server listen on port 3000")
})
```

Tạo deployment, khi ta tạo deployment nó sẽ tạo một thằng ReplicaSet ở bên dưới, và thằng ReplicaSet đó sẽ tạo Pod.

`kubectl apply -f hello-deploy.yaml --record`

![image.png](./images/00aea675-d64b-4a6d-aece-301848857542.png)

Ta test thử deployment của chúng ta, ở file trên ta có tạo thêm một thằng Service NodePort với port 31000, ta có thể truy cập ứng dụng với địa chỉ localhost:31000.

![image.png](./images/a40b7716-af41-47be-8f34-95de0563ab3f.png)

Vậy là deployment của chúng ta đã chạy thành công. Bây giờ ta sẽ tiến hành update lại ứng dụng đang chạy trong Pod. Ứng dụng của Pod mới sẽ có image là **080196/hello-app:v2**. Code của image như sau:
```javascript
const http = require("http");

const server = http.createServer((req, res) => {
  res.end("Hello application v2\n") // change v1 to v2
});

server.listen(3000, () => {
  console.log("Server listen on port 3000")
})
```

Để update lại ứng dụng trong Pod với Deployment. Ta chạy câu lệnh sau:

`kubectl set image deployment hello-app hello-app=080196/hello-app:v2`

Cấu trúc câu lệnh `kubectl set image deployment <deployment-name> <container-name>=<new-image>`

Kiểm tra qua trình update đã xong chưa:

`kubectl rollout status deploy hello-app`

![image.png](./images/24f2aca7-cf52-467f-9523-2d15a8460511.png)

Nếu câu lệnh rollout status in ra successfully, là ứng dụng của chúng ta đã cập nhật lên version mới. Ta test bằng cách gửi request tới ứng dụng ở địa chỉ localhost:31000.

![image.png](./images/30a78178-e6e6-4cb0-a44d-33cf1c55994b.png)

Nếu ở đây đã in ra được **Hello application v2** thì ứng dụng của chúng ta đã cập nhật được version mới. Như các bạn thấy thì quá trình deploy một version của ứng dụng cực kì đơn giản nếu chúng ta xài Deployment. Ta cũng có thể chọn deploy strategy ta muốn cực kì đơn giản bằng cách chỉ định trong thuộc tính **strategy** của Deployment.
```yaml
apiVersion: apps/v1
kind: Deployment 
metadata:
  name: hello-app
spec:
  replicas: 3
  strategy: # change here
    type: RollingUpdate # strategy type is Recreate or RollingUpdate. Default is Recreate
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
      - image: 080196/hello-app:v1
        name: hello-app
        ports:
          - containerPort: 3000
```

### Cách Deploymnet update Pod
Thì ở dưới kubernetes, khi ta thay đổi image của Deployment, nó sẽ tạo ra một ReplicaSet khác, và ReplicaSet đó sẽ giữ template Pod mới, và các Pod mới sẽ được tạo ra bởi ReplicaSet.

![image.png](./images/6f5c8dff-4833-4161-9588-ff1ce5a7f9dd.png)

Và ReplicaSet cũ sẽ không bị xóa, mà trong trường này thuộc tính **replicas** của nó sẽ được cập nhật lại là 0. Vì sao ReplicaSet cũ không bị xóa thì nó sẽ có vai trò là ở trong quá trình mà ta update version mới của ứng dụng, và ta phát hiện version mới nó bị lỗi, ta muốn rollback lại version trước, thì ReplicaSet cũ của ta vẫn ở đó, ta sẽ rollback lại dễ dàng hơn.

### Rollback lại version trước khi version mới của ứng dụng bị lỗi
Image của version mới của chúng ta là **080196/hello-app:v3**. Code của image như sau, khi ta gọi request tới nó hơn 3 lần thì nó sẽ trả về lỗi:
```javascript
const http = require("http");

let requestCount = 0;

const server = http.createServer((req, res) => {
  if (++requestCount > 3) {
    res.writeHead(500); // set status code 500
    res.end("Application v3 have some internal error has occurred!\n");
    return;
  }

  res.end("Hello application v3\n");
});

server.listen(3000, () => {
  console.log("Server listen on port 3000")
})
```

Cập nhật lại Deployment

`kubectl set image deployment hello-app hello-app=080196/hello-app:v3`

![image.png](./images/f5abee90-e9ea-4f01-ae82-0d62888f99bd.png)

Test thử, vì ta có tới 3 replicas ở phía sau nên có thể ta gọi nhiều hơn 3 lần nó mới hiển thị lỗi ra, vì mỗi lần ta gửi request nó sẽ dẫn tới Pod khác nhau.

![image.png](./images/9cf09ef5-cea4-4290-9287-12454baac886.png)

Như bạn thấy ở đây là ứng dụng v3 của chúng ta đã bị lỗi, và ta muốn nhanh chóng rollback lại version trước đó là v2, để không ảnh hưởng tới client. Để làm được việc đó thì là dùng câu lệnh kubectl rollout. Trước tiên bạn có thể kiểm tra lịch sử các lần ứng dụng của chúng ta đã cập nhật.

`kubectl rollout history deploy hello-app`

![image.png](./images/559f6ed9-6d34-4975-beb5-b24b4d309b9a.png)

Ta sẽ thấy là có 3 lần chúng ta đã cập nhập ứng dụng. Ta đang ở revision là 3, ta muốn quay lại revision là 2, lúc ứng dụng của ta chưa có lỗi

`kubectl rollout undo deployment hello-app --to-revision=2`

![image.png](./images/6f09e8e5-3833-4981-aba4-627002f4a8fe.png)

Ta test thử

![image.png](./images/b4e560bc-5ffb-46cf-a10b-c3cbd07e2075.png)

Yepp, ứng dụng của ta đã in ra **Hello application v2**, vậy là ta đã rollback về version trước thành công. Giờ ta chỉ việc fixbug và deploy version mới không có bug. Mọi chuyện trở nên dễ dàng hơn, không cần phải revert code trên git hay gì cả, vẫn có thời gian fixbug mà không ảnh hưởng tởi client. Như các bạn thấy đây là điểm mạnh của việc ta sử dụng Deployment để chạy ứng dụng của chúng ta. Đây là minh họa của revision history.

![image.png](./images/2360ad92-7426-4c34-9268-0cf41904a8fa.png)

Về revision thì mặc định nó sẽ lưu là 10, bạn có thể điều chỉnh số lượng history bạn muốn lưu lại bằng thuộc tính **revisionHistoryLimit** của Deployment
```yaml
apiVersion: apps/v1
kind: Deployment 
metadata:
  name: hello-app
spec:
  revisionHistoryLimit: 1
  replicas: 3
  ...
```

Ngoài ra Deployment còn có một điểm mạnh rất tuyệt vời nữa là có thể giúp ta config để deploy ứng dụng zero downtime, tuy ta dùng RollingUpdate nhưng không cũng không chắc được là ứng dụng của chúng ta zero downtime. Về phần config cái này thì mình sẽ nói ở các bài sau. Xóa resource đi nhé:

`k delete -f hello-deploy.yaml `

![image.png](./images/eddc56c6-8097-4fb8-a500-598a7f4eca7d.png)

Dưới đây là config CI/CD dùng Deployment với gitlabCI trong thực tế, các bạn có thể tham khảo thêm:
```yaml
stages:
  - build
  - deploy

image:
  name: docker:stable
  
variables:
  DOCKER_HOST: tcp://docker:2375
  DOCKER_TLS_CERTDIR: ""

services:
  - docker:stable-dind

build root image:
  stage: build
  before_script:
    - echo "$REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u $REGISTRY_USER --password-stdin
  script:
    - docker pull $CI_REGISTRY_IMAGE:latest || true
    - docker build . --cache-from $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main

deploy k8s:
  stage: deploy
  image: bitnami/kubectl
  script:
    - |
      kubectl -n testing set image deployment microservice microservice=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  when: manual
  only:
    - main
```

### Kết luận
Vậy là ta đã nói xong về Kubernetes Deploymnet, sử dụng nó sẽ giúp ta dễ dàng hơn trong việc deploy version mới của ứng dụng trong quá trình CI/CD. Cảm ơn các bạn đã đọc bài của mình. Nếu có thắc mắc hoặc cần giải thích rõ thêm chỗ nào thì các bạn có thể hỏi dưới phần comment. Hẹn gặp lại các bạn ở bài tiếp theo mình sẽ nói về **Kubernetes Volumes**, đây là resource mà giúp ta có thể lưu trữ data persistence.