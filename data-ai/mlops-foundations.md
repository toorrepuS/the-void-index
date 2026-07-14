# Nền tảng MLOps: từ Experiment đến Inference

Status: GROWING
Last reviewed: 2026-07-14

## Vấn đề

MLOps có nhiều thuật ngữ dễ bị dùng lẫn lộn: experiment, run, training job, artifact, model, deployment, endpoint và inference. Ghi chú này xây dựng một bức tranh tổng thể để trả lời:

- Mỗi khái niệm là gì và tồn tại ở giai đoạn nào?
- Chúng liên kết với nhau ra sao trong vòng đời machine learning?
- Khi nào nên dùng real-time, batch, asynchronous hoặc serverless inference?
- MLflow hỗ trợ phần nào của quy trình MLOps?

## Ghi chú

### 1. MLOps là gì?

**MLOps (Machine Learning Operations)** là tập hợp thực hành, quy trình và công cụ giúp đưa mô hình ML từ thử nghiệm lên production một cách có thể lặp lại, kiểm soát, quan sát và duy trì.

MLOps kết nối ba vùng công việc:

```text
Machine Learning  +  Data Engineering  +  DevOps
       │                    │                 │
       └────────────── MLOps lifecycle ──────┘
```

Một vòng đời đơn giản:

```text
Data → Experiment → Training → Evaluation → Model Registry
                                              │
                                              ▼
Monitoring ← Inference ← Endpoint/Job ← Deployment
     │
     └────────────── Feedback / Retraining ──────────────┘
```

Mục tiêu không chỉ là tạo ra một model có metric tốt, mà là tạo ra một **hệ thống ML đáng tin cậy**.

### 2. Experiment và Run

#### Experiment

**Experiment** là một nhóm các lần thử có cùng mục tiêu, ví dụ: “dự đoán khách hàng rời bỏ dịch vụ”. Nó là không gian logic để tổ chức và so sánh kết quả.

#### Run

**Run** là một lần thực thi cụ thể bên trong experiment. Một run thường lưu:

- **Parameters:** learning rate, batch size, số cây, kiến trúc model.
- **Metrics:** accuracy, F1, RMSE, latency.
- **Artifacts:** model file, biểu đồ, log, feature importance.
- **Metadata:** Git commit, thời gian, người chạy, phiên bản dữ liệu.

```text
Experiment: churn-prediction
├── Run 001: XGBoost, depth=5, F1=0.82
├── Run 002: XGBoost, depth=8, F1=0.85
└── Run 003: LightGBM, leaves=31, F1=0.87  ← candidate
```

Experiment tracking giúp trả lời: **Model này đến từ đâu, được huấn luyện bằng cấu hình nào và vì sao nó được chọn?**

### 3. Training

**Training** là quá trình model học các pattern từ dữ liệu bằng cách tối ưu tham số nội tại của nó.

Một training pipeline thường gồm:

1. Đọc và kiểm tra dữ liệu.
2. Tiền xử lý hoặc tạo features.
3. Chia train/validation/test.
4. Huấn luyện model.
5. Đánh giá bằng metric và các quality gate.
6. Lưu artifacts và metadata.
7. Đăng ký model đạt yêu cầu vào model registry.

**Training job** là một lần chạy được đóng gói của quá trình trên, thường có môi trường, code, dữ liệu đầu vào và tài nguyên compute xác định.

Training cần có khả năng tái lập. Tối thiểu phải truy vết được:

```text
code version + data version + dependencies + parameters + random seed
```

### 4. Artifact

**Artifact** là file hoặc tập file được sinh ra trong một bước của ML lifecycle và cần được lưu lại.

Ví dụ:

- Model đã serialize: `model.pkl`, `model.onnx`, `model.pt`.
- Model package: `model.tar.gz`.
- Preprocessor, tokenizer, vocabulary.
- Biểu đồ đánh giá, confusion matrix, feature importance.
- Log, report hoặc file dự đoán.

Điểm quan trọng:

> Model có thể là một artifact, nhưng không phải artifact nào cũng là model.

Artifact nên được lưu trong nơi bền vững như object storage hoặc artifact store, kèm phiên bản và liên kết tới run đã tạo ra nó.

### 5. Model

Từ “model” có thể mang ba nghĩa tùy ngữ cảnh:

1. **Model algorithm/architecture:** cấu trúc học, ví dụ XGBoost hoặc Transformer.
2. **Trained model:** cấu trúc cộng với weights/parameters đã học.
3. **Deployable model package:** trained model cùng preprocessing, dependencies và inference code cần để phục vụ dự đoán.

Trong production, một model tốt cần nhiều hơn metric offline:

- Đúng định dạng input/output.
- Có thể load ổn định.
- Latency và throughput đạt yêu cầu.
- Dependencies được khóa phiên bản.
- Có version, lineage và owner.
- Được kiểm tra với dữ liệu gần production.

### 6. Model Registry

**Model Registry** là catalog quản lý các model version và vòng đời của chúng.

Registry thường lưu:

- Model name và version.
- Link tới artifact.
- Metrics và metadata.
- Trạng thái như candidate, staging, production hoặc archived.
- Approval, tags và lineage.

Registry không nhất thiết trực tiếp chạy model. Nó quản lý **model nào được phép đi tiếp**, còn deployment system chịu trách nhiệm đưa model vào môi trường phục vụ.

### 7. Deployment

**Deployment** là quá trình đưa một model version vào môi trường có thể thực hiện prediction.

Deployment thường bao gồm:

- Chuẩn bị inference image/runtime.
- Tải model artifact.
- Cấu hình compute và autoscaling.
- Thiết lập network, security và logging.
- Kiểm tra health, smoke test và rollback.
- Chọn chiến lược rollout: all-at-once, canary, blue/green hoặc shadow.

### 8. Endpoint

**Endpoint** là địa chỉ hoặc giao diện mà client gọi để yêu cầu prediction từ model đã deploy.

```text
Client → Request → Endpoint → Inference runtime → Model → Response
```

Một endpoint có thể:

- Phục vụ một hoặc nhiều model/variant.
- Chạy trên một hoặc nhiều instances.
- Có autoscaling, authentication và monitoring.
- Định tuyến traffic để A/B test hoặc canary deployment.

Endpoint không đồng nghĩa với model: **model là logic dự đoán; endpoint là lớp phục vụ model cho client**.

### 9. Inference

**Inference** là quá trình dùng model đã train để tạo prediction từ dữ liệu mới.

```text
Raw input → Preprocessing → Model prediction → Postprocessing → Output
```

Sai khác giữa training và inference:

| Training | Inference |
|---|---|
| Model học từ dữ liệu | Model áp dụng điều đã học |
| Tốn compute, có thể kéo dài | Thường tối ưu latency hoặc throughput |
| Thay đổi weights/parameters | Không thay đổi weights thông thường |
| Chạy định kỳ hoặc khi cần retrain | Chạy theo request, event hoặc lịch |

Preprocessing tại inference phải tương thích với training. Sự khác biệt giữa hai bên có thể gây **training-serving skew**.

### 10. Các kiểu inference

#### Real-time inference

Client gửi request và chờ prediction trả về ngay.

Phù hợp khi:

- Cần phản hồi trong mili giây hoặc vài giây.
- Traffic tương đối liên tục hoặc khó dự đoán.
- Ví dụ: fraud detection, recommendation, scoring khi người dùng thao tác.

Đặc trưng:

- Endpoint luôn sẵn sàng.
- Ưu tiên latency và availability.
- Có thể tốn chi phí ngay cả khi ít request.

#### Batch inference

Hệ thống xử lý một tập dữ liệu lớn theo job và ghi kết quả ra storage, không cần endpoint luôn hoạt động.

Phù hợp khi:

- Không cần kết quả ngay.
- Có hàng nghìn đến hàng triệu records.
- Xử lý theo giờ, ngày hoặc lịch định kỳ.
- Ví dụ: chấm điểm toàn bộ khách hàng mỗi đêm.

Đặc trưng:

- Ưu tiên throughput và chi phí tổng.
- Compute tồn tại trong thời gian job chạy.
- Kết quả được đọc sau khi job hoàn tất.

#### Asynchronous inference

Client gửi request, hệ thống xếp hàng xử lý và trả kết quả qua một location hoặc notification sau đó.

Phù hợp khi:

- Mỗi request mất nhiều giây hoặc phút.
- Payload lớn.
- Traffic có burst và client không cần giữ kết nối chờ.
- Ví dụ: xử lý video, tài liệu dài hoặc ảnh độ phân giải cao.

Đặc trưng:

- Có queue làm bộ đệm.
- Tách thời điểm gửi request khỏi thời điểm nhận kết quả.
- Cần quản lý job ID, retry, timeout và nơi lưu output.

#### Serverless inference

Nền tảng tự cấp phát compute khi có request và có thể scale về zero khi rảnh.

Phù hợp khi:

- Traffic thấp, gián đoạn hoặc khó dự đoán.
- Muốn giảm vận hành infrastructure.
- Có thể chấp nhận cold start.
- Ví dụ: prototype, internal tool hoặc API ít được gọi.

Đặc trưng:

- Trả tiền theo mức sử dụng thay vì giữ instance liên tục.
- Có nguy cơ cold-start latency.
- Thường bị giới hạn về tài nguyên, payload hoặc thời gian xử lý tùy nền tảng.

#### Bảng chọn nhanh

| Kiểu | Client chờ trực tiếp? | Pattern dữ liệu | Ưu tiên chính | Điểm đánh đổi |
|---|---:|---|---|---|
| Real-time | Có | Request nhỏ, liên tục | Latency | Chi phí endpoint luôn sẵn sàng |
| Batch | Không | Dataset lớn | Throughput | Kết quả không tức thời |
| Async | Không | Request lớn/chậm, có burst | Khả năng hấp thụ tải | Phức tạp ở queue và theo dõi job |
| Serverless | Có | Traffic thưa/khó đoán | Chi phí và ít vận hành | Cold start và giới hạn nền tảng |

> “Serverless” mô tả cách cấp phát compute; “real-time/async/batch” chủ yếu mô tả cách workload được gửi và nhận kết quả. Khả năng kết hợp cụ thể phụ thuộc nền tảng.

### 11. MLflow

**MLflow** là một nền tảng mã nguồn mở hỗ trợ quản lý vòng đời ML. Các thành phần thường gặp:

- **Tracking:** ghi parameters, metrics, artifacts và runs.
- **Models:** đóng gói model theo format có thể tái sử dụng qua nhiều công cụ triển khai.
- **Model Registry:** quản lý model versions, aliases, tags và vòng đời.
- **Projects:** quy ước đóng gói code để chạy lặp lại; mức độ sử dụng tùy hệ thống.

Luồng khái niệm với MLflow:

```text
Experiment
└── Run
    ├── Parameters
    ├── Metrics
    ├── Artifacts
    └── Logged Model
            │
            ▼
      Registered Model
            │
            ├── Version 1
            ├── Version 2 (alias: champion)
            └── Version 3 (alias: challenger)
```

MLflow không tự động thay thế toàn bộ MLOps platform. Ta vẫn có thể cần workflow orchestration, data validation, feature store, CI/CD, infrastructure, serving và production monitoring.

### 12. Monitoring và feedback loop

Sau deployment, cần quan sát cả hệ thống và chất lượng ML:

- **System metrics:** latency, throughput, error rate, CPU/GPU, memory.
- **Data quality:** schema, missing values, phạm vi và phân phối dữ liệu.
- **Data drift:** input production thay đổi so với dữ liệu tham chiếu.
- **Prediction drift:** phân phối prediction thay đổi.
- **Model quality:** accuracy/F1/RMSE thực tế khi ground truth xuất hiện.
- **Business metrics:** conversion, fraud loss, churn hoặc KPI liên quan.

Drift là tín hiệu cần điều tra, không tự động đồng nghĩa model đã sai. Feedback loop dùng dữ liệu và ground truth mới để đánh giá, quyết định retraining và triển khai model version tiếp theo.

### 13. Ví dụ end-to-end

Bài toán dự đoán churn:

1. Data pipeline tạo dataset phiên bản `2026-07`.
2. Các training runs thử nhiều algorithms và hyperparameters.
3. MLflow Tracking lưu parameters, metrics và artifacts.
4. Run tốt nhất tạo `model.tar.gz` cùng preprocessor.
5. Model được đăng ký thành version mới trong registry.
6. Quality gate và người có thẩm quyền phê duyệt candidate.
7. Deployment đưa model lên real-time endpoint.
8. Ứng dụng gọi endpoint để nhận churn probability.
9. Monitoring theo dõi latency, drift và chất lượng thực tế.
10. Khi chất lượng giảm hoặc có dữ liệu mới, pipeline retraining được kích hoạt.

## Điều rút ra

- **Experiment** tổ chức mục tiêu thử nghiệm; **run** là một lần thử cụ thể.
- **Training** tạo ra trained model; **artifact** là mọi đầu ra cần lưu, bao gồm model file.
- **Model Registry** quản lý danh tính, phiên bản và trạng thái model.
- **Deployment** đưa model vào runtime; **endpoint** là giao diện để client truy cập runtime đó.
- **Inference** là lúc model tạo prediction từ dữ liệu mới.
- Chọn kiểu inference dựa trên latency, payload, traffic, throughput, chi phí và cách client nhận kết quả.
- **MLflow** mạnh về tracking, model packaging và registry, nhưng chỉ là một phần của MLOps platform.
- Một hệ thống ML chưa hoàn thành khi model được deploy; monitoring và feedback loop mới giúp nó sống được trong production.

## Nguồn

- [MLflow Documentation](https://mlflow.org/docs/latest/)
- [Amazon SageMaker AI — Deploy models for inference](https://docs.aws.amazon.com/sagemaker/latest/dg/deploy-model.html)
- [Google Cloud — MLOps: Continuous delivery and automation pipelines in machine learning](https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning)
- [Microsoft Azure Architecture Center — Machine learning operations](https://learn.microsoft.com/azure/architecture/ai-ml/guide/machine-learning-operations-v2)
