# CSE-CIC-IDS2018 Network Anomaly Detection

Đồ án cuối kỳ môn Deep Learning / Học máy và Khai phá dữ liệu. Dự án xây dựng pipeline phát hiện bất thường mạng trên bộ dữ liệu CSE-CIC-IDS2018 và so sánh bốn kiến trúc Deep Learning trên cùng một tập dữ liệu:

- MLP: baseline cho dữ liệu dạng bảng.
- CNN1D: học mẫu cục bộ giữa các đặc trưng.
- BiLSTM: mô hình hóa vector đặc trưng như một chuỗi.
- Transformer Encoder: dùng self-attention để học quan hệ toàn cục giữa các đặc trưng.

Notebook final đã được đặt tại [notebooks/final/CICIDS2018_Network_Anomaly_Detection_Final.ipynb](notebooks/final/CICIDS2018_Network_Anomaly_Detection_Final.ipynb).

## 1. Thành viên nhóm

| STT | Họ và tên | MSSV | Phần phụ trách |
|---:|---|---|---|
| 1 | Vũ Tuấn Đạt           | BIT240055 | Đọc CICIDS2018; làm sạch; cân bằng; chia Train/Validation/Test 70/15/15 |
| 2 | Vũ Minh Đức           | BIT240066 | Xây dựng, huấn luyện CNN1D |
| 3 | Hoàng Phúc Vinh       | BIT240255 | Chuẩn bị sequence input; huấn luyện BiLSTM |
| 4 | Nguyễn Ngọc Tuấn Linh | BIT240140 | Hoàn thiện Transformer; trực quan hóa attention và live demo |
| 5 | Mai Đức Minh          | BIT240151 | Xây dựng mô hình mạng nơ-ron truyền thống |


## 2. Bài toán

Mục tiêu của hệ thống là phân loại một network flow thành:

- BENIGN: lưu lượng bình thường.
- ATTACK: lưu lượng có dấu hiệu tấn công.

Notebook cũng hỗ trợ cấu hình multiclass để phân loại từng loại tấn công. Cấu hình được sử dụng và có output đầy đủ trong notebook là binary, vì đây là cấu hình ổn định nhất cho live demo và phù hợp với yêu cầu xây dựng một pipeline phân loại hoàn chỉnh.

## 3. Dataset và thiết kế thực nghiệm

Nguồn dữ liệu là CSE-CIC-IDS2018, gồm các thống kê network flow do CICFlowMeter-V3 trích xuất, được phân bố trong nhiều file CSV tương ứng với các ngày/kịch bản tấn công.

Cấu hình mặc định trong notebook:

| Thành phần | Thiết lập |
|---|---|
| Số file CSV đọc | 10 |
| Số dòng lấy tối đa mỗi file | 40.000 |
| Số dòng sau khi đọc mẫu | 400.000 |
| Giới hạn mỗi lớp sau sampling | 50.000 |
| Dataset cuối dùng huấn luyện | 100.000 flow, cân bằng 50.000 ATTACK và 50.000 BENIGN |
| Số đặc trưng sau preprocessing | 71 |
| Chia dữ liệu | 70% train, 15% validation, 15% test |
| Random seed | 42 |
| Task mặc định | Binary classification |

Sampling được sử dụng để giới hạn bộ nhớ và thời gian chạy trên Google Colab, không phải tạo dữ liệu giả. Các cột định danh như Flow ID, địa chỉ IP, timestamp và tên file nguồn được loại khỏi feature để giảm nguy cơ mô hình ghi nhớ danh tính hoặc ngày thu thập dữ liệu.

## 4. Pipeline xử lý

    CSV files
      -> Đọc theo chunk và lấy mẫu từng file
      -> Chuẩn hóa tên cột và nhãn
      -> Gộp dữ liệu, giới hạn số mẫu mỗi lớp
      -> Stratified split: train / validation / test
      -> Loại cột định danh
      -> Chuyển feature sang numeric
      -> inf -> NaN
      -> Median imputation fit trên train
      -> RobustScaler fit trên train
      -> Clip giá trị trong [-10, 10]
      -> Huấn luyện 4 mô hình
      -> Đánh giá trên test
      -> Confusion matrix, ROC/PR, failure analysis
      -> Lưu model, preprocessing và kết quả

Preprocessing chỉ được fit trên tập train, sau đó mới transform validation và test. Đây là bước quan trọng để tránh data leakage.

## 5. Các kiến trúc

### MLP

MLP gồm Dense 128, Batch Normalization, ReLU, Dropout 0,30, Dense 64, Batch Normalization, ReLU, Dropout 0,20 và lớp phân loại softmax.

### CNN1D

CNN1D xem 71 feature như một chuỗi một chiều, gồm Conv1D 64 filters, MaxPooling1D, Conv1D 128 filters, GlobalAveragePooling1D, Dropout và Dense classifier.

### BiLSTM

BiLSTM gồm hai tầng Bidirectional LSTM, lần lượt 48 và 24 units, kết hợp Dropout và Dense classifier.

### Transformer

Transformer chiếu mỗi scalar feature sang vector d_model=64, thêm learnable positional embedding, MultiHeadAttention với 4 heads, residual connection, Layer Normalization, feed-forward network và GlobalAveragePooling1D.

Scaled dot-product attention được trình bày trong notebook:

    Attention(Q, K, V) = softmax(QK^T / sqrt(d_k))V

Notebook tạo riêng AttentionInspector để trực quan hóa mức attention trung bình của các đặc trưng. Attention giúp diễn giải hành vi mô hình, nhưng không nên được xem là bằng chứng nhân quả tuyệt đối.

## 6. Huấn luyện và đánh giá

Các mô hình dùng cùng:

- 15 epochs tối đa.
- Batch size 512.
- Adam, learning rate 0,001.
- Sparse categorical cross-entropy.
- ReduceLROnPlateau.
- Early stopping và khôi phục best checkpoint theo validation loss.
- Class weight được tính từ tập train.
- Cùng train/validation/test split để so sánh công bằng.

Metric chính là Macro-F1 vì dataset IDS có thể mất cân bằng. Accuracy, Balanced Accuracy, Weighted-F1, ROC-AUC và PR-AUC được báo cáo bổ sung.

## 7. Kết quả thực nghiệm

Kết quả dưới đây là output đã lưu trong notebook final, với cấu hình binary và 100.000 flow sau sampling.

| Model | Params | Best epoch | Test loss | Accuracy | Balanced Acc. | Macro-F1 | ROC-AUC | PR-AUC |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| MLP | 18.178 | 14 | 0,1015 | 0,9679 | 0,9679 | 0,9679 | 0,9859 | 0,9900 |
| BiLSTM | 45.698 | 15 | 0,1102 | 0,9669 | 0,9669 | 0,9668 | 0,9828 | 0,9882 |
| Transformer | 42.434 | 15 | 0,1187 | 0,9649 | 0,9649 | 0,9649 | 0,9825 | 0,9879 |
| CNN1D | 34.050 | 13 | 0,1316 | 0,9612 | 0,9612 | 0,9612 | 0,9808 | 0,9866 |

Theo Macro-F1 trên test, MLP đạt kết quả cao nhất trong lần chạy hiện tại. Classification report của MLP:

| Class | Precision | Recall | F1-score |
|---|---:|---:|---:|
| ATTACK | 0,9966 | 0,9391 | 0,9670 |
| BENIGN | 0,9424 | 0,9968 | 0,9688 |
| Macro average | 0,9695 | 0,9679 | 0,9679 |

Lưu ý phương pháp: notebook dùng test set để tổng kết và sắp xếp mô hình theo Macro-F1 sau khi các cấu hình đã cố định. Khi viết báo cáo theo hướng chặt chẽ hơn, nên chọn model bằng validation Macro-F1 rồi chỉ dùng test một lần để báo cáo cuối cùng.

## 8. Loss, confusion matrix và prediction visualization

Notebook cung cấp đầy đủ các thành phần theo yêu cầu:

- Train loss và validation loss của từng mô hình.
- Train accuracy và validation accuracy của từng mô hình.
- Biểu đồ so sánh các metric trên test set.
- Normalized confusion matrix cho MLP, CNN1D, BiLSTM và Transformer.
- Bảng prediction gồm nhãn thật, nhãn dự đoán, confidence và trạng thái đúng/sai.
- Classification report của model tốt nhất.
- ROC curve và Precision-Recall curve cho bài toán binary.

## 9. Failure analysis

Notebook phân tích các mẫu dự đoán sai theo confidence, gồm:

- Vị trí mẫu trong test set.
- Nhãn thật.
- Nhãn dự đoán.
- Confidence của dự đoán.
- File nguồn của flow.
- Các cặp nhãn bị nhầm nhiều nhất.

Trong output hiện tại, MLP dự đoán sai 481/15.000 mẫu, tương đương 3,21%. MLP có attack recall 0,9391 và attack precision 0,9966: hệ thống bỏ sót một phần ATTACK nhưng số cảnh báo ATTACK nhầm là tương đối thấp.

Khi trình bày, cần đưa vào slide:

1. Confusion matrix của model tốt nhất.
2. ROC/PR curve.
3. Một số mẫu sai có confidence cao.
4. Nhận xét về trade-off giữa precision và recall.
5. Ảnh hưởng của sampling và việc loại các cột định danh.

## 10. Live demo

Notebook có cell live demo thực hiện các bước:

1. Chọn một flow chưa dùng để train từ test set.
2. Chạy lại đúng preprocessing đã fit trên train.
3. Đưa flow vào model tốt nhất.
4. Trả về nhãn dự đoán, confidence và xác suất của từng lớp.

Khi bảo vệ, nên chuẩn bị một bản dữ liệu nhỏ trên Google Drive để live demo không phụ thuộc việc tải lại dataset lớn. Không sử dụng video quay sẵn thay cho việc chạy trực tiếp.

## 11. Model artifacts

Notebook tạo thư mục artifact và file ZIP gồm:

- best_network_anomaly_model.keras
- preprocessing.joblib
- model_comparison.csv
- experiment_config.json

preprocessing.joblib chứa feature columns, train medians, scaler, label encoder, task, model được chọn, loại input và clip range. Model chỉ có thể tái sử dụng đúng khi đi kèm bundle preprocessing tương ứng.

## 12. Cài đặt và chạy

Khuyến nghị sử dụng Google Colab có GPU.

1. Clone repository hoặc upload notebook lên Colab.
2. Đặt file ZIP dataset tại:

    /content/drive/MyDrive/CICIDS2018.zip

3. Mở notebook final.
4. Chạy Runtime > Run all.
5. Kiểm tra các cell đọc dữ liệu, bảng results_df, loss curves, confusion matrix, failure analysis và live demo.
6. Tải file ZIP artifact sau khi cell lưu model chạy thành công.

Các thư viện chính:

    pip install "pandas>=2.0" "scikit-learn>=1.3" seaborn joblib
    pip install tensorflow matplotlib numpy

Không commit dataset lớn, file kaggle.json, model binary hoặc artifact cá nhân lên GitHub.

## 13. Cấu trúc repository

    .
    ├── data/
    │   ├── README.md
    │   └── models/README.md
    ├── docs/
    │   ├── experiment_notes.md
    │   └── member_contributions.md
    ├── notebooks/
    │   ├── final/
    │   │   └── CICIDS2018_Network_Anomaly_Detection_Final.ipynb
    │   └── weekly/
    ├── results/
    │   └── notes.md
    └── README.md

Các notebook weekly là quá trình phát triển trước đó. Kết quả chính và nội dung bảo vệ cuối kỳ phải lấy từ notebook final.

## 14. Checklist đối chiếu yêu cầu

- [x] Xác định bài toán phân loại BENIGN/ATTACK.
- [x] Mô tả dataset, số lượng, đặc trưng và phân bố lớp.
- [x] Có pipeline preprocessing và train/validation/test.
- [x] Có MLP baseline.
- [x] Có CNN1D.
- [x] Có BiLSTM.
- [x] Có Transformer và self-attention.
- [x] Có loss train/validation.
- [x] Có bảng so sánh metric.
- [x] Có confusion matrix.
- [x] Có prediction visualization.
- [x] Có failure analysis.
- [x] Có attention visualization.
- [x] Có live demo.
- [x] Có lưu model và preprocessing artifact.
- [ ] Điền thông tin thành viên và đóng góp thực tế.
- [ ] Chạy Runtime > Run all trên runtime sạch trước khi nộp.
- [ ] Chọn model theo validation nếu muốn quy trình đánh giá nghiêm ngặt hơn.
- [ ] Hoàn thiện slide tối đa 10 trang và ghi nguồn/công cụ AI đã sử dụng.

## 15. Hướng cải tiến

- Chọn best model theo validation Macro-F1, không dùng test để chọn model.
- Chạy thêm nhiều seed hoặc cross-validation để kiểm tra độ ổn định.
- Đánh giá theo split theo ngày để đo khả năng tổng quát sang thời điểm/kịch bản mới.
- So sánh thêm threshold hoặc class weighting theo mục tiêu giảm false negative ATTACK.
- Giữ nguyên toàn bộ phân bố tự nhiên của dataset trong một thí nghiệm bổ sung, bên cạnh bản sampling phục vụ Colab.
- Bổ sung permutation importance hoặc SHAP để hỗ trợ giải thích feature ngoài attention.
- Đóng gói live demo thành hàm nhận CSV/network flow mới từ người dùng.

## 16. Tài liệu tham khảo

- CSE-CIC-IDS2018: https://www.unb.ca/cic/datasets/ids-2018.html
- TensorFlow Documentation: https://www.tensorflow.org/
- Keras Documentation: https://keras.io/
- Scikit-learn Documentation: https://scikit-learn.org/stable/

