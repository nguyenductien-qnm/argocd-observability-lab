# Evidence

## 1. Mục tiêu bài làm

Mục tiêu của bài làm là triển khai một quy trình phát hành an toàn cho `api` trên Kubernetes theo GitOps, có quan sát được chất lượng dịch vụ, có cảnh báo email khi chất lượng giảm, và có khả năng tự rollback khi bản mới không đạt chất lượng.

Các ý chính đã hiện thực:

- mọi thay đổi đi qua Git
- Argo CD đồng bộ trạng thái từ Git vào cluster
- Argo Rollouts triển khai canary cho `api` và `frontend`
- Prometheus scrape metric từ `api`
- đã có SLO dựa trên error rate của `/api/overview`
- Alertmanager gửi email khi vi phạm ngưỡng
- rollout của `api` dùng AnalysisTemplate để tự đánh giá và abort khi bản mới xấu

## 2. Kiến trúc tổng thể

### 2.1 Các repo

Hệ thống được tách thành 3 repo:

- `api`: mã nguồn backend Node.js
- `web`: mã nguồn frontend
- `gitops`: nguồn sự thật cho manifest Kubernetes và observability

Ngoài 3 repo này, image của `api` và `frontend` được build và đẩy lên `Amazon ECR`. Kubernetes không build image trong cluster, mà chỉ pull image đã được publish sẵn từ ECR thông qua `imagePullSecrets`.

Việc tách repo như vậy giúp:

- app code và infra không dính vào nhau
- thay đổi triển khai được quản lý tập trung trong GitOps repo
- Argo CD chỉ cần theo dõi một repo duy nhất để đồng bộ xuống cluster
- vòng đời build image và vòng đời deploy manifest được tách riêng, rõ ràng

### 2.2 Thành phần chính trong cluster

- `Argo CD`: đồng bộ trạng thái từ GitOps repo vào cluster
- `Argo Rollouts`: điều khiển canary rollout
- `Prometheus`: scrape metric và đánh giá chất lượng API
- `Alertmanager`: gửi email khi alert firing
- `Grafana`: quan sát dashboard
- `Loki + Promtail`: thu thập log
- `frontend`: giao diện người dùng
- `api`: backend có `/api/overview`, `/health`, `/metrics`
- `Amazon ECR`: nơi lưu image cho cả `frontend` và `api`

### 2.3 Luồng hoạt động

```text
Git push
  -> Argo CD sync
  -> cập nhật rollout/frontend/observability

Traffic vào API
  -> /api/overview
  -> API sinh metric qua prom-client
  -> Prometheus scrape /metrics

Prometheus
  -> tính error rate
  -> dùng cho alert email
  -> dùng cho AnalysisTemplate của Argo Rollouts

Nếu error rate tốt
  -> rollout đi tiếp tới stable

Nếu error rate xấu
  -> AnalysisRun fail
  -> rollout abort
  -> quay về stable revision trước đó
```

Luồng build và phát hành image:

```text
api repo / web repo
  -> build Docker image
  -> push image lên Amazon ECR
  -> gitops repo cập nhật rollout / env / manifest
  -> Argo CD sync
  -> cluster pull image từ ECR
```

## 3. Cách tổ chức GitOps repo

Repo `gitops` được tổ chức theo mô hình `App of Apps`.

### 3.1 Root app

- `root-app` là Application gốc trong Argo CD
- `root-app` theo dõi thư mục `argocd-apps/`
- mỗi file trong `argocd-apps/` là một Application con

Điều này cho phép:

- không cài tay từng app
- chỉ cần thêm file YAML vào Git là Argo CD tự nhận diện
- dễ mở rộng khi cần thêm workload hoặc công cụ quan sát

### 3.2 Các application con

Các Application con chính:

- `api`
- `frontend`
- `argo-rollouts`
- `prometheus`
- `grafana`
- `loki`
- `promtail`
- `monitoring-secrets`

Trong đó:

- `api` và `frontend` là workload nghiệp vụ
- `argo-rollouts` cài controller phục vụ canary
- `prometheus/grafana/loki/promtail` là stack observability
- `monitoring-secrets` cung cấp Secret cho monitoring, ví dụ SMTP password

Riêng repo `gitops` đóng vai trò trung tâm:

- khai báo app nào được chạy trong cluster
- khai báo chart/values cho observability
- khai báo rollout strategy của `api` và `frontend`
- giữ toàn bộ manifest cần thiết để cluster có thể được tái tạo lại từ Git

## 4. So sánh với bộ yêu cầu / CSI

Phần này đối chiếu cách làm thực tế với bộ yêu cầu trong ảnh bài lab.

### 4.1 Lab 1 - Cài Prometheus + Argo Rollouts qua GitOps

Yêu cầu:

- không cài tay
- tạo Application qua Git
- root app tự sync và tạo app con

Bài làm đã thực hiện đúng:

- stack observability và Argo Rollouts đều được khai báo thành Application trong GitOps repo
- `root-app` tự tạo các app con
- toàn bộ quá trình cài đặt đi qua Git thay vì `kubectl apply` thủ công

So với yêu cầu tối thiểu, bài làm còn mở rộng thêm:

- Grafana
- Loki
- Promtail
- Alertmanager email
- quản lý secret cho monitoring

### 4.2 Lab 2 - Tự viết app API có metrics

Slide mẫu dùng Flask để minh họa, nhưng mục tiêu cốt lõi của lab là:

- tự viết API
- có endpoint hoạt động
- có `/metrics`
- có thể build image và deploy

Bài làm thực hiện bằng `Node.js` thay vì Flask.

Điểm đúng của cách làm này:

- vẫn đáp ứng đầy đủ yêu cầu về API và metric
- dùng `prom-client` để publish metric Prometheus
- có thêm khả năng điều khiển `VERSION` và `ERROR_RATE` qua env
- image backend được đóng gói thành container và phát hành qua ECR

Vì vậy, khác framework nhưng vẫn đúng mục tiêu của bài. Thậm chí cách làm này còn sát thực tế hơn với hệ thống bạn muốn trình bày.

### 4.3 Lab 3 - Tự viết manifest và để Prometheus thấy metric

Yêu cầu:

- tự viết manifest cho app
- deploy app bằng GitOps
- để Prometheus nhìn thấy metric

Bài làm đã đáp ứng:

- `api` được khai báo bằng `Rollout` và `Service`
- Service của API có annotation scrape `/metrics`
- Prometheus đã scrape được metric `w9_api_http_requests_total`
- metric đó được dùng lại cho cả alert và analysis

Điểm mạnh là metric không chỉ “để xem”, mà trở thành nguồn dữ liệu thật cho quyết định rollout.

### 4.4 Lab 4 - Canary rollout

Yêu cầu gốc của lab 4 là canary thủ công:

- rollout lên một phần traffic
- người vận hành tự `promote` hoặc `abort`

Bài làm của bạn có hai mức:

- `frontend`: canary thủ công, phù hợp với flow lab
- `api`: canary tự động, dùng AnalysisTemplate để quyết định tiếp tục hay abort

Điều này cho thấy bài làm không chỉ dừng ở thao tác tay, mà đã tiến thêm một bước sang tự động hóa kiểm soát chất lượng.

### 4.5 Yêu cầu cuối bài

Ba ý chính trong đề cuối:

1. mọi thay đổi đi qua Git
2. có đo lường chất lượng và gửi email khi chất lượng giảm
3. canary tự động, bản lỗi quay về bản cũ

Bài làm hiện tại đã có đủ:

- thay đổi qua Git và Argo CD sync
- đo lường bằng Prometheus
- email alert qua Alertmanager
- Argo Rollouts dùng AnalysisTemplate để auto-abort khi bản mới lỗi

## 5. API được hiện thực bằng Node.js

Phần này cần nhấn mạnh rõ vì đây là lựa chọn triển khai thực tế của bài làm.

### 5.1 Vì sao dùng Node.js

- phù hợp với stack bạn chọn
- dễ viết nhanh fake API phục vụ lab
- dễ control `VERSION` và `ERROR_RATE`
- đủ để chứng minh rollout, metric, alerting, rollback

### 5.2 Các endpoint chính

API có các endpoint phục vụ trực tiếp cho bài toán:

- `/api/overview`: endpoint chính để sinh traffic và đo chất lượng
- `/health` và `/api/health`: phục vụ probe
- `/metrics`: endpoint cho Prometheus scrape

Về đóng gói và triển khai:

- backend image được build từ repo `api`
- frontend image được build từ repo `web`
- cả hai image đều được push lên `Amazon ECR`
- manifest rollout trong GitOps dùng `imagePullSecrets: ecr-registry` để pull image trong namespace `api` và `frontend`

### 5.3 Cơ chế inject lỗi

API đọc 2 biến env:

- `VERSION`: đánh dấu bản phát hành đang chạy
- `ERROR_RATE`: điều khiển xác suất trả lỗi

Ý nghĩa:

- khi `ERROR_RATE=0`, API hoạt động bình thường
- khi `ERROR_RATE` tăng, request vào `/api/overview` bắt đầu trả `500`
- nhờ đó có thể mô phỏng bản lỗi để test alerting và auto-rollback

Đây là điểm làm cho bài lab có tính thực chiến hơn, vì rollout không chỉ dựa trên cấu hình tĩnh mà có thể chứng minh bằng hành vi runtime.

## 6. Chiến lược rollout

### 6.1 Frontend

`frontend` dùng `Argo Rollout` với canary đơn giản:

- `50%`
- pause thủ công
- `100%`

Flow này phù hợp để trình bày khái niệm canary promote bằng tay.

Frontend không dùng image local trong cluster, mà dùng image trên ECR. Điều này giúp luồng của frontend khớp với mô hình phát hành chuẩn:

- sửa code ở repo `web`
- build image
- push ECR
- Argo CD sync GitOps
- rollout tạo revision mới

### 6.2 API

`api` dùng `Argo Rollout` với canary có kiểm tra chất lượng:

- `25%`
- pause `30s`
- chạy `AnalysisTemplate`
- `50%`
- pause `30s`
- chạy `AnalysisTemplate`
- `100%`

Nhờ vậy, bản mới không được lên thẳng `100%` ngay, mà phải vượt qua các checkpoint về chất lượng.

## 7. SLO, metric và alerting

### 7.1 SLI

SLI của bài làm là chất lượng của endpoint `/api/overview`, được đo bằng error rate:

```text
error_rate = 5xx_requests / total_requests
```

Nguồn dữ liệu là metric:

- `w9_api_http_requests_total`

Metric này được tách theo:

- method
- route
- status_code

### 7.2 SLO

SLO hiện tại của bài làm là:

- giữ `error rate <= 5%` cho endpoint `/api/overview`
- được tính trên cửa sổ `2 phút`

Nói cách khác:

- success rate mục tiêu là `>= 95%`
- nếu error rate vượt `5%` thì chất lượng không đạt SLO

Điểm quan trọng là:

- SLO này không chỉ viết trong README
- nó được mã hóa thành công thức thật trong Prometheus rule và AnalysisTemplate

### 7.3 Alerting policy

Alert rule hiện tại:

- alert name: `ApiHighErrorRate`
- query theo error rate của `/api/overview`
- ngưỡng: `> 0.05`
- `for: 10s`

Khi vi phạm ngưỡng:

- Prometheus chuyển alert sang `Firing`
- Alertmanager gửi email qua Gmail SMTP

Điều này đã được chứng minh bằng mail thật nhận trong inbox.

## 8. AnalysisTemplate và auto-abort

`AnalysisTemplate` là phần quan trọng nhất để biến canary tay thành canary tự động.

### 8.1 Logic đánh giá

Rollout gọi Prometheus query để lấy error rate của `/api/overview`.

Nếu:

- có dữ liệu
- và `error_rate <= 0.05`

thì measurement pass.

Nếu:

- `error_rate > 0.05`

thì measurement fail.

Khi fail vượt `failureLimit`, rollout bị abort và quay về stable revision trước đó.

### 8.2 Điểm kỹ thuật quan trọng đã xử lý

Ban đầu, logic cho phép:

- `result=[]` cũng được tính là success

Điều này gây pass nhầm khi Prometheus chưa có data tại đúng thời điểm measurement chạy.

Sau đó rule đã được siết lại:

- chỉ pass khi `len(result) > 0`
- tức là phải có dữ liệu thật thì mới đánh giá là đạt chất lượng

Chỉnh sửa này làm cho cơ chế auto-abort phản ánh đúng trạng thái thực tế của bản canary.

## 9. Những gì bài làm đã chứng minh được

Qua quá trình triển khai và evidence runtime, bài làm đã chứng minh được:

- GitOps repo là nguồn sự thật
- `frontend` và `api` đều được phát hành bằng image trên Amazon ECR
- Argo CD quản lý toàn bộ stack theo App of Apps
- frontend rollout được theo canary
- API rollout có thể lên version mới, có thể lỗi, và có thể bị abort
- Prometheus scrape được metric thật từ API
- SLO đã được hiện thực bằng công thức error rate
- Alertmanager gửi email thật khi vi phạm ngưỡng
- runtime có thể xác thực bằng `VERSION` và `ERROR_RATE`

Điểm mạnh nhất của bài làm là không chỉ dừng ở YAML, mà có bằng chứng runtime:

- rollout status
- analysis result
- metric values
- response thực tế từ API
- email alert trong inbox

## 10. Kết luận

Bài làm này đáp ứng trọn vẹn tinh thần của đề:

- phát hành qua GitOps
- có đo lường chất lượng thật
- có alert email
- có canary rollout
- có tự bảo vệ bằng auto-abort

Việc dùng `Node.js` thay cho `Flask` không làm lệch yêu cầu. Ngược lại, nó thể hiện đúng hơn hệ thống bạn chủ động xây dựng:

- API có metric thật
- có controllable error rate
- có version marker để chứng minh runtime
- gắn trực tiếp với quy trình rollout và rollback

Tóm lại, đây là một pipeline phát hành an toàn cho `api`, trong đó Git, Argo CD, Argo Rollouts, Prometheus và Alertmanager phối hợp với nhau để đưa bản mới ra dần dần, đo chất lượng thật, cảnh báo khi chất lượng giảm, và tự quay về bản ổn định khi bản mới không đạt SLO.

## 11. Mô tả ảnh chứng minh

### image.png - Tổng quan danh sách application trong Argo CD

Ảnh này cho thấy toàn bộ các application đang được Argo CD quản lý:

- `api`
- `frontend`
- `argo-rollouts`
- `grafana`
- `loki`
- `prometheus`
- `promtail`
- `monitoring-secrets`
- `root-app`

Ý nghĩa của ảnh:

- chứng minh mô hình App of Apps đang hoạt động
- chứng minh stack observability và rollout được triển khai qua GitOps chứ không phải cài tay

![ArgoCD applications overview](<image/image.png>)

### image copy.png - Cây tài nguyên của root-app

Ảnh này mô tả `root-app` là application gốc và bên dưới là các application con.

Ý nghĩa của ảnh:

- chứng minh `root-app` thực sự tạo và quản lý các app con
- thể hiện cấu trúc điều phối trung tâm của GitOps repo

![Root app tree](<image/image copy.png>)

### image copy 7.png - Frontend rollout đã lên stable

Ảnh này thể hiện `frontend` rollout thành công:

- status `Healthy`
- `setWeight = 100`
- revision mới đã trở thành `stable`

Ý nghĩa của ảnh:

- chứng minh frontend được triển khai bằng Argo Rollouts
- chứng minh canary cho frontend có thể đi đến stable

![Frontend rollout stable](<image/image copy 7.png>)

### image copy 14.png - API rollout fail và quay về stable

Ảnh này thể hiện một lần rollout API bị fail:

- rollout `Degraded`
- metric `api-error-rate` bị fail
- revision canary bị scale down
- stable revision trước đó vẫn phục vụ traffic

Ý nghĩa của ảnh:

- chứng minh cơ chế auto-abort / rollback đang hoạt động
- đây là bằng chứng rất quan trọng cho mục tiêu “tự bảo vệ”

![API rollout degraded and rollback evidence](<image/image copy 14.png>)

## 12. Full image gallery

### image.png

![image.png](<image/image.png>)

### image copy.png

![image copy.png](<image/image copy.png>)

### image copy 2.png

![image copy 2.png](<image/image copy 2.png>)

### image copy 3.png

![image copy 3.png](<image/image copy 3.png>)

### image copy 4.png

![image copy 4.png](<image/image copy 4.png>)

### image copy 5.png

![image copy 5.png](<image/image copy 5.png>)

### image copy 6.png

![image copy 6.png](<image/image copy 6.png>)

### image copy 7.png

![image copy 7.png](<image/image copy 7.png>)

### image copy 8.png

![image copy 8.png](<image/image copy 8.png>)

### image copy 9.png

![image copy 9.png](<image/image copy 9.png>)

### image copy 10.png

![image copy 10.png](<image/image copy 10.png>)

### image copy 11.png

![image copy 11.png](<image/image copy 11.png>)

### image copy 12.png

![image copy 12.png](<image/image copy 12.png>)

### image copy 13.png

![image copy 13.png](<image/image copy 13.png>)

### image copy 14.png

![image copy 14.png](<image/image copy 14.png>)

### image copy 16.png

![image copy 16.png](<image/image copy 16.png>)

### image copy 17.png

![image copy 17.png](<image/image copy 17.png>)
