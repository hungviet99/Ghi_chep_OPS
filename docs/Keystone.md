# Keystone

Môi trường cloud theo mô hình dịch vụ IaaS cung cấp cho người dùng khả năng truy cập vào các tài nguyên quan trọng như máy ảo, kho lưu trữ và kết nối mạng. Bảo mật vẫn luôn là vấn đề quan trọng đối với cloud, trong OPS, keystone là dịch vụ đảm nhiệm công việc bảo mật. Nó chịu trách nhiệm cung cấp các kết nối mang tính bảo mật tới các nguồn tài nguyên Cloud.

## 1. Ba chức năng chính của keystone

### 1.1. Identity (Định danh)

- Nhận diện các truy cập vào tài nguyên trên cloud
- Những mô hình OPS nhỏ, identity của user thường được lưu trữ trong database của keystone. Đối với những mô hình lớn cho doanh nghiệp thì 1 external Identity Provider thường được sử dụng

### 1.2. Authentication (Xác thực)

- Là quá trình xác thực những thông tin dùng để nhận định user
- Keystone có thể liên kết với những dịch vụ xác thực người dùng khác như LDAP hoặc AD
- Thường thì keystone sử dụng password cho việc xác thực người dùng. Đối với những phần còn lại, keystone sử dụng token
- OPS dựa nhiều vào token để xác thực và keystone chính là dịch vụ duy nhất có thể tạo ra tokens.
- Token có thể giới hạn thời gian được phép sử dụng. Khi token hết hạn thì user sẽ được cấp 1 token mới.

### 1.3. Access Management (Quản lý truy cập, quyền hạn)

- Đây là quá trình xác định những tài nguyên mà user được phép truy cập tới.
- Trong OPS, keystone kết nối user với những PJ hoặc Domain bằng cách gán role cho user vào những project hoặc domain ấy.
- Các project trong OPS như nova, cinder, ..  sẽ kiểm tra mối quan hệ giữa role và các user project và xác định giá trị của những thông tin này theo cơ chế các quy định (policy engine). Policy engine sẽ tự động kiểm tra các thông tin và xác định xem user được phép thực hiện những gì.

## 2. Các khái niệm của keystone

### 2.1. Projects

Trong keystone, project có thể hiểu là 1 nhóm các tài nguyên mà chỉ có 1 số các user mới có thể truy cập và hoàn toàn tách biệt với các nhóm khác.

Ban đầu được gọi là tenants, sau đó được đổi tên thành projects.

Mục đích cơ bản nhất của keystone là nới đăng ký cho các projects và xác định ai được phép truy cập project đó.

Project không sử dụng user hay groups mà users và group được cấp quyền truy cập tới project 

### 2.2. Domains

Domain sử dụng để cô lập danh sách các project và users.

Domain là tập hợp các users, groups và projects. Nó cho phép người dùng chia nguồn tài nguyên cho từng tổ chức sử dụng mà không phải lo xung đột hay nhầm lẫn.


## 3. Identity

Identity trong môi trường Cloud có thể đến từ các vị trí khác nhau, bao gồm SQL, LDAP và Federated Identity Providers

### 3.1. SQL

Keystone có tùy chọn cho phép lưu trữ actor trong SQL

Các database được hỗ trợ: MySQL, PostgreSQL, DB2

Keystone sẽ lưu trữ các thông tin như: name, password, description

### 3.2. LDAP

Keystone sử dụng tùy chọn sử dụng LDAP để khôi phục và lưu trữ các actor

Keystone sẽ truy cập đến LDAP giống như các ứng dụng sử dụng LDAP

Cài đặt kết nối đến LDAP phải được chỉ rõ trong file cấu hình keystone,

LDAP chỉ nên thực hiện đọc như là tìm kiếm user và group và xác thực 

### 3.3. Identity provider

Keystone sẽ sử dụng bên thứ 3 để xác thực, nó có thể là những phần mềm sử dụng các backends (LADP, AD, MongoDB) hoặc mạng xã hội như google, facebook, twitter

**Ưu điểm:**

- Có thể tận dụng phần mềm và cơ sở hạ tầng cũ để xác thực cũng như lấy thông tin của users
- Tách biệt keystone và nhiệm vụ định danh, xác thực thông tin
- Keystone không thể xem mật khẩu, mọi thứ đều không còn liên quan tới keystone

**Nhược điểm:**

- Phức tạp nhất về việc setup trong các loại

### 3.4. Multiple backends

- Mỗi một domain có thể có 1 identity source (backend) khác nhau

- Domain mặc định thường sử dụng SQL backend bởi nó được dùng để lưu các host service accounts. Service accounts là các tài khoản được dùng bởi các dịch vụ OPS khác nhau để tương tác với keystone

## 4. Authentication

Có rất nhiều cách để xác thực với keystone service trong đó 2 phương thức được dùng nhiều nhất là password và token.

### 4.1. Password

Phương thức phổ biến nhất  cho user hoặc service để xác thực là dùng password.


