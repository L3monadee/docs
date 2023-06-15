## Giới thiệu
Chào mọi người đến với series practice về kubernetes của mình. Tiếp tục ở [bài trước](https://viblo.asia/p/kubernetes-practice-trien-khai-nodejs-microservice-tren-kubernetes-phan-1-viet-config-4P856GRRKY3) (các bạn nên đọc bài trước để hiểu về hệ thống ta đang triển khai ra sao). Ở bài trước chúng ta đã nói về cách triển khai nodejs microservice trên môi trường kubernetes, ta đã nói qua về từng thành phần và cách viết config cho từng thành phần đó và cách deploy chúng lên môi trường kubernetes.

## Kiến trúc của ứng dụng
![alt](https://github.com/hoalongnatsu/k8s-practice/blob/main/microservices/microservices.png?raw=true)

## Tự động cập nhật ứng dụng khi application template thay đổi
Một application template sẽ chứa toàn bộ config file của tất cả các thành phần trong ứng dụng của ta.

Các bạn sẽ để ý rằng mỗi lần ta viết file config cho một thành phần mới và thêm nó vào trong application template, ta phải chạy lại câu lệnh kubectl để tạo ra nó. Hoặc khi ta thay đổi config của một thành phần có sẵn, ta cũng phải chạy câu lệnh kubectl để cập nhật lại config cho thành phần đó.

Thì chúng ta sẽ nói qua cách làm sao để khi ta thêm một file config mới hoặc thay đổi file config nào đó trong application template, thì ứng dụng trên kubernetes của chúng ta sẽ tự động cập nhật lại. Có rất nhiều cách để ta làm được việc này, ta có thể sử dụng jenkins, gitlab CI để thực hiện quá trình này tự động. Nhưng khi đụng vào bạn sẽ thấy có khá nhiều việc để làm.

Ví dụ với gitlab CI, thì ta sẽ tạo một repo cho application template và nó sẽ chứa toàn bộ file config của ứng dụng, khi ta thay đổi file config nào đó, push lên gitlab, gitlab CI sẽ trigger và thực hiện quá trình CI/CD. Với gitlab thì ta sẽ thực hiện CI/CD bằng cách viết config trong file `.gitlab-ci.yml`, ví dụ ta có một file .gitlab-ci.yml như sau:

```yaml
deploy template:
  stage: deploy
  tags:
    - kala
  script:
    - rsync -avz -e 'ssh -i ~/.ssh/kala.pem' . ec2-user@$SERVER:~/kubernetes --delete --force
    - |
      sudo ssh -i ~/.ssh/kala.pem ec2-user@$SERVER << EOF
        cd ~/kubernetes
        kubectl apply -f . --recursive
        exit
      EOF
  when: manual
  only:
    - production
```

Quá trình CI/CD của gitlab đơn giản sẽ như sau. Khi ta push code lên gitlab, nó sẽ kiểm tra xem trong repo của chúng ta có file `.gitlab-ci.yml` hay không, nếu có thì nó sẽ trigger quá trình CI/CD này. Đầu tiên, nó pull code của repo xuống server ta chạy CI/CD và sau đó thực hiện công việc ở trường script. Ở trên trường script của ta sẽ thực hiện hai công việc như sau:

+ Công việc thứ nhất là nó sẽ thực hiện câu lệnh rsync (rsync đơn giản chỉ là một câu lệnh giúp ta chuyển file từ máy này sang máy khác), câu lệnh ở trên nó sẽ chuyển tất cả các file cấu hình ứng dụng trong repo của chúng ta lên trên server mà đang chạy kubernetes.
+ Công việc thứ hai là nó sẽ ssh tới server kubernetes master node, di chuyển vào folder ban nãy ta chuyển config file lên, và thực hiện câu lệnh kubectl apply để cập nhật lại application của chúng ta theo template mới.

Tuy đây cũng là một cách để chúng ta cập nhật lại ứng dụng một cách tự động. Nhưng điểm yếu của cách này là khi ta chạy câu lệnh trên, toàn bộ file config đang có sẽ bị tác động hết, cái ta mong muốn là khi ta thay đổi config của thành phần nào thì chỉ thành phần đó mới được apply. Và để làm được việc đó thì ta cần phải viết một đoạn script khá dài và cũng không đơn giản chút nào. Việc này khá là tốn thời gian.

Thì thay vì phải tự viết script để thực hiện CI/CD, thì ta có một số công cụ hỗ trợ công việc này tự động. Một trong những công cụ phổ biến nhất là Argocd.

## Argocd
Đây là một công cụ hỗ trợ công việc tự động cập nhật lại application của chúng ta khi ta thêm hoặc thay đổi config nào đó, nhưng thay vì cập nhật lại toàn bộ như trên thì nó chỉ cập nhật lại những thành phần nào mà có thay đổi config, và tạo thêm thành phần mới nếu ta có thêm file config cho thành phần mới.

![image.png](./images/b77c926a-552a-486f-b848-896fa4074a0f.png)

### Cài đặt Argocd
Để cài Argocd, các bạn làm theo các bước sau đây

**1. Install Argo CD**
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Sau khi chạy câu lệnh xong thì ta chờ một chút để Argo CD install tất cả các Deployment và Service resource của nó.

**2. Install Argo CD CLI**

Đây là cách cho linux:
```
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

Các môi trường khác các bạn có thể xem ở đây https://argo-cd.readthedocs.io/en/stable/cli_installation/.

**3. Access The Argo CD API Server**

Ta chạy câu lệnh get pod để kiểm tra toàn bộ Pod của chúng ta đã chạy thành công hay chưa.

```
$ kubectl get pod -n argocd
argocd-application-controller-0      1/1     Running   0          2m23s
argocd-dex-server-5fc596bcdd-7nvlp   1/1     Running   0          2m23s
argocd-redis-5b6967fdfc-xtns6        1/1     Running   0          2m23s
argocd-repo-server-98598b6c7-chr9k   1/1     Running   0          2m23s
argocd-server-5b4b7b868b-ffs9w       1/1     Running   0          2m23s
```

Nếu tất cả đều chạy thành công, để truy cập Argo CD thì ta có thể sử dụng NodePort Service, Ingress. Ở đây để nhanh thì mình dùng Port Forwarding.

```
$ kubectl port-forward svc/argocd-server -n argocd 8080:443
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

Bây giờ thì ta mở web browser và truy cập vào địa chỉ  `http://localhost:8080`. Vì ta chạy localhost nên khi nó báo unsafe, thì ta chỉ việc bấm proceed to localhost (unsafe) là được.

![image.png](./images/d07b9637-7c41-4477-9392-a4f8246f2e19.png)

Tới đây thì bạn sẽ thấy UI như sau

![image.png](./images/5ec1605d-8e10-466e-a6de-77ec72cd78f4.png)

Với username của ta sẽ là **admin**, và password bạn lấy bằng cách chạy câu lệnh như sau:

```
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Sau đó ta lấy password được in ra terminal để đăng nhập. Sau khi login sau bạn sẽ thấy được giao diện.

![image.png](./images/80385e3f-fd68-49ed-abfb-ad441f53425b.png)

Nếu bạn muốn update lại password thì ta làm như sau, trước tiên chạy câu lệnh:

```
$ argocd login <ARGOCD_SERVER>
```

Với ARGOCD_SERVER là IP của máy ta + port 8080 ở trên. Sau khi login sau thì ta chạy câu lệnh:

```
$ argocd account update-password
```

Ok, vậy là ta đã cài đặt xong được Argocd, bước tiếp theo là ta dùng nó để tự động tạo ứng dụng và cập nhật lại ứng dụng theo application template của chúng ta.

### Tạo một App trên Argocd
Để kết nối Argocd tới kubernetes cluster và git repo, ta cần tạo một APP trên Argocd. Nhấn vào nút **+ New App** ở trên UI.

![image.png](./images/91bd3d14-ce3d-4ff3-a12c-ad9896df8924.png)

Bạn sẽ thấy nó mở ra một form như sau:

![image.png](./images/f70acc18-312d-4712-bc7c-a4bc662f3c34.png)

Điền vào trường Application Name với tên bạn muốn, ở đây mình sẽ điền là nodejs-microservice, trường Project bạn chọn default. Trường SYNC POLICY sẽ có hai giá trị là Manual với Automatic, với Manual là khi bạn push code lên git, bạn phải vào bấm bằng tay để ứng dụng ta cập nhật lại theo template config mới, còn Automatic thì nó sẽ tự động luôn, giá trị này bạn có thể thay đổi bất cứ lúc nào cũng được. Ở đây mình sẽ chọn giá trị là Automatic. Các thuộc tính SYNC OPTIONS tạm thời ta cứ bỏ qua. Đi tiếp tới phần form tiếp theo.

![image.png](./images/97d14bc0-27aa-4fd5-8802-b6bf170daf92.png)

Đây là phần quan trọng, bạn sẽ điền vào trường Repository URL đường dẫn tới git repo mà bạn chứa template config của ứng dụng. Ở đây mình sẽ điền vào đường dẫn repo của mình. Trường Path, ta sẽ điền vào tên folder mà chứa file kubernetes config của ứng dụng, nếu nằm ở root thì bạn điền vào đường dẫn là /, của mình thì mình sẽ điền vào giá trị là k8s.

Form tiếp theo ta sẽ điền vào như sau.

![image.png](./images/3387f0ed-4973-4cfc-9e03-76770ed27909.png)

Với Cluster URL bạn điền giá trị `https://kubernetes.default.svc`, còn namespace bạn điền tên namespace bạn muốn ứng dụng ta deploy tới. Ở đây mình điền là default. Sau khi điền sau hết, bạn nhấn nút create, lúc này Argocd sẽ tạo một APP cho chúng ta và tiến hành deploy ứng dụng lên trên kubernetes cluster của chúng ta.

![image.png](./images/8a0b5d76-77f0-4ac0-b23e-11b66f5e95fd.png)

Sau khi tạo xong bạn sẽ thấy UI như hiện tại, bạn để ý thấy trường status, nếu nó hiển thị giá trị synced có nghĩa là nó đã sync ứng dụng của chúng ta lên trên môi trường kubernetes thành công. Bạn có thể kiểm tra bằng cách gõ câu lệnh get deployment để xem các thành phần trong ứng dụng nodejs microservice của chúng ta.

```
$ kubectl get deploy
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
api-gateway          1/1     1            1           4m
categories-service   1/1     1            1           4m
nats                 1/1     1            1           4m
news-service         1/1     1            1           4m
redis                1/1     1            1           4m
```

Nếu bạn hiển thị ra được tất cả các Deployment thì chúc mừng là ta đã deploy được ứng dụng thành công. Ngoài ra Argocd còn có UI hiển thị các thành phần của chúng ta trên kubernetes kết nối với nhau ra sao và tình trạng hiện của các resource như thế nào, rất hữu ích, bạn nhấn vào APP nodejs-microservice, bạn sẽ thấy UI như sau, rất đẹp và chi tiết.

![image.png](./images/723b90af-ea9a-43e7-8859-eb916dea26f6.png)

Bây giờ thì khi bạn thay đổi một file config nào đó, push code lên github, thì Argocd sẽ tự động cập nhật lại APP cho chúng ta, ta không cần phải viết file CI/CD để thực hiện quá trình này nữa.

### Kết nối với Private Repositories
Nếu bạn có git riêng, thì để kết nối với Private Repositories, ta làm theo bước sau, chọn icon Settings và chọn Repositories.

![image.png](./images/f4c498af-34d9-470f-b920-f2738cdb9229.png)

Có rất nhiều lựa chọn, tùy thuộc vào trường hợp thì bạn sẽ lựa cách phù hợp.

![image.png](./images/7da50164-a358-4a50-af0b-ef4a98d48a0c.png)

Ví dụ bạn bấm vào Connect repo using HTTPS thì nó sẽ hiện ra một form như sau, bạn chỉ cần điền vào thông tin chính xác là ta có thể kết nối được tới Private Repositories

![image.png](./images/84b507d5-3cf6-4937-930c-d39ae1f45889.png)

### Xóa ứng dụng
Để xóa ứng dụng trên Argocd rất đơn giản, ở ngoài phần quản lý APP, bạn chỉ cần nhấn vào icon xóa, là tất cả mọi thứ trên APP nó trong kubernetes của chúng ta sẽ được remove đi hết.

![image.png](./images/a824d760-9570-4697-bf4b-3f7f026a2349.png)

Bạn chạy câu lệnh get deployment thì sẽ thấy toàn bộ resource của ta đã bị xóa đi.

```
$ kubectl get deploy
No resources found in default namespace.
```

Cẩn thận khi xóa APP nhé 😂.

## Kết luận
Vậy là ta đã tìm hiểu xong cơ bản về cách sử dụng Argocd, sử dụng công cụ này sẽ giúp bạn thoải mái hơn trong quá trình cập nhật lại một template config mới cho ứng dụng kubernetes của ta. Ở phần này ta chỉ nói về cách cập nhật template. Phần tiếp theo mình sẽ nói về cách làm sao khi các dev viết code cho chức năng mới xong, dev đẩy lên git repo. Ta sẽ viết CI/CD thế nào để tự động cập nhật lại các chức năng mới cho ứng dụng của chúng ta. Nếu các bạn có thắc mắc hoặc chưa hiểu rõ chỗ nào, các bạn có thể hỏi ở phần comment.