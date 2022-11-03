# Kubernetes Autoscaling Demo

## Đề bài
+ Cho một service với 1 API xử lí một công việc rất tốn tài nguyên CPU.
+ Mỗi 1 instance của service chỉ được sử dụng tối đa là 0.1 core, và 100MB ram
+ Với mỗi request từ client, server sẽ tốn tầm 0.05-0.1s để xử lí.
+ Tốc độ response của service có thể tăng lên 0.5-1s nếu lượng request tăng đột biến do tốc độ xử lí bị giới hạn bởi số core.
+ Số lượng request đến server sẽ là tăng giảm không biết trước theo khoảng thời gian khác nhau (để giả lập trường hợp thực tế).
=> Sử dụng kubernetes để áp dụng autoscaling cho service

## Tài nguyên cho trước
### Docker image demo-server
+ Tạo file Dockerfile cho file ```demo-server.go``` (đã có)
+ Run câu lệnh build image: ```docker build . -t demo-server```
+ Test thử container: ```docker run -d --name demo-server -p 3000:3000 demo-server```
+ Xóa container: ```docker rm -f demo-server```

### Resource 
+ Kubernetes single node cluster trên Docker desktop
+ Giới hạn phần cứng Docker desktop là 4 core cpu, 4gb ram, 32gb disk.

## Phân tích và triển khai
+ Số lượng request đến không thể dự đoán trước và tài nguyên/thời gian xử lí request biết trước nên ta scale theo chiều ngang bằng HPA (Horizontal autoscaling)
+ Vì service thiên về sử dụng cpu, memory và i/o không đáng kể, nếu ta sẽ sử dụng scale theo chỉ số CPU.
+ Trước hết phải cài đặt metric-server để HPA có thể đọc được metric của pods.
+ Tạo ra môt Deployment có port là 3000, sử dụng image demo-server và cấu hình các resource theo đề bài
+ Tạo ra Service NodePort để expose ra ngoài ở cổng 30001, 
+ Ta tạo ra một HPA, đọc thông tin metric theo resource cpu, đặt ngưỡng bắt đầu scale là 50% cpu của pod.

### Setup metrics server 
+ Chạy lệnh: ```kubectl apply -f demo-metric.yaml```
+ Xác nhận metric server đã chạy: ```kubectl get deployment metrics-server -n kube-system```
+ Kiểm tra metric server: ```kubectl top pod```
### Setup scale bằng HPA theo CPU usage
+ Tạo file tên demo-deployment.yaml (đã có)
+ Tạo file tên demo-hpa.yaml (đã có)
+ Tạo Deployment + Service NodePort: ```kubectl apply -f demo-deployment.yaml```
+ Tạo HPA: ```kubectl apply -f demo-hpa.yaml```
+ Kiểm tra quá trình scale up: ```kubectl get hpa,deployment``` hoặc ```watch -n 1 kubectl get hpa,deployment``` nếu cài watch
+ Chạy file demo-client để tạo ra nhiều requests đến cluster, có thể thay đổi số threadNumber ở client để giả lập lượng request lớn hơn
+ Kiểm tra thông tin chi tiết của HPA: ```kubectl describe hpa demo``` hoặc ```watch -n 1 kubectl describe hpa demo``` nếu cài watch
+ Xóa Deployment: ```kubectl delete deployment demo```
+ Xóa Service: ```kubectl delete service demo```
+ Xóa HPA: ```kubectl delete hpa demo```

### Giải thích quá trình autoscaling
+ Khi bắt đầu tạo ra deployment, service và hpa ta sẽ thấy 3 container tạo ra ở 3 pod.
+ Sau đó một thời gian, HPA scale lượng pod xuống 1 do các pod đang sử dụng mức cpu rất thấp (scale down)
+ Khi ta chạy demo-client lượng lớn request gọi đến cluster, HPA tính toán lại và tăng số pod lên (scale up)
+ Quá trình tính toán số desired pod sẽ diễn ra sau mỗi 60s và sau đó sẽ được scale (tùy config)

## Kết luận
+ Sử dụng Autoscaling với Kubernetes tăng tốc độ xử lí cho ứng dụng đồng thời giữ mức tiêu thụ tài nguyên một cách hợp lí, qua đó tối ưu lượng tài nguyên được sử dụng nhưng vẫn đảm bảo khả năng phục vụ tốt nhất cho service trên lượng phần cứng cho trước.
+ Sử dụng Autoscaling với Kubernetes vẫn có nhược điểm rằng toàn bộ quá trình này được ảo hóa trong Kubernetes, néu có vấn đề liên quan đên lỗi do Kube thì sẽ ảnh hưởng đến toàn bộ service, nên phải cẩn trọng trong việc chọn version và thử nghiệm kĩ lưỡng