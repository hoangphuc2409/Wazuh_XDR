## Triển khai DVWA và Mod Security
### 1. Cài đặt DVWA
#### 1.1 Tạo DVWA Apache website
Truy cập vào thư mục `/var/www/html` chứa mã nguồn Apache Server và clone repo DVWA vào đây.
```
cd /var/www/html
git clone https://github.com/digininja/DVWA.git
```
Thay đổi permission và khởi động `apache2` service
```
sudo chmod 777 DVWA
sudo service apache2 start
```
Truy cập `http://127.0.0.1/` trên browser để kiểm tra kết quả, nếu xuất hiện hình ảnh này thì đã cài đặt thành công.

![Alt text](Image/setup_apache2.jfif)

Copy nội dung file `config.inc.php.dist` vào file `config.inc.php`
```
cd DVWA
sudo cp config/config.inc.php.dist config/config.inc.php
```
#### 1.2 Tạo DVWA user
Truy cập vào file `config.inc.php` để thay đổi username và mật khẩu. Ở đây, ta đặt username = 'admin' và password = 'password'
```
sudo nano config/config.inc.php
```
![Alt text](Image/dvwa_user.jfif)

Khởi động `mysql` service
```
sudo service mysql start
```
Truy cập với quyền root
```
sudo mysql -u root -p
```
Thực hiện các câu lệnh sau để tạo user
```
create database dvwa;
create user 'admin'@'127.0.0.1' identified by 'password';
grant all on dvwa.* to 'admin'@'127.0.0.1';
exit
```
Vào thư mục `/etc/php/8.2/apache2` và cấu hình file `php.ini`, thay đổi `allow_url_fopen = On` và `allow_url_include = On`
```
cd /etc/php/8.2/apache2
sudo nano php.ini
```
Khởi động lại `apache2` service
```
systemctl restart apache2
```
Truy cập vào `http://127.0.0.1/DVWA` và đăng nhập bằng tài khoản đã tạo -> click `Create/Reset Database`. Cài đặt thành công sẽ có kết quả như hình bên dưới.

![Alt text](Image/DVWA_successfully.jfif)
### 2. Cấu hình ModSecurity
#### 2.1 Cài đặt ModSecurity
```
sudo apt install libapache2-mod-security2 -y
```
Sau khi đã cài đặt, bật Apache2 headers và khởi động lại Apache
```
sudo a2enmod headers
sudo systemctl restart apache2
```
Sử dụng tệp cấu hình mặc định của Mod Security
```
sudo cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf
```
Mở file `/etc/modsecurity/modsecurity.conf` và thay đổi giá trị của `SecRuleEngine` thành `On`
#### 2.2 Cài đặt The OWASP ModSecurity Core Rule Set
Xóa bộ rule hiện tại của ModSecurity
```
sudo rm -rf /usr/share/modsecurity-crs
```
Clone Core Rule Set từ github và lưu vào tệp `/usr/share/modsecurity-crs`
```
sudo git clone https://github.com/coreruleset/coreruleset /usr/share/modsecurity-crs
```
Sử dụng `crs-setup.conf.example` làm file cấu hình
```
sudo cp /usr/share/modsecurity-crs/crs-setup.conf.example /usr/share/modsecurity-crs/crs-setup.conf
sudo cp /usr/share/modsecurity-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf.example /usr/share/modsecurity-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
```
#### 2.3 Kích hoạt ModSecurity trên Apache
Chỉnh sửa file `/etc/apache2/mods-available/security2.conf` theo nội dung bên dưới:
```
<IfModule security2_module>
        SecDataDir /var/cache/modsecurity
        Include /usr/share/modsecurity-crs/crs-setup.conf
        Include /usr/share/modsecurity-crs/rules/*.conf
</IfModule>
```
Chỉnh sửa file `/etc/apache2/sites-enabled/000-default.conf` theo nội dung bên dưới:
```
<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
        SecRuleEngine On
</VirtualHost>
```
Khởi động lại Apache
```
systemctl restart apache2
```
#### 2.4 Kiểm tra hoạt động của ModSecurity
Mở trình duyệt và truy cập `http://127.0.0.1/DVWA`. Chọn mục `DVWA Security`, chỉnh độ khó thành `Low` và nhấn `Submit`

![Alt text](Image/dvwa_security.jfif)

Tiếp theo, chọn một loại tấn công bất kỳ, ví dụ ở đây là `SQL Injection` và thực hiện tấn công

![Alt text](Image/sqli.jfif)

Kết quả là tấn công đã bị Mod Security chặn

![Alt text](Image/result.jfif)
