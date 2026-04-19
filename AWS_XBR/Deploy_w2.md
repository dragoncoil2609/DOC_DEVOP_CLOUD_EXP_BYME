# IAM user

1. tạo IAM user

User name: Nhập tên user.

Tích chọn Provide user access to the AWS Management Console.

Chọn I want to create an IAM user.

Console password: Chọn Custom password.

Tích chọn Users must create a new password at next sign-in - Recommended.

Click Next.

2. set quyền

Chọn ô Attach policies directly.

Ở thanh tìm kiếm bên dưới, tìm quyền muốn set trường hợp này muốn set quyền admin nên là gõ AdministratorAccess và tích chọn.

Click Next.

3. tạo mã MFA

Góc trên cùng bên phải (chỗ tên user), click vào -> Chọn Security credentials.

Tìm mục Multi-factor authentication (MFA) -> Click Assign MFA device.

Device name: Gõ tên nào gợi nhớ phân biệt.

Chọn Authenticator app -> Click Next.

Dùng app Google Authenticator hoặc Authy trên điện thoại quét mã QR và nhập 2 mã liên tiếp để xác nhận.

# Tạo Roles cho ứng dụng ECS
1. tạo roles ecsTaskExecutionRole

Ở menu bên trái (Side Bar) của IAM, chọn Roles -> Click Create role.
Trusted entity type: Chọn AWS service.

Use case: Tìm ở ô dropdown chữ Elastic Container Service, sau đó chọn Elastic Container Service Task (Lưu ý bỏng tay: chọn đúng cái có chữ Task).

Click Next.

add quyền 
Gõ AmazonECSTaskExecutionRolePolicy -> Tích chọn.

Gõ SecretsManagerReadWrite -> Tích chọn.

tên role để ecsTaskExecutionRole

Click Create role.

ecsTaskExecutionRole: Khi cấu hình chạy App, máy chủ ảo của AWS (ECS Agent) cần quyền để đi lấy cái Docker image từ ECR về, và cần quyền mở Secrets Manager để lấy mật khẩu Database nhét vào container. role này cung cấp quyền đó.

2. tạo role ecsTaskRole

khác ở chỗ chúng ta phải tạo giấy phép cho role này 

Tại giao diện IAM, chọn Policies ở menu bên trái -> Click nút Create policy.

Tại mục Policy editor, chọn tab JSON.

Xóa hết nội dung cũ và dán đoạn mã dưới đây vào. Đoạn mã này cho phép ứng dụng của  có quyền xem, tải lên và xóa file trong S3:

JSON
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:DeleteObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "*"
        }
    ]
}
(Lưu ý bỏng tay: Trong môi trường thực tế, ta nên thay dấu * bằng tên chính xác của Bucket để bảo mật hơn, nhưng hiện tại ta chưa tạo Bucket nên để * để linh hoạt).
  Click Next.
  Policy name: Đặt tên là EcomS3AccessPolicy.
  Click Create policy.

  tiếp theo tương tự tạo role ở trên nhưng chúng ta sẽ gắn quyềnn vừa tạo vào EcomS3AccessPolicy

ecsTaskRole: Đây là quyền cấp cho chính mã nguồn ứng dụng (code) của , role này để gắn thêm quyền "Được phép ghi file lên S3".

# Tạo VPC

1. 
 VPCs -> Click Create VPC

Chọn VPC only (chúng ta tạo tay từng bước để hiểu rõ luồng đi, thay vì dùng VPC and more).

Name tag: Gõ VPC-Lab.

IPv4 CIDR block: Chọn IPv4 CIDR và gõ 10.0.0.0/16.

Các phần khác để mặc định -> Click Create VPC.

2. tạo Internet Gateway sẽ là cổng chính kết nối internet

Internet gateways -> Click Create internet gateway.

Name tag: Gõ IGW-Lab -> Click Create internet gateway.

khi tạo thành công Actions -> Attach to VPC.

Chọn VPC-Lab từ danh sách thả xuống -> Click Attach internet gateway.

3. tạo subnet cho layer public

Khu vực này sẽ dùng để chứa NAT Gateway và Load Balancer (ALB) sau này.

Subnets -> Click Create subnet.

VPC ID: Chọn VPC-Lab.

Subnet 1:

Subnet name: Gõ Nat-Subnet-1.

Availability Zone: Chọn us-east-1a (hoặc vùng a tương ứng với region ).

IPv4 CIDR block: Gõ 10.0.0.0/24.

Kéo xuống dưới cùng click Add new subnet để tạo luôn Subnet thứ 2:

Subnet name: Gõ Nat-Subnet-2.

Availability Zone: Chọn us-east-1b (chọn vùng b để dự phòng nếu vùng a sập).

IPv4 CIDR block: Gõ 10.0.1.0/24.

Click Create subnet.

4. tạo routable cho 2 public subnet

 chọn Route tables -> Click Create route table.

Name: Gõ IGW-Route-Table.

VPC: Chọn VPC-Lab -> Click Create route table.

Trỏ đường ra Internet:

Vẫn ở màn hình của Route table vừa tạo, chọn tab Routes ở dưới -> Click Edit routes.

Click Add route.

Destination: Gõ 0.0.0.0/0 (đại diện cho toàn bộ Internet).

Target: Chọn Internet Gateway -> Chọn IGW-Lab.

Click Save changes.

Chuyển sang tab Subnet associations -> Click Edit subnet associations.

Tích chọn 2 subnet là Nat-Subnet-1 và Nat-Subnet-2.

Click Save associations.

VPC: Là hàng rào bao quanh toàn bộ hệ thống E-commerce .

IGW: Là cánh cổng duy nhất nối từ bên trong hàng rào ra thế giới Internet bên ngoài.

Public Subnet & Route Table: Khi gắn luật 0.0.0.0/0 -> IGW cho Nat-Subnet-1 và Nat-Subnet-2, chúng chính thức trở thành "Khu vực Public". Bất cứ dịch vụ nào đặt ở đây (như Load Balancer sắp tới) đều có thể giao tiếp trực tiếp với người dùng trên Internet.

5. tạo 2 nat cho 2 subnet puclic

Name: Gõ Nat-Lab-1.

Subnet: Chọn Nat-Subnet-1 (Đây chính là lý do chúng ta phải tạo Subnet trước!).

Elastic IP allocation ID: Click vào nút Allocate Elastic IP để AWS cấp cho NAT một địa chỉ IP tĩnh công cộng.

Click Create NAT gateway.

nat 2 tương tự 

6. tạo layer private cho app 
chọn Subnets -> Click Create subnet. 

chọn đúng vpc

Tạo App Subnet 1:

Subnet name: Gõ App-Subnet-1.

Availability Zone: Chọn us-east-1a.

IPv4 CIDR block: Gõ 10.0.10.0/22.

Kéo xuống click Add new subnet để tạo App Subnet 2:

Subnet name: Gõ App-Subnet-2.

Availability Zone: Chọn us-east-1b.

IPv4 CIDR block: Gõ 10.0.14.0/22.

tạo routable cho 2 subnet trên 

Tạo Route Table cho App 1:

Name: Gõ App-1-Route-Table -> Chọn VPC VPC-Lab -> Click Create.

Tại tab Routes, click Edit routes -> Add route.

Destination: 0.0.0.0/0 -> Target: Chọn NAT Gateway rồi chọn Nat-Lab-1 -> Save changes.

Chuyển sang tab Subnet associations -> Edit subnet associations -> Tích chọn App-Subnet-1 -> Save.

Tạo Route Table cho App 2:

Lặp lại y hệt bước trên: Tạo App-2-Route-Table, trỏ route 0.0.0.0/0 về đích là Nat-Lab-2, và gắn (associate) nó với App-Subnet-2.
6. tạo layer private cho db

Vào Subnets -> Create subnet -> Chọn VPC-Lab.

Tạo DB Subnet 1: Name DB-Subnet-1, AZ us-east-1a, CIDR 10.0.2.0/24.

Tạo DB Subnet 2: Kéo xuống Add new subnet, Name DB-Subnet-2, AZ us-east-1b, CIDR 10.0.3.0/24.

Click Create subnet.

lưu ý cho Database: * Vào Route tables -> Tạo DB-Route-Table cho VPC-Lab.

Tại tab Subnet associations, gắn nó với cả DB-Subnet-1 và DB-Subnet-2.

(Không thêm bất kỳ route nào ra Internet ở đây).
 
7. tạo VPC Endpoint

chọn Endpoints -> Click Create endpoint.

Name tag: Gõ EndPoint-Lab.

Service category: Chọn EC2 Instance Connect Endpoint (Theo tài liệu của ).

VPC: Chọn VPC-Lab.

Subnets: Chọn subnet ứng dụng (như App-Subnet-1 hoặc App-Subnet-2).

Kéo xuống dưới cùng và click Create endpoint.

Tổng kết công dụng của các bước vừa làm:

NAT Gateway: Giúp các container Backend (như Nodejs/NestJS) gọi được API bên ngoài (ví dụ API thanh toán Momo/ZaloPay, hoặc kéo Docker image) nhưng hacker bên ngoài không thể chủ động gọi ngược vào container.

App Subnet & Route Table: Thiết lập mạng lưới để ép toàn bộ luồng mạng của Backend phải đi qua chốt chặn NAT.

DB Subnet: Đây là nơi chứa Database. Bằng việc không có Route Table hướng ra NAT hay IGW, Database bị cô lập hoàn toàn khỏi Internet, đảm bảo dữ liệu E-commerce an toàn tuyệt đối.

# Tạo, cấu hình Security Groups

Đầu tiên, tạo quy tắc cho ALB. Vì ALB nằm ở Public Subnet, nó có nhiệm vụ nhận request từ mọi nơi trên Internet.

Click nút Create security group.

Basic details:

Security group name: Gõ ALB-SG.

Description: Gõ Cho phep truy cap Internet.

VPC: Bắt buộc chọn đúng VPC-Lab của .

Inbound rules (Luật vào): Click Add rule 2 lần để thêm 2 luật sau:

Rule 1: Type chọn HTTP, Source chọn Anywhere-IPv4 (0.0.0.0/0).

Rule 2: Type chọn HTTPS, Source chọn Anywhere-IPv4 (0.0.0.0/0). (E-commerce bắt buộc phải có HTTPS để bảo mật thanh toán).

Kéo xuống dưới cùng click Create security group.

SG CHO APP

Đây là tường lửa cho các container chứa code Backend của . Backend này chạy ở port 3000 (theo tài liệu). Tuyệt đối không mở port này ra Internet, mà chỉ mở cho ALB.

Quay lại danh sách, click Create security group.

Basic details:

Security group name: Gõ App-ECS-SG.

Description: Gõ Chi nhan traffic tu ALB.

VPC: Chọn VPC-Lab.

Inbound rules (Luật vào): Click Add rule.

Type: Chọn Custom TCP.

Port range: Gõ 3000.

Source: Chọn Custom, click vào ô tìm kiếm bên cạnh và chọn cái tên ALB-SG mà  vừa tạo ở bước 1.

Click Create security group.

SG CHO DB

Đây là lớp bảo vệ cuối cùng cho RDS. Nó chỉ nhận lệnh từ Backend.

Tiếp tục click Create security group.

Basic details:

Security group name: Gõ DB-SG.

Description: Gõ Chi nhan traffic tu App Backend.

VPC: Chọn VPC-Lab.

Inbound rules (Luật vào): Click Add rule.

Type: Chọn MySQL (Hệ thống sẽ tự điền Port là 3306).

Source: Chọn Custom, click vào ô tìm kiếm và chọn cái tên App-ECS-SG vừa tạo ở bước 2.

Click Create security group.

Nhờ cách cấu hình "móc xích" (Source của rule này là Security Group của layer kia), hệ thống của  đã sở hữu khả năng phòng thủ chiều sâu (Defense in Depth). Dù hacker có biết IP nội bộ của Database, họ cũng không thể ping hay gửi request vào được, vì bảo vệ ở DB-SG sẽ kiểm tra và thấy request đó không xuất phát từ App-ECS-SG.



# tạo cấu hình DB
 1. tạo subet gr
 AWS RDS yêu cầu  phải nhóm các Subnet lại để nó biết được phép đặt Database ở những vùng nào.

RDS, chọn Subnet groups -> Click Create DB subnet group.

Name: Gõ DB-Subnet-Group.

Description: Gõ Nhom Subnet cho Database E-com.

VPC: Chọn đúng VPC-Lab của .

Add subnets:

Availability Zones: Chọn 2 vùng là us-east-1a và us-east-1b.

Subnets: Tích chọn 2 cái subnet tương ứng với DB-Subnet-1 và DB-Subnet-2 ( có thể nhìn vào dải IP 10.0.2.0/24 và 10.0.3.0/24 để chọn cho chuẩn).

Click Create.

2. tạo Database Chính (Master DB)
Đây sẽ là nơi lưu trữ thông tin đơn hàng, khách hàng, sản phẩm.

Ở menu bên trái, chọn Databases -> Click Create database.

Choose a database creation method: Chọn Standard create (Full configuration).

Engine options: Chọn MySQL (hoặc Aurora MySQL nếu dự án E-com cần tốc độ siêu cao).

Templates: Chọn Production hoặc Dev/Test tùy vào nhu cầu hiện tại.

Settings:

DB instance identifier: Gõ database-lab.

Master username: Gõ root (hoặc admin).


Connectivity (Rất quan trọng):

Virtual private cloud (VPC): Chọn VPC-Lab.

DB Subnet Group: Chọn DB-Subnet-Group vừa tạo ở trên.

Public access: Bắt buộc chọn No (Để không ai từ Internet chọc thẳng được vào DB).

VPC security group (firewall): Chọn Choose existing. Bỏ dấu 'X' ở cái default đi, và tìm chọn cái DB-SG mà chúng ta vừa tạo ở Step 2.

Availability Zone: Chọn us-east-1a.

Kéo xuống dưới cùng click Create database.

# Tạo, CẤU HÌNH ALB
1. Tạo target gr
VÀO EC 2 Ở menu bên trái, chọn Target Groups -> Click Create target group.

Basic configuration:

Target type: Bắt buộc chọn IP addresses (Vì chúng ta sẽ chạy container ECS Fargate, mỗi container sẽ có một IP riêng biệt).

Target group name: Gõ tg-items (hoặc tên tùy dự án).

Protocol: Chọn HTTP.

Port: Gõ 3000 (đây là port mà code Backend đang chạy).

VPC: Chọn đúng VPC-Lab.

Health checks (Khám sức khỏe):

Health check path: Gõ một đường dẫn API luôn trả về mã HTTP 200 (ví dụ: /api/v1/items/product hoặc /health). ALB sẽ liên tục gọi vào đường link này, nếu container trả lời "Tôi ổn", ALB mới gửi khách hàng vào.

Click Next -> Ở bước Register targets, tạm thời để trống (vì ta chưa chạy container nào) -> Click Create target group.

2. Tạo "Người điều phối" (Load Balancer)
Ở menu bên trái, chọn Load Balancers -> Click Create load balancer.

Load balancer types: Tìm ô Application Load Balancer và click Create.

Basic configuration:

Load balancer name: Gõ domain-alb.

Chọn Internet-facing và IPv4.

Network mapping (Rất quan trọng):

VPC: Chọn VPC-Lab.

Mappings: Tích chọn 2 vùng us-east-1a và us-east-1b. Tại ô Subnet xổ xuống, BẮT BUỘC CHỌN 2 PUBLIC SUBNET là Nat-Subnet-1 và Nat-Subnet-2. (Vì ALB phải nằm ở khu vực có Internet để đón khách).

Security groups:

Xóa cái default đi và chọn cái tường lửa ALB-SG (Cho phép cổng 80/443) mà chúng ta đã tạo ở Step 2.

Listeners and routing:

Protocol: HTTP - Port: 80.

Default action: Chọn Return fixed response.

Response code: 404.

Response body: Gõ Not Found. (Cấu hình này là một trick bảo mật: Nếu hacker quét lung tung hoặc gõ sai đường dẫn API, ALB sẽ chặn lại và ném ra chữ "Not Found" ngay lập tức, không cho phép chạm tới code Backend)

Kéo xuống dưới cùng click Create load balancer

3. phân luồng alb Quay lại danh sách Load Balancers, click vào domain-alb vừa tạo.

Chuyển sang tab Listeners and rules, tích chọn cái Listener HTTP:80.

Click vào liên kết của listener đó để vào chi tiết, sau đó click Manage rules -> Add rule (hoặc nút + Add Rule trên giao diện mới của AWS).

Name / Priority: Gõ 1. Click Next.

Add condition (Điều kiện):

Rule condition types: Chọn Path.

Gõ đường dẫn API: /api/v1/items/* (dấu sao đại diện cho mọi thứ đằng sau). Click Confirm -> Next.

Add action (Hành động):

Routing actions: Chọn Forward to target groups.

Target group: Chọn cái tg-items vừa tạo ở bước 1. Click Next -> Click Create hoặc Add rule.

# khởi tạo redis
1. Tạo (Security Group)
Redis cần một quy tắc: Chỉ nhận truy cập từ các container Backend, chặn toàn bộ phần còn lại.

Vào dịch vụ VPC -> Chọn Security groups (ở menu bên trái) -> Click Create security group.

Basic details:

Security group name: Gõ Redis-SG.

Description: Gõ Cho phep App Backend truy cap Redis.

VPC: Bắt buộc chọn VPC-Lab.

Inbound rules (Luật vào): Click Add rule.

Type: Chọn Custom TCP.

Port range: Gõ 6379 (Đây là cổng tiêu chuẩn mặc định của Redis).

Source: Chọn Custom -> Click vào ô tìm kiếm bên cạnh và chọn App-ECS-SG (Cái tường lửa của ECS Fargate mà ta tạo ở Step 2).

Kéo xuống dưới cùng click Create security group.

2. tạo Subnet Group
Giống như Aurora Database, AWS ElastiCache cần biết nó được phép nằm ở những vùng nào.

Chuyển sang dịch vụ ElastiCache (gõ trên thanh tìm kiếm trên cùng).

Ở menu bên trái, chọn Subnet groups -> Click Create subnet group.

Subnet group details:

Name: Gõ Redis-Subnet-Group.

Description: Gõ Khu vuc dat Redis Cache.

VPC ID: Chọn VPC-Lab.

Add subnets:

Availability Zones: Tích chọn 2 vùng là us-east-1a và us-east-1b.

Subnet ID: Tích chọn đúng 2 cái DB-Subnet-1 và DB-Subnet-2 (Dựa vào dải IP 10.0.2.0/24 và 10.0.3.0/24 để chọn cho chuẩn xác).

Click Create.

3. Khởi tạo Cụm Redis (Redis Cluster)


ở menu bên trái của ElastiCache, chọn Redis clusters -> Click Create Redis cluster.

Chọn Design your own cache và Cluster cache.

Cluster mode: Chọn Disabled (Đối với hệ thống E-com cơ bản, Disabled là đủ để tạo 1 Master và 1 Replica dự phòng).

Cluster info:

Name: Gõ redis-ecom-lab.

Node type: Nhấn vào ô tìm kiếm, chọn một loại nhỏ gọn để tiết kiệm chi phí như cache.t3.micro hoặc cache.t4g.micro.

Number of replicas: Gõ 1.

Connectivity & Security:

Subnet group: Chọn cái Redis-Subnet-Group vừa tạo ở bước 2.

Security groups: Click Manage -> Bỏ chọn cái mặc định (default) đi, và tích chọn cái Redis-SG vừa tạo ở bước 1.

Kéo xuống dưới cùng click Create.

# deploy ecs
1. tạo cluter
Ở menu bên trái, chọn Clusters -> Click Create cluster.

Cluster name: Gõ Lab-Cluster.

Infrastructure: AWS sẽ mặc định chọn AWS Fargate (Serverl không cần quản lý máy chủ vật lý).

Click Create.
tạo task defition 

Ở menu bên trái, chọn Task definitions -> Click nút Create new task definition.

Task definition family: Gõ tên-task.

2. Cấu hình Hạ tầng & Phân quyền (Infrastructure)
Launch type: AWS mặc định chọn AWS Fargate (Serverless container).

Task size: Chọn CPU và Memory phù hợp (ví dụ: .5 vCPU và 1 GB).

Task role (Quan trọng): Chọn ecsTaskRole. Đây là cái role chúng ta đã gắn quyền S3 ở Step 0, giúp code sau này upload được ảnh.

Task execution role: Chọn ecsTaskExecutionRole. Nhờ role này, ECS mới có quyền kéo image và mở "két sắt" Secrets Manager.

3. Cấu hình Container (App Backend)
Name: Gõ tên-container.

Image URI: Gõ địa chỉ image trên Docker Hub/ ecr. (Nếu image là private, phần này sẽ phức tạp hơn chút, nhưng nếu public thì gõ thẳng vào là xong).

Container port: Gõ 3000 (Port mà code Backend đang chạy).

4. Bơm "Nhiên liệu" cho App (Environment variables & Secrets)
Cuộn xuống phần Environment. Đây là bước ghép nối mọi thứ lại với nhau.

A. Thêm Biến môi trường thông thường (Environment variables):
Click Add environment variable và điền lần lượt:

thêm biến cần điêfn tuỳ dự án

B. Thêm Biến bảo mật từ Két sắt (Secrets):
Tìm phần Secrets (hoặc chọn type là ValueFrom tùy giao diện) -> Click Add secret:

Key: RDS_USERNAME | ValueFrom: Dán chuỗi ARN của Secret vào, thêm :RDS_USERNAME:: ở cuối. (Ví dụ: arn:aws:secretsmanager:...:RDS_USERNAME::)

Key: RDS_PASSWORD | ValueFrom: Dán chuỗi ARN vào, thêm :RDS_PASSWORD:: ở cuối.

5. Hoàn tất
Kéo xuống dưới cùng và click Create.

# các bước để có thể truy cập vào đb 
Bước 1: Khởi tạo EC2 Instance (Làm Bastion Host)
Truy cập giao diện EC2, bấm Launch Instance.

OS (AMI): Chọn hệ điều hành quen thuộc, ví dụ như Ubuntu.

Network settings:

VPC: Bắt buộc phải chọn cùng VPC với con RDS Database.

Subnet: Phải chọn một Public Subnet (để con EC2 này có đường ra Internet để có thể SSH vào nó).

Auto-assign public IP: Chọn Enable .

Security Group: Tạo mới 1 SG (ví dụ: EC2-Bastion-SG). Mở port 22 (SSH) với Source là My IP .

Tải file key .pem về và Launch Instance.

Bước 2: SSH vào EC2 và cài đặt DB Client
Mở Terminal (hoặc CMD/PowerShell) và SSH vào con EC2 bằng lệnh:

Bash
ssh -i "duong-dan-toi-file-key.pem" ubuntu@IP_PUBLIC_CUA_EC2
Khi đã vào được bên trong màn hình của Ubuntu
lệnh cài
Nếu DB là MySQL:

Bash
sudo apt update
sudo apt install mysql-client -y

Lệnh kết nối MySQL:

Bash
mysql -h THAY_BANG_RDS_ENDPOINT -u THAY_BANG_USERNAME -p


