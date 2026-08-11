# Trained Models

Các file model không được commit trực tiếp vì dung lượng lớn.

Sau khi chạy notebook cuối, hệ thống tạo:

- `best_network_anomaly_model.keras`
- `preprocessing.joblib`
- `experiment_config.json`

## Tái tạo model

1. Mở notebook cuối.
2. Cấu hình đường dẫn dataset.
3. Chạy toàn bộ notebook.
4. Model được lưu trong thư mục artifact của Colab.

## Sử dụng model

Model phải được sử dụng cùng scaler, median, label encoder và danh sách
feature trong `preprocessing.joblib`.