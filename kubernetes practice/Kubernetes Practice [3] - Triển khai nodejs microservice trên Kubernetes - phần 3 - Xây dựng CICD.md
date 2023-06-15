## Giới thiệu
Chào mọi người đến với series practice về kubernetes của mình. Tiếp tục ở bài các phần trước, các bạn nên đọc bài trước để hiểu về hệ thống ta đang triển khai ra sao, [phần 1](https://techhay.vn/kubernetes-practice-trien-khai-nodejs-microservice-tren-kubernetes-phan-1/) nói về cách viết config và cách triển khai microservice lên trên Kubernetes, [phần 2](https://techhay.vn/kubernetes-practice-trien-khai-nodejs-microservice-tren-kubernetes-phan-2/) nói về ta dùng Argocd để tự động cập nhật application khi template config kubernetes của ta thay đổi. Ở bài này, mình sẽ nói về cách khi developer viết code cho chức năng mới của ứng dụng xong và push nó lên trên git repo, thì ta sẽ xây dựng luồng CI/CD thế nào để cập nhật lại ứng dụng với code mới của developer trên môi trường kubernetes.

Ở bài này mình sẽ sử dụng repository là gitlab và chạy CI/CD với gitlab CI. Để làm được bài này thì các bạn cần tạo một repo trên gitlab và đẩy code lên đó nhé, các bạn đặt tên repo là gì cũng được. Các bạn đẩy code nằm ở trong folder code của repo này https://github.com/hoalongnatsu/microservices.git lên repo gitlab của các bạn nha. Sau khi xong hết ta bắt tay vào làm.

## Gitlab Runner
Gitlab cung cấp cho ta khá nhiều cách để chạy CI/CD, mình sẽ nói tới phần đơn giản nhất trước, là ta sử dụng Gitlab Runner. Gitlab Runner là một application mà ta sẽ cài ở trên server ta muốn nó chạy CI/CD. Để install gitlab runner, các bạn ssh tới server mà mình muốn chạy CI/CD trên đó, và cài runner như sau, mình sẽ hướng dẫn cài ở trên server linux.

### Install gitlab-runner
**Download the binary for your system**

Chọn OS phù hợp với server của các bạn nhé.

```
# Linux x86-64
sudo curl -L --output /usr/local/bin/gitlab-runner "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64"

# Linux x86
sudo curl -L --output /usr/local/bin/gitlab-runner "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-386"

# Linux arm
sudo curl -L --output /usr/local/bin/gitlab-runner "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-arm"

# Linux arm64
sudo curl -L --output /usr/local/bin/gitlab-runner "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-arm64"

# Linux s390x
sudo curl -L --output /usr/local/bin/gitlab-runner "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-s390x"

# Linux ppc64le
sudo curl -L --output /usr/local/bin/gitlab-runner "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-ppc64le"
```

**Give it permissions**
```
sudo chmod +x /usr/local/bin/gitlab-runner
```

**Create a GitLab CI user**
```
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
```

**Install and run as service**
```
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
```

Nếu bạn gặp lỗi `sudo: gitlab-runner: command not found` thì mở /etc/sudoers lên và thêm vào ở `Defaults secure_path` với dường dẫn `/usr/local/bin` nữa, còn không thì bạn install gitlab-runner ở dường dẫn `/usr/bin` cũng được.

Để install trên các môi trường khác thì các bạn xem thêm ở đây https://docs.gitlab.com/runner/install/.

Sau khi install xong thì bạn có thể dùng câu lệnh này để kiểm tra:

```
$ sudo gitlab-runner list
Runtime platform                                    arch=amd64 os=linux pid=2850 revision=de104fcd version=14.5.1
Listing configured runners                          ConfigFile=/etc/gitlab-runner/config.toml
```

Câu lệnh này là để ta list toàn bộ runner hiện tại trên server, bây giờ thì tất niên khi ta list ra thì sẽ không có runner nào 😁. Giờ thì ta sẽ đăng ký một con runner để nó chạy CI/CD cho repo của ta ở trên gitlab. Bạn mở gitlab lên, ở chỗ repo của mình chọn mục Settings -> 
CI/CD.

![image.png](./images/9986aa22-39d0-4c90-ad8f-0b65c720e7ec.png)

Sau đó bạn mở mục Runners ra.

![image.png](./images/3fe1a21b-ed16-494a-afb5-59d5d96a6331.png)

Sau khi mở ra, bạn sẽ thấy hai trường mà ta sẽ dùng nó để register runner cho repo của ta là gitlab url và token.

![image.png](./images/98fc7743-c49c-430b-a034-46e99daee4c1.png)

### Register runner
Giờ thì ta sẽ register một con runner cho repo của ta lên trên server, các bạn làm theo các bước như sau, đầu tiên ta chạy câu lệnh register.

```
sudo gitlab-runner register
```

Sau khi run thì nó sẽ bắt bạn nhập gitlab url vào, nhập giá trị url ở phía trên vào. Tiếp đó nó sẽ bắt bạn nhập token, bạn nhập token ở trên vào.

![image.png](./images/b454e6b3-a366-4927-bbe2-19beb9f12a39.png)

Tiếp theo là nó bắt bạn nhập description, chỗ này thì bạn nhập gì cũng được. Tiếp theo là phần tags, phần này thì quan trọng, lúc chạy CI thì runner sẽ được chạy dựa vào tags, bạn nhập ở đây là microservice nhé.

![image.png](./images/e22cc92e-74de-4de1-869d-1339a6889812.png)

Bây giờ thì sẽ tới bước ta chọn runner của ta sẽ được thực thi ra sao. Nó sẽ chạy thẳng trên server, hay là mỗi lần chạy nó sẽ tạo một container ra và chạy job CI/CD trên container đó. Chọn shell nếu các bạn muốn runner chạy thẳng trên môi trường server, chọn docker nếu bạn muốn runner chạy trong môi trường docker. Ở đây thì mình sẽ chọn docker, các bạn nên chọn theo mình để tránh bị lỗi nha.

![image.png](./images/1f39b48b-dcdb-4b6f-ab44-2b11370a59f0.png)

Tiếp theo nó sẽ hỏi default Docker image thì bạn nhập vào là `docker:stable`, tới đây thì ta đã register runner cho repo của ta thành công, để kiểm tra các runner ta đã đăng ký thì ta chạy câu lệnh list ở trên, giờ thì bạn sẽ thấy được runner mà ta vừa đăng ký.

```
Runtime platform                                    arch=amd64 os=linux pid=2969 revision=de104fcd version=14.5.1
Listing configured runners                          ConfigFile=/etc/gitlab-runner/config.toml
microservice                                        Executor=docker Token=zC8B6PZzDouA46mgGry6 URL=https://gitlab.kala.ai/
```

Kiểm tra trên gitlab bạn sẽ thấy nó hiển thị con runner của ta.

![image.png](./images/cedd4e07-d6da-41c4-83af-a14210facd59.png)

Oke vậy là ta đã xong bước register runner. Và thay vì phải nhập từng bước khá lằng nhằng như trên thì bạn có thể register nhanh một con runner bằng câu lệnh sau:

```
sudo gitlab-runner register -n \
  --url https://gitlab.com/ \
  --registration-token REGISTRATION_TOKEN \
  --executor docker \
  --description "microservice" \
  --docker-image "docker:stable" \
  --docker-privileged
```

Giờ thì ta chỉ cần viết file CI/CD ở trong repo của chúng ta, và mỗi lần ta đẩy code lên thì file CI/CD này sẽ được con runner thực thi. **Lưu ý là vì ta chọn docker nên bắt buộc trên server chạy CI/CD của ta phải có cài docker rồi thì runner mới thực thi được**. Tiếp theo ta sẽ bắt tay vào làm luồng CI.

## Continuous Integration
Đầu tiên là ta sẽ làm bước mà integrate code mới của developer với image mà dùng để chạy ứng dụng của ta trước, sau đó ta mới làm bước deploy. Ở bước integrate này thì ta sẽ viết file CI, công việc của nó sẽ là build image tương ứng với code của branch hiện tại, sau đó sẽ push image đó lên docker registery (nơi ta chứa image).

Gitlab có hỗ trợ cho ta image registry, nên ta sẽ sử dụng nó luôn cho tiện. Để viết CI/CD cho gitlab thì ta sẽ viết trong file `.gitlab-ci.yml`, ở folder mà ta chứa code của repo, tạo một file .gitlab-ci.yml với config như sau:

![image.png](./images/a47a005d-1e71-40c0-b671-58fa55807a1b.png)

```yaml
stages:
  - build

build root image:
  stage: build
  tags:
    - microservice
  script:
    - docker build . -t $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main
```

Trường stages là ta sẽ định nghĩa những pipe trong một job của ta, hiện tại thì ta chỉ có một pipe là build thôi. Ở trong file config trên ta sẽ có biến là CI_REGISTRY_IMAGE, đây là một trong những biến mặc định của gitlab CI, biến CI_REGISTRY_IMAGE sẽ là tên image của chúng ta. Ở trên gitlab repo, bạn bấm sang phần **Package & Registries -> 
Container Registry**, thì bạn sẽ thấy được tên image của chúng ta. Khi ta push image thì ta sẽ push lên chỗ Container Registry này. Trường script là trường ta sẽ chỉ định những công việc ta sẽ làm, ở file trên đơn giản là ta sẽ build image và push image với code mới nhất lên.

![image.png](./images/80489829-8321-4030-ae0a-e901ffe93107.png)

Giờ ta commit và push code lên, ta sẽ thấy CI/CD của ta sẽ được thực thi.

![image.png](./images/78550b10-7ee2-42fb-929a-af176f0509d6.png)

Bạn nhấn vào chữ running thì nó sẽ dẫn bạn vào trong.

![image.png](./images/ce22ca29-abc6-48fa-a2af-27320620531f.png)

Sau đó bạn nhấn vào chỗ build thì nó sẽ dẫn ta qua chỗ coi log.

![image.png](./images/7626b67c-3258-4a87-93a8-ea864cb52d4c.png)

Runner của ta sẽ chạy và nó tự động pull code của chúng ta xuống container đang chạy job, sau đó nó sẽ thực thi build image.

![image.png](./images/098b4574-72d0-4c09-bbbc-88aeaca98ff3.png)

Sau khi image build xong thì nó sẽ push lên gitlab registry, và ở đây bạn sẽ thấy nó có lỗi là `denied: access forbidden`.

![image.png](./images/0b19c8e8-d50a-4043-a3e6-be2463fcce67.png)

Lý do là do ta chưa login tới gitlab registry để docker có thể push image lên được. Để login vào gitlab registry, ta làm như sau.

### Login gitlab registry
Ta chọn phần Settings -> Repository, chọn mục Deploy tokens

![image.png](./images/b98f45df-a067-40b7-b87a-7a2166dcab88.png)

Ở phần Deploy tokens, bạn nhập trường name và username là microservice (này bạn muốn nhập gì cũng được).

![image.png](./images/e41ce681-37d0-4dc4-a1de-253f099b7df7.png)

Chỗ scope thì bạn tạm thời chứ chọn hết. Sau đó ta nhấn tạo.

![image.png](./images/8e7c3d56-6b24-4065-a427-2b86ef2e9ad9.png)

Sau khi nhấn tạo xong thì nó sẽ hiện ra cho bạn hai trường là username và password, nhớ lưu lại nha.

![image.png](./images/2510ca02-77d3-4887-a494-e1ac4e445939.png)

Sau khi có deploy token rồi, bạn chọn mục Settings -> CI/CD, mục Variables. Tạo một env là REGISTRY_PASSWORD để lưu giá trị password của deploy token.

![image.png](./images/96a76b76-93b9-433f-86d5-6a0a46e4cb3d.png)

![image.png](./images/2d6a8f68-a437-4ee3-bba4-0c9df91f58aa.png)

Ok, sau khi tạo xong thì bạn update lại file .gitlab-ci.yml như sau:

```yaml
stages:
  - build

build root image:
  stage: build
  tags:
    - microservice
  before_script:
    - echo "$REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u microservice --password-stdin
  script:
    - docker build . -t $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main
```

Ta sẽ thêm trường before_script để chạy câu lệnh login vào gitlab  registry. Biến CI_REGISTRY là biến mặc định của gitlab CI, biến REGISTRY_PASSWORD là ta vừa mới tạo. Cập nhật xong thì ta commit code và push nó lên lại. Lúc này bạn sẽ thấy là ta đã build được image và push nó lên thành công.

![image.png](./images/62dea37b-9d01-479b-bd00-f6e76c329aba.png)

Nhưng bạn sẽ để ý là vì ta chạy build image trong một container, nên khi ta build nó không có lưu cache, khiến việc ta build image bị lâu hơn, để khác phục việc này thì trước khi build ta sẽ pull image đã build xuống trước, để nó có layer cache, và khi ta build ta sẽ chỉ định thêm option là `--cache-from $CI_REGISTRY_IMAGE:latest`. Cập nhật lại file .gitlab-ci.yml.

```yaml
stages:
  - build

build root image:
  stage: build
  tags:
    - microservice
  before_script:
    - echo "$REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u microservice --password-stdin
  script:
    - docker pull $CI_REGISTRY_IMAGE:latest || true
    - docker build . --cache-from $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main
```

Commit code và push lên, lúc này thì image của ta sẽ được build với tốc độ nhanh hơn.

### Cập nhật lại file config của kubernetes
Tiếp theo ta sẽ cập nhật lại file config Deployment với image mà ta mới build. Để coi tên image, bạn chọn mục copy ở phần Container Registry.

![image.png](./images/e0d99075-6963-474e-b741-60bf7e30ba51.png)

Các file Deployment ở folder k8s của repo này https://github.com/hoalongnatsu/microservices.git, như api-gateway-deployment.yaml, categories-news-deployment.yaml bạn sửa lại thành tên image của gitlab

```yaml

# api-gateway-deployment.yaml
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
          image: registry.kala.ai/microservice/code
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

```yaml
# categories-news-deployment.yaml
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
          image: registry.kala.ai/microservice/code
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
          image: registry.kala.ai/microservice/code
          env:
            - name: SERVICES
              value: news
          envFrom:
            - configMapRef:
                name: microservice-cm
```

Image của mình thì mình sửa lại là `registry.kala.ai/microservice/code`. Sau đó nếu bạn làm như theo bài trước ở phần 2, thì bạn chỉ cần push config lên repo chứa template config để Argocd nó tự động cập nhật lại. Còn không thì bạn chạy câu lệnh sau ở folder k8s.

```
$ kubectl apply -f k8s --recursive
deployment.apps/api-gateway created
deployment.apps/categories-service created
deployment.apps/news-service created
configmap/microservice-cm created
deployment.apps/nats created
service/nats created
service/postgres created
statefulset.apps/postgres created
deployment.apps/redis created
service/redis created
```

Sau khi ta chạy câu lệnh trên thì ta đã deploy ứng dụng của ta lên kubernetes với image là image mà ta mới build và push trên gitlab. Giờ ta sẽ get pod ra để xem thử.

```
$ kubectl get pod
NAME                                  READY   STATUS             RESTARTS   AGE
api-gateway-84677bf776-jj8x4          0/1     ErrImagePull       0          86s
categories-service-867f848c77-l2c4m   0/1     ImagePullBackOff   0          86s
nats-65687968fc-rbtd4                 1/1     Running            0          86s
news-service-686b8557c8-ch2v5         0/1     ImagePullBackOff   0          86s
postgres-0                            1/1     Running            0          86s
redis-58c4799ccc-xn9zk                1/1     Running            0          86s
```

Nhưng khi bạn get pod ra, bạn sẽ thấy những Deployment mà ta chỉ định dùng image của gitlab thì nó sẽ bị lỗi là ErrImagePull. Bạn describe nó để xem lý do thì sẽ thấy là do những node mà pod được deploy tới không có quyền để pull image từ gitlab của ta xuống.

```
$ kubectl describe pod api-gateway-84677bf776-jj8x4
...
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  ...
  Normal   Pulling    61s (x4 over 2m48s)  kubelet            Pulling image "registry.kala.ai/microservice/code"
  Warning  Failed     61s (x4 over 2m37s)  kubelet            Failed to pull image "registry.kala.ai/microservice/code": rpc error: code = Unknown desc = Error response from daemon: pull access denied for registry.kala.ai/microservice/code, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed     61s (x4 over 2m37s)  kubelet            Error: ErrImagePull
  Warning  Failed     36s (x6 over 2m37s)  kubelet            Error: ImagePullBackOff
  Normal   BackOff    25s (x7 over 2m37s)  kubelet            Back-off pulling image "registry.kala.ai/microservice/code"
```

Để kubernetes có thể pull được image từ gitlab, ta cần tạo một Secret với loại là docker-registry rồi chỉ định vào trường **imagePullSecrets** khi khai báo config cho Pod. Ta tạo Secret bằng câu lệnh sau.

```
$ kubectl create secret docker-registry microservice-registry --docker-server=https://registry.kala.ai --docker-username=microservice --docker-password=8Su2-9MnqT7Qt4uNyfqz
secret/microservice-registry created
```

Với **--docker-server** là tên gitlab server của bạn, **--docker-username** là username của deploy token ta tạo khi nãy, và **--docker-password** là password của deploy token. Cập nhật lại file Deployment thêm vào trường imagePullSecrets với Secret ta vừa tạo.

```yaml

# api-gateway-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  labels:
    component: api-gateway
spec:
    ...
    spec:
      imagePullSecrets:
        - name: microservice-registry
      containers:
        - name: api-gateway
          image: registry.kala.ai/microservice/code
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
          ...
```

```yaml
# categories-news-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: categories-service
  labels:
    component: categories-service
spec:
    ...
    spec:
      imagePullSecrets:
        - name: microservice-registry
      containers:
        - name: categories-service
          image: registry.kala.ai/microservice/code
          ...

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: news-service
  labels:
    component: news-service
spec:
    ...
    spec:
      imagePullSecrets:
        - name: microservice-registry
      containers:
        - name: news-service
          ...
```

```
$ kubectl apply -f k8s --recursive
deployment.apps/api-gateway configured
deployment.apps/categories-service configured
...
statefulset.apps/postgres configured
...

$ kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
api-gateway-6c9f8f68b7-mj6r9          1/1     Running   0          40s
categories-service-84db8d885b-g85md   1/1     Running   0          40s
nats-65687968fc-rbtd4                 1/1     Running   0          19m
news-service-76d559db69-vqg59         1/1     Running   0          40s
postgres-0                            1/1     Running   0          19m
redis-58c4799ccc-xn9zk                1/1     Running   0          19m
```

Vậy là ta đã hoàn thành xong bước CI, tiếp theo ta sẽ xem cách làm CD.

## Continuous deployment
Bước này thì ta cũng sẽ có nhiều cách, mình sẽ nói về cách dễ nhất trước là ta sẽ ssh lên kubernetes master và thực hiện câu lệnh kubectl, các cách khác mình sẽ nói ở bài sau. Cập nhật lại file .gitlab-ci.yml như sau:

```yaml
stages:
  - build
  - deploy

build root image:
  stage: build
  tags:
    - microservice
  script:
    - docker build . -t $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main

deploy stage:
  stage: deploy
  tags:
    - microservice
  before_script:
    - mkdir ~/.ssh
    - echo -e "${SERVER_KEY_PEM//_/\\n}" > ~/.ssh/key.pem
    - apt-get -y update && apt-get -y install openssh-client rsync grsync
    - ssh-keyscan -H $SERVER_IP >> ~/.ssh/known_hosts
    - chmod 400 ~/.ssh/rnd-ecommerce.pem
  script:
    - |
      sudo ssh -i ~/.ssh/key.pem $SERVER_USER@$SERVER_IP << EOF
        kubectl set image deployment/api-gateway api-gateway=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        kubectl set image deployment/news-service news-service=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        kubectl set image deployment/categories-service categories-service=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        exit
      EOF
  only:
    - main
```

Ta sẽ thêm một stages nữa là deploy. Để ssh được tới server thì tùy cách bạn dùng là password hoặc key pem thì sẽ khác nhau. Của mình sẽ dùng cách key pem. Bạn vào phần Settings -> CI/CD -> Variables khi nãy, khai báo thêm các biến là SERVER_KEY_PEM, SERVER_USER, SERVER_IP tương ứng với server mà đang chạy kubernetes master. Khi ta build image và push lên gitlab xong, job deploy sẽ được chạy, nó sẽ ssh tới kubernetes master và cập nhật lại Deployment với image là image là mới build xong. **Vì ở đây mình làm demo nên mình gộp 3 câu cập nhật Deployment lại trong một repo, còn thực tế các bạn nên làm mỗi repo một Deployment khác nhau nhé**.

Commit và push code lên, lúc này bạn sẽ thấy pipe của ta sẽ có 2 job là build với deploy.

![image.png](./images/fe8cd4dc-e2b2-45ae-9512-f62d5006ab82.png)

![image.png](./images/aa1e303d-6b80-4526-9034-2feace623570.png)

Vào xem log phần deploy.

![image.png](./images/c4d1d76b-5678-439e-80ba-d65560c7d9a9.png)

Ok, vậy là luồng CD của ta đã chạy thành công 😁.

## Kết luận
Vậy là ta đã tìm hiểu xong cách xây dựng luồn CI/CD với kubernetes, khi ta xây dựng CI/CD thì công việc deploy của ta trở nên dễ dàng hơn nhiều. Nếu các bạn có thắc mắc hoặc chưa hiểu rõ chỗ nào, các bạn có thể hỏi ở phần comment. Ở phần tiếp theo mình sẽ nói về cách integrate gitlab CI với thẳng kubernetes, không cần phải chạy gitlab runner, và cách xài RBAC để thiết lập permission cho từng job với namespace cụ thể.