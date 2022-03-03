## I. Cấu hình máy chủ WEB với các thành phần MySQL, PHP, Nginx.
    apt update

### Cài đặt **nginx**: 

    apt install nginx

### Cài đặt **php** (version 7.4):

    add-apt-repository ppa:ondrej/php
>
    apt install php7.4-fpm php7.4-curl php7.4-mysql php7.4-xml

Gõ `php -v` để kiểm tra

![](https://i.imgur.com/6ONIAVM.png)


### Cài đặt **MySQL**:

    apt install mysql-server

Gõ `service mysql status` để kiểm tra trạng thái:

![](https://i.imgur.com/6Zvxkhl.png)

Tạo cơ sở dữ liệu cho wordpress:

    mysql -h localhost -u root -P 3306 -p
>
    CREATE DATABASE wordpress; 

Tạo user cho wordpress:

    CREATE USER wordpress@localhost IDENTIFIED BY '123456'; 
>
    GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ALTER ON wordpress.* TO wordpress@localhost; 
>
    FLUSH PRIVILEGES; 
>
    quit 

### Cài đặt **Wordpress**:

    cd /var/www/ 

Tải Wordpress từ trang chủ:

    curl -O https://wordpress.org/latest.tar.gz
>
    tar xzvf latest.tar.gz 

Hoặc git clone file đã chuẩn bị:

    git clone https://github.com/VoNgocThanhHao/wordpress.git

Sau đó di chuyển vào thư mục wordpress:

    cd wordpress 
>
    cp wp-config-sample.php wp-config.php 

Cấu hình cho wordpress:

    nano wp-config.php 

Thay đổi **db_name, db_user, db_password, db_host** lần lượt thành **wordpress, wordpress, 123456, localhost**:

    define( 'DB_NAME', 'wordpress' );
>
    define( 'DB_USER', 'wordpress' );
>
    define( 'DB_PASSWORD', '123456' );
>
    define( 'DB_HOST', 'localhost' );

Thêm `define('FS_METHOD','direct');` vào cuối file.

Thoát và lưu lại.

Cấp quyền truy cập cho thư mục:

    chmod -R 777 ./* 

Cấu hình **nginx** cho **wordpress**: 

    nano /etc/nginx/sites-enabled/default

Thay đổi:

`root /var/www/html; `

`index index.html index.htm index.nginx-debian.html; `

`server_name _; `

Thành:

    root /var/www/wordpress;

    index index.html index.htm index.php; 

    server_name 0.0.0.0; 

Xóa dấu `#` ở các dòng:

    location ~ \.php${ 

    include snippets/fastcgi-php.conf; 

    fastcgi_pass unix:/var/run/php7.4-fpm.sock; 

    }

Khởi động lại **nginx**:

    service nginx restart

Kết quả nhận được:

![](https://i.imgur.com/YtfU39W.png)

## II. Cài đặt SSL

### 1.Tạo chứng chỉ SSL

    sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt

Điền thông tin của bạn vào để tạo chứng chỉ:

    Country Name (2 letter code) [AU]:AS
    State or Province Name (full name) [Some-State]:Viet Nam
    Locality Name (eg, city) []:Vinh Long
    Organization Name (eg, company) [Internet Widgits Pty Ltd]: VLUTE
    Organizational Unit Name (eg, section) []: FIT
    Common Name (e.g. server FQDN or YOUR name) []:server_IP
    Email Address []:admin@your_domain.com

Tiếp theo:

    sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096

Đợi 1 vài phút để tạo thành công chứng chỉ.

### 2.Cấu hình Nginx để sử dụng SSL

    sudo nano /etc/nginx/snippets/self-signed.conf

Thêm vào file `self-signed.conf` các dòng:

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

Tiếp theo, bạn sẽ tạo một đoạn mã khác sẽ xác định một số cài đặt SSL. Điều này sẽ thiết lập Nginx với bộ mật mã SSL mạnh mẽ và kích hoạt một số tính năng nâng cao giúp giữ an toàn cho máy chủ của bạn.

    sudo nano /etc/nginx/snippets/ssl-params.conf

Thêm vào file `ssl-params.conf` các dòng:

    ssl_protocols TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_dhparam /etc/nginx/dhparam.pem; 
    ssl_ciphers EECDH+AESGCM:EDH+AESGCM;
    ssl_ecdh_curve secp384r1;
    ssl_session_timeout  10m;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;
    # Disable strict transport security for now. You can uncomment the following
    # line if you understand the implications.
    #add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";

Điều chỉnh cấu hình **Nginx** để sử dụng SSL:

Tạo file cấu hình backup:

    sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.bak

Chỉnh sửa file `default`:

    sudo nano /etc/nginx/sites-available/default

Bên dưới:

    server {
            listen 80 default_server;
            listen [::]:80 default_server;

Thêm vào:

            listen 443 ssl default_server;
            listen [::]:443 ssl default_server;
            include snippets/self-signed.conf;
            include snippets/ssl-params.conf;
            
Thêm các dòng bên dưới vào cuối file:

    server {
        listen 80;
        listen [::]:80;

        server_name 0.0.0.0;

        return 302 https://$server_name$request_uri;
    }

Thiết lập tường lửa cho phép truy cập **nginx** :

    sudo ufw allow 'Nginx Full'

Kiểm tra xem không có lỗi cú pháp nào trong các tệp:

    sudo nginx -t

Khởi động lại **nginx**:

    sudo systemctl restart nginx

Kiểm tra bằng cách truy cập `https://ip_máy`:

![](https://i.imgur.com/vmx1sxa.png)
