# Cài đặt Openstack train 

## 1. IP Planning

**Controller**

*IP: 192.168.10.50*  
*Hostname: controller*  

**Compute**

*IP: 192.168.10.51*
*Hostname: compute1*

## 2. Cài đặt

## 2.1. Cài đặt Openstack train trên Controller

*Thực hiện trên controller*

### 2.1.1. Cài đặt môi trường

- Tắt firewall và selinux

```
yum update -y

yum install epel-release -y

yum update -y
```
```
yum install -y wget byobu git vim

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl disable firewalld
systemctl stop firewalld
setenforce 0
```

- Set host name

```
echo "192.168.10.50 controller" >> /etc/hosts
echo "192.168.10.51 compute1" >> /etc/hosts
```

### 2.1.2. Cài đặt openstack 

#### Cài đặt packet openstack

```
yum -y install centos-release-openstack-train
yum -y upgrade
yum -y install crudini wget vim
yum -y install python-openstackclient openstack-selinux
yum -y update
```

#### Cài đặt và cấu hình memcached

```
yum -y install memcached python-memcached
```
```
cp /etc/sysconfig/memcached /etc/sysconfig/memcached.bak
```
```
sed -i "s/-l 127.0.0.1,::1/-l 127.0.0.1,::1,192.168.10.50/g" /etc/sysconfig/memcached
systemctl enable memcached.service
systemctl restart memcached.service
```
>Cấu hình dịch vụ để sử dụng địa chỉ IP quản lý của node controller. Điều này là để cho phép truy cập bởi các node khác thông qua dải mạng VLAN MGNT (dải mạng quản lý)

#### Cài đặt và cấu hinh mariadb

```
yum -y install mariadb mariadb-server python2-PyMySQL
```
```
cat <<EOF> /etc/my.cnf.d/openstack.cnf
[mysqld]
bind-address = 192.168.10.50

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
EOF
```
```
systemctl enable mariadb.service
systemctl start mariadb.service
```

Đặt mật khẩu cho mysql bằng lệnh sau:

```
mysql_secure_installation
```
> Setpass mysql là Viethung1008@

```
mysql -u root -pViethung1008@
GRANT ALL PRIVILEGES ON *.* TO 'root'@'192.168.10.50' IDENTIFIED BY 'Viethung1008@' WITH GRANT OPTION;
FLUSH PRIVILEGES;
DROP USER 'root'@'::1';
exit
```

#### Cài đặt và cấu hình rabbitmq

```
yum -y install rabbitmq-server
```
```
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
```

Khai báo pluggin cho rabbitmq

```
rabbitmq-plugins enable rabbitmq_management
curl -O http://localhost:15672/cli/rabbitmqadmin
chmod a+x rabbitmqadmin
mv rabbitmqadmin /usr/sbin/
```
```
rabbitmqctl add_user openstack Viethung1008
```

Cấp quyền cho openstack truy cập rabbitmq

```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

Set tag adminnistrator

```
rabbitmqctl set_user_tags openstack administrator
```

Show danh sách user

```
rabbitmqadmin list users
```

#### Cài đặt etcd

```
yum -y install etcd
```

Backup cấu hình etcd

```
cp /etc/etcd/etcd.conf /etc/etcd/etcd.conf.bak
```

```
sed -i '/ETCD_DATA_DIR=/cETCD_DATA_DIR="/var/lib/etcd/default.etcd"' /etc/etcd/etcd.conf
sed -i '/ETCD_LISTEN_PEER_URLS=/cETCD_LISTEN_PEER_URLS="http://192.168.10.50:2380"' /etc/etcd/etcd.conf
sed -i '/ETCD_LISTEN_CLIENT_URLS=/cETCD_LISTEN_CLIENT_URLS="http://192.168.10.50:2379"' /etc/etcd/etcd.conf
sed -i '/ETCD_NAME=/cETCD_NAME="controller"' /etc/etcd/etcd.conf
sed -i '/ETCD_INITIAL_ADVERTISE_PEER_URLS=/cETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.10.50:2380"' /etc/etcd/etcd.conf
sed -i '/ETCD_ADVERTISE_CLIENT_URLS=/cETCD_ADVERTISE_CLIENT_URLS="http://192.168.10.50:2379"' /etc/etcd/etcd.conf
sed -i '/ETCD_INITIAL_CLUSTER=/cETCD_INITIAL_CLUSTER="controller=http://192.168.10.50:2380"' /etc/etcd/etcd.conf
sed -i '/ETCD_INITIAL_CLUSTER_TOKEN=/cETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"' /etc/etcd/etcd.conf
sed -i '/ETCD_INITIAL_CLUSTER_STATE=/cETCD_INITIAL_CLUSTER_STATE="new"' /etc/etcd/etcd.conf
```

```
systemctl enable etcd
systemctl restart etcd
systemctl status etcd
```

#### Cài đặt Keystone

Tạo database, user và phân quyền cho keystone

```
mysql -u root -pViethung1008@
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'192.168.10.50' IDENTIFIED BY 'Viethung1008';
FLUSH PRIVILEGES;
exit
```

```
yum -y install openstack-keystone httpd mod_wsgi
```
```
cp /etc/keystone/keystone.conf /etc/keystone/keystone.conf.bak
crudini --set /etc/keystone/keystone.conf database connection mysql+pymysql://keystone:Viethung1008@192.168.10.50/keystone
crudini --set /etc/keystone/keystone.conf token provider fernet
chown root:keystone /etc/keystone/keystone.conf
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

Thiết lập boottrap cho keystone

```
keystone-manage bootstrap --bootstrap-password Viethung1008 \
--bootstrap-admin-url http://192.168.10.50:5000/v3/ \
--bootstrap-internal-url http://192.168.10.50:5000/v3/ \
--bootstrap-public-url http://192.168.10.50:5000/v3/ \
--bootstrap-region-id RegionOne
```

Cấu hình httpd

```
sed -i "s/#ServerName www.example.com:80/ServerName controller/g" /etc/httpd/conf/httpd.conf
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

```
systemctl enable httpd.service
systemctl start httpd.service
```

Tạo file biên môi trường cho keystone

```
cat << EOF > /root/admin-openrc
export OS_USERNAME=admin
export OS_PASSWORD=Viethung1008
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://192.168.10.50:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF
```

Thực thi với quyền admin và kiểm tra hoạt động của keystone

```
source /root/admin-openrc
openstack token issue
```

Khai báo user `demo`, project `demo`

```
openstack project create service --domain default --description "Service Project" 
openstack project create demo --domain default --description "Demo Project" 
openstack user create demo --domain default --password Viethung1008
openstack role create user
openstack role add --project demo --user demo user
```

#### Cài đặt và cấu hình Glance

Tạo database cho glance

```
mysql -uroot -pViethung1008@
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'192.168.10.50' IDENTIFIED BY 'Viethung1008';
FLUSH PRIVILEGES;
exit
```

```
source /root/admin-openrc
```

Tạo user, project cho glance

```
openstack user create  glance --domain default --password Viethung1008
openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image" image
openstack endpoint create --region RegionOne image public http://192.168.10.50:9292
openstack endpoint create --region RegionOne image internal http://192.168.10.50:9292
openstack endpoint create --region RegionOne image admin http://192.168.10.50:9292
```

```
yum install -y openstack-glance
yum install -y MySQL-python
yum install -y python-devel
```

```
cp /etc/glance/glance-api.conf /etc/glance/glance-api.conf.bak
```

Cấu hình glance

```
crudini --set /etc/glance/glance-api.conf database connection  mysql+pymysql://glance:Viethung1008@192.168.10.50/glance
crudini --set /etc/glance/glance-api.conf keystone_authtoken www_authenticate_uri http://192.168.10.50:5000
crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_url  http://192.168.10.50:5000
crudini --set /etc/glance/glance-api.conf keystone_authtoken memcached_servers 192.168.10.50:11211
crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_type password 
crudini --set /etc/glance/glance-api.conf keystone_authtoken project_domain_name Default
crudini --set /etc/glance/glance-api.conf keystone_authtoken user_domain_name Default
crudini --set /etc/glance/glance-api.conf keystone_authtoken project_name service
crudini --set /etc/glance/glance-api.conf keystone_authtoken username glance
crudini --set /etc/glance/glance-api.conf keystone_authtoken password Viethung1008
crudini --set /etc/glance/glance-api.conf paste_deploy flavor keystone
crudini --set /etc/glance/glance-api.conf glance_store stores file,http
crudini --set /etc/glance/glance-api.conf glance_store default_store file
crudini --set /etc/glance/glance-api.conf glance_store filesystem_store_datadir /var/lib/glance/images/
```

```
su -s /bin/sh -c "glance-manage db_sync" glance
```

```
systemctl enable openstack-glance-api.service
systemctl start openstack-glance-api.service
```

Tải 1 image mẫu

```
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
openstack image create "cirros" --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --public
```

Kiểm tra lại image đã up chưa 

```
openstack image list
```

#### Cài đặt và cấu hình placement

Tạo database

```
mysql -uroot -pViethung1008@
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'192.168.10.50' IDENTIFIED BY 'Viethung1008';
FLUSH PRIVILEGES;
exit
```

```
source /root/admin-openrc
```

```
openstack user create  placement --domain default --password Viethung1008
openstack role add --project service --user placement admin
openstack service create --name placement --description "Placement API" placement
openstack endpoint create --region RegionOne placement public http://192.168.10.50:8778
openstack endpoint create --region RegionOne placement internal http://192.168.10.50:8778
openstack endpoint create --region RegionOne placement admin http://192.168.10.50:8778
```

Install placement

```
yum install -y openstack-placement-api
cp /etc/placement/placement.conf /etc/placement/placement.conf.bak
```

```
crudini --set  /etc/placement/placement.conf placement_database connection mysql+pymysql://placement:Viethung1008@192.168.10.50/placement
crudini --set  /etc/placement/placement.conf api auth_strategy keystone
crudini --set  /etc/placement/placement.conf keystone_authtoken auth_url  http://192.168.10.50:5000/v3
crudini --set  /etc/placement/placement.conf keystone_authtoken memcached_servers 192.168.10.50:11211
crudini --set  /etc/placement/placement.conf keystone_authtoken auth_type password
crudini --set  /etc/placement/placement.conf keystone_authtoken project_domain_name Default
crudini --set  /etc/placement/placement.conf keystone_authtoken user_domain_name Default
crudini --set  /etc/placement/placement.conf keystone_authtoken project_name service
crudini --set  /etc/placement/placement.conf keystone_authtoken username placement
crudini --set  /etc/placement/placement.conf keystone_authtoken password Viethung1008
```

Khai báo phân quyền cho placement

```
cat <<EOF>> /etc/httpd/conf.d/00-nova-placement-api.conf
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
EOF
```

#### Cài đặt và cấu hình Nova

Tạo database

```
mysql -uroot -pViethung1008@

CREATE DATABASE nova_api;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'192.168.10.50' IDENTIFIED BY 'Viethung1008';

CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'192.168.10.50' IDENTIFIED BY 'Viethung1008';

CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'192.168.10.50' IDENTIFIED BY 'Viethung1008';
FLUSH PRIVILEGES;

exit
```

```
openstack user create nova --domain default --password Viethung1008
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute
openstack endpoint create --region RegionOne compute public http://192.168.10.50:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://192.168.10.50:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://192.168.10.50:8774/v2.1
```

Install nova

```
yum install -y openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler
cp /etc/nova/nova.conf /etc/nova/nova.conf.bak
```

```
crudini --set /etc/nova/nova.conf DEFAULT my_ip 192.168.10.50
crudini --set /etc/nova/nova.conf DEFAULT use_neutron true
crudini --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver
crudini --set /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
crudini --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:Viethung1008@192.168.10.50:5672/

crudini --set /etc/nova/nova.conf api_database connection mysql+pymysql://nova:Viethung1008@192.168.10.50/nova_api
crudini --set /etc/nova/nova.conf database connection mysql+pymysql://nova:Viethung1008@192.168.10.50/nova
crudini --set /etc/nova/nova.conf api connection  mysql+pymysql://nova:Viethung1008@192.168.10.50/nova

crudini --set /etc/nova/nova.conf keystone_authtoken www_authenticate_uri http://192.168.10.50:5000/
crudini --set /etc/nova/nova.conf keystone_authtoken auth_url http://192.168.10.50:5000/
crudini --set /etc/nova/nova.conf keystone_authtoken memcached_servers 192.168.10.50:11211
crudini --set /etc/nova/nova.conf keystone_authtoken auth_type password
crudini --set /etc/nova/nova.conf keystone_authtoken project_domain_name Default
crudini --set /etc/nova/nova.conf keystone_authtoken user_domain_name Default
crudini --set /etc/nova/nova.conf keystone_authtoken project_name service
crudini --set /etc/nova/nova.conf keystone_authtoken username nova
crudini --set /etc/nova/nova.conf keystone_authtoken password Viethung1008

crudini --set /etc/nova/nova.conf vnc enabled true 
crudini --set /etc/nova/nova.conf vnc server_listen \$my_ip
crudini --set /etc/nova/nova.conf vnc server_proxyclient_address \$my_ip

crudini --set /etc/nova/nova.conf glance api_servers http://192.168.10.50:9292

crudini --set /etc/nova/nova.conf oslo_concurrency lock_path /var/lib/nova/tmp

crudini --set /etc/nova/nova.conf placement region_name RegionOne
crudini --set /etc/nova/nova.conf placement project_domain_name Default
crudini --set /etc/nova/nova.conf placement project_name service
crudini --set /etc/nova/nova.conf placement auth_type password
crudini --set /etc/nova/nova.conf placement user_domain_name Default
crudini --set /etc/nova/nova.conf placement auth_url http://192.168.10.50:5000/v3
crudini --set /etc/nova/nova.conf placement username placement
crudini --set /etc/nova/nova.conf placement password Viethung1008

crudini --set /etc/nova/nova.conf scheduler discover_hosts_in_cells_interval 300

crudini --set /etc/nova/nova.conf neutron url http://192.168.10.50:9696
crudini --set /etc/nova/nova.conf neutron auth_url http://192.168.10.50:5000
crudini --set /etc/nova/nova.conf neutron auth_type password
crudini --set /etc/nova/nova.conf neutron project_domain_name Default
crudini --set /etc/nova/nova.conf neutron user_domain_name Default
crudini --set /etc/nova/nova.conf neutron project_name service
crudini --set /etc/nova/nova.conf neutron username neutron
crudini --set /etc/nova/nova.conf neutron password Viethung1008
crudini --set /etc/nova/nova.conf neutron service_metadata_proxy True
crudini --set /etc/nova/nova.conf neutron metadata_proxy_shared_secret Viethung1008
```

```
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
su -s /bin/sh -c "nova-manage db sync" nova
```

Kiểm tra Cell đã được đăng ký hay chưa

```
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

Khởi động các dịch vụ của nova

```
systemctl enable \
openstack-nova-api.service \
openstack-nova-scheduler.service \
openstack-nova-conductor.service \
openstack-nova-novncproxy.service
```

```
systemctl start \
openstack-nova-api.service \
openstack-nova-scheduler.service \
openstack-nova-conductor.service \
openstack-nova-novncproxy.service
```

Kiểm tra dịch vụ nova đã chạy hay chưa

```
openstack compute service list
```

Restart node controller

```
reboot
```

## 2.2. Cài đặt Openstack train trên Compute

*Thực hiện trên compute*

### 2.2.1. Cài đặt môi trường

- Tắt firewall và selinux

```
yum update -y

yum install epel-release -y

yum update -y
```
```
yum install -y wget byobu git vim

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl disable firewalld
systemctl stop firewalld
setenforce 0
```

- Set host name

```
echo "192.168.10.50 controller" >> /etc/hosts
echo "192.168.10.51 compute1" >> /etc/hosts
```

### 2.2.2. Cài đặt openstack

#### Cài đặt Openstack

```
yum -y install centos-release-openstack-train
yum -y upgrade
yum -y install crudini wget vim
yum -y install python-openstackclient openstack-selinux python2-PyMySQL
yum -y update
```

#### Cài đặt Nova compute

```
yum install -y python-openstackclient openstack-selinux openstack-utils
yum install -y openstack-nova-compute
```

```
cp  /etc/nova/nova.conf  /etc/nova/nova.conf.bak
```

```
crudini --set /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
crudini --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:Viethung1008@192.168.10.50
crudini --set /etc/nova/nova.conf DEFAULT my_ip 192.168.10.51
crudini --set /etc/nova/nova.conf DEFAULT use_neutron true
crudini --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver

crudini --set /etc/nova/nova.conf api_database connection mysql+pymysql://nova:Viethung1008@192.168.10.50/nova_api

crudini --set /etc/nova/nova.conf database connection mysql+pymysql://nova:Viethung1008@192.168.10.50/nova

crudini --set /etc/nova/nova.conf api auth_strategy keystone

crudini --set /etc/nova/nova.conf keystone_authtoken www_authenticate_uri http://192.168.10.50:5000/
crudini --set /etc/nova/nova.conf keystone_authtoken auth_url http://192.168.10.50:5000/
crudini --set /etc/nova/nova.conf keystone_authtoken memcached_servers 192.168.10.50:11211
crudini --set /etc/nova/nova.conf keystone_authtoken auth_type password
crudini --set /etc/nova/nova.conf keystone_authtoken project_domain_name Default
crudini --set /etc/nova/nova.conf keystone_authtoken user_domain_name Default
crudini --set /etc/nova/nova.conf keystone_authtoken project_name service
crudini --set /etc/nova/nova.conf keystone_authtoken username nova
crudini --set /etc/nova/nova.conf keystone_authtoken password Viethung1008

crudini --set /etc/nova/nova.conf vnc enabled true
crudini --set /etc/nova/nova.conf vnc server_listen 0.0.0.0
crudini --set /etc/nova/nova.conf vnc server_proxyclient_address \$my_ip
crudini --set /etc/nova/nova.conf vnc novncproxy_base_url http://192.168.10.50:6080/vnc_auto.html

crudini --set /etc/nova/nova.conf glance api_servers http://192.168.10.50:9292

crudini --set /etc/nova/nova.conf oslo_concurrency lock_path /var/lib/nova/tmp

crudini --set /etc/nova/nova.conf placement region_name RegionOne
crudini --set /etc/nova/nova.conf placement project_domain_name Default
crudini --set /etc/nova/nova.conf placement project_name service
crudini --set /etc/nova/nova.conf placement auth_type password
crudini --set /etc/nova/nova.conf placement user_domain_name Default
crudini --set /etc/nova/nova.conf placement auth_url http://192.168.10.50:5000/v3
crudini --set /etc/nova/nova.conf placement username placement
crudini --set /etc/nova/nova.conf placement password Viethung1008

crudini --set /etc/nova/nova.conf libvirt virt_type  $(count=$(egrep -c '(vmx|svm)' /proc/cpuinfo); if [ $count -eq 0 ];then   echo "qemu"; else   echo "kvm"; fi)

```

Khởi động libvirt và nova compute

```
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
```

## 2.3. Cấu hình sau khi cài nova compute

*Thực hiện trên controller*

#### Thêm node compute vào hệ thống

```
source /root/admin-openrc
openstack compute service list --service nova-compute
```

Thêm node compute vào cell

```
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```

#### Cài đặt Neutron

Tạo database

```
mysql -uroot -pViethung1008@
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'192.168.10.50' IDENTIFIED BY 'Viethung1008';
FLUSH PRIVILEGES;
exit
```

```
source /root/admin-openrc
openstack user create neutron --domain default --password Viethung1008
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Compute" network
openstack endpoint create --region RegionOne network public http://192.168.10.50:9696
openstack endpoint create --region RegionOne network internal http://192.168.10.50:9696
openstack endpoint create --region RegionOne network admin http://192.168.10.50:9696
```

```
yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables
```

```
cp  /etc/neutron/neutron.conf  /etc/neutron/neutron.conf.bak
cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.bak
cp  /etc/neutron/plugins/ml2/linuxbridge_agent.ini  /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak 
cp /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.bak
cp /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini.bak
```

```
crudini --set  /etc/neutron/neutron.conf DEFAULT core_plugin ml2
crudini --set  /etc/neutron/neutron.conf DEFAULT service_plugins router
crudini --set  /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:Viethung1008@192.168.10.50
crudini --set  /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
crudini --set  /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes True
crudini --set  /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes True 

crudini --set  /etc/neutron/neutron.conf database connection  mysql+pymysql://neutron:Viethung1008@192.168.10.50/neutron

crudini --set  /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://192.168.10.50:5000
crudini --set  /etc/neutron/neutron.conf keystone_authtoken auth_url http://192.168.10.50:5000
crudini --set  /etc/neutron/neutron.conf keystone_authtoken memcached_servers 192.168.10.50:11211
crudini --set  /etc/neutron/neutron.conf keystone_authtoken auth_type password
crudini --set  /etc/neutron/neutron.conf keystone_authtoken project_domain_name default
crudini --set  /etc/neutron/neutron.conf keystone_authtoken user_domain_name default
crudini --set  /etc/neutron/neutron.conf keystone_authtoken project_name service
crudini --set  /etc/neutron/neutron.conf keystone_authtoken username neutron
crudini --set  /etc/neutron/neutron.conf keystone_authtoken password Viethung1008

crudini --set /etc/neutron/neutron.conf nova auth_url http://192.168.10.50:5000
crudini --set /etc/neutron/neutron.conf nova auth_type password
crudini --set /etc/neutron/neutron.conf nova project_domain_name Default
crudini --set /etc/neutron/neutron.conf nova user_domain_name Default
crudini --set /etc/neutron/neutron.conf nova region_name RegionOne
crudini --set /etc/neutron/neutron.conf nova project_name service
crudini --set /etc/neutron/neutron.conf nova username nova
crudini --set /etc/neutron/neutron.conf nova password Viethung1008

crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp

crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers flat,vlan,vxlan
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types vxlan
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers linuxbridge
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers port_security          
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks provider
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vni_ranges 1:1000        
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_ipset True


crudini --set  /etc/neutron/plugins/ml2/linuxbridge_agent.ini linux_bridge physical_interface_mappings provider:eth1
crudini --set  /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan enable_vxlan True
crudini --set  /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan local_ip $(ip addr show dev eth2 scope global | grep "inet " | sed -e 's#.*inet ##g' -e 's#/.*##g')
crudini --set  /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup enable_security_group True
crudini --set  /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

```
echo 'net.bridge.bridge-nf-call-iptables = 1' >> /etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-ip6tables = 1' >> /etc/sysctl.conf
modprobe br_netfilter
/sbin/sysctl -p
```

```
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
--config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

Khởi động neutron

```
systemctl enable neutron-server.service \
neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
neutron-metadata-agent.service

systemctl start neutron-server.service \
neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
neutron-metadata-agent.service
```

Kiểm tra network

```
openstack network agent list
```

## 2.4. Cài neutron trên node compute

*Thực hiện trên compute*

```
crudini --set /etc/nova/nova.conf neutron url http://192.168.10.50:9696
crudini --set /etc/nova/nova.conf neutron auth_url http://192.168.10.50:5000
crudini --set /etc/nova/nova.conf neutron auth_type password
crudini --set /etc/nova/nova.conf neutron project_domain_name Default
crudini --set /etc/nova/nova.conf neutron user_domain_name Default
crudini --set /etc/nova/nova.conf neutron project_name service
crudini --set /etc/nova/nova.conf neutron username neutron
crudini --set /etc/nova/nova.conf neutron password Viethung1008
```

```
yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables ipset
```

```
cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.bak
cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak
cp /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.bak
cp /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini.bak

crudini --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
crudini --set /etc/neutron/neutron.conf DEFAULT core_plugin ml2
crudini --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:Viethung1008@192.168.10.50
crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes true
crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes true

crudini --set /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://192.168.10.50:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_url http://192.168.10.50:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken memcached_servers 192.168.10.50:11211
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_type password
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_domain_name Default
crudini --set /etc/neutron/neutron.conf keystone_authtoken user_domain_name Default
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_name service
crudini --set /etc/neutron/neutron.conf keystone_authtoken username neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken password Viethung1008

crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp
```

```
echo 'net.bridge.bridge-nf-call-iptables = 1' >> /etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-ip6tables = 1' >> /etc/sysctl.conf
modprobe br_netfilter
/sbin/sysctl -p
```

```
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini linux_bridge physical_interface_mappings provider:eth1
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan enable_vxlan True
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan local_ip $(ip addr show dev eth2 scope global | grep "inet " | sed -e 's#.*inet ##g' -e 's#/.*##g')
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup enable_security_group True
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

crudini --set /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_host 192.168.10.50
crudini --set /etc/neutron/metadata_agent.ini DEFAULT metadata_proxy_shared_secret Viethung1008


crudini --set /etc/neutron/dhcp_agent.ini DEFAULT interface_driver neutron.agent.linux.interface.BridgeInterfaceDriver
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT enable_isolated_metadata True
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT force_metadata True
```

```
systemctl enable neutron-linuxbridge-agent.service
systemctl enable neutron-metadata-agent.service
systemctl enable neutron-dhcp-agent.service

systemctl start neutron-linuxbridge-agent.service
systemctl start neutron-metadata-agent.service
systemctl start neutron-dhcp-agent.service
systemctl restart openstack-nova-compute.service
```

## 2.5. Cài horizon trên controller

*Thực hiện trên node controller*

```
yum install openstack-dashboard -y
```
```
filehtml=/var/www/html/index.html
touch $filehtml
cat << EOF >> $filehtml
<html>
<head>
<META HTTP-EQUIV="Refresh" Content="0.5; URL=http://192.168.10.50/dashboard">
</head>
<body>
<center> <h1>Redirecting to OpenStack Dashboard</h1> </center>
</body>
</html>
EOF
```

Backup file cấu hình 

```
cp /etc/openstack-dashboard/local_settings /etc/openstack-dashboard/local_settings.org
```

Thay file cấu hình `/etc/openstack-dashboard/local_settings` bằng cấu hình như sau:

```
import os
from django.utils.translation import ugettext_lazy as _
from openstack_dashboard.settings import HORIZON_CONFIG
DEBUG = False
WEBROOT = '/dashboard/'
ALLOWED_HOSTS = ['*',]
OPENSTACK_API_VERSIONS = {
"identity": 3,
"image": 2,
"volume": 2,
}
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'
LOCAL_PATH = '/tmp'
SECRET_KEY='368b60149f3fc3228007'
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': ['192.168.10.50:11211',]
    }
}
OPENSTACK_HOST = "192.168.10.50"
OPENSTACK_KEYSTONE_URL = "http://192.168.10.50:5000/v3"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
OPENSTACK_NEUTRON_NETWORK = {
    'enable_router': False,
    'enable_quotas': False,
    'enable_ipv6': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_fip_topology_check': False,
}
TIME_ZONE = "Asia/Ho_Chi_Minh"
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'console': {
            'format': '%(levelname)s %(name)s %(message)s'
        },
        'operation': {
            'format': '%(message)s'
        },
    },
    'handlers': {
        'null': {
            'level': 'DEBUG',
            'class': 'logging.NullHandler',
        },
        'console': {
            'level': 'DEBUG' if DEBUG else 'INFO',
            'class': 'logging.StreamHandler',
            'formatter': 'console',
        },
        'operation': {
            'level': 'INFO',
            'class': 'logging.StreamHandler',
            'formatter': 'operation',
        },
    },
    'loggers': {
        'horizon': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'horizon.operation_log': {
            'handlers': ['operation'],
            'level': 'INFO',
            'propagate': False,
        },
        'openstack_dashboard': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'novaclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'cinderclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'keystoneauth': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'keystoneclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'glanceclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'neutronclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'swiftclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'oslo_policy': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'openstack_auth': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'django': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'django.db.backends': {
            'handlers': ['null'],
            'propagate': False,
        },
        'requests': {
            'handlers': ['null'],
            'propagate': False,
        },
        'urllib3': {
            'handlers': ['null'],
            'propagate': False,
        },
        'chardet.charsetprober': {
            'handlers': ['null'],
            'propagate': False,
        },
        'iso8601': {
            'handlers': ['null'],
            'propagate': False,
        },
        'scss': {
            'handlers': ['null'],
            'propagate': False,
        },
    },
}
SECURITY_GROUP_RULES = {
    'all_tcp': {
        'name': _('All TCP'),
        'ip_protocol': 'tcp',
        'from_port': '1',
        'to_port': '65535',
    },
    'all_udp': {
        'name': _('All UDP'),
        'ip_protocol': 'udp',
        'from_port': '1',
        'to_port': '65535',
    },
    'all_icmp': {
        'name': _('All ICMP'),
        'ip_protocol': 'icmp',
        'from_port': '-1',
        'to_port': '-1',
    },
    'ssh': {
        'name': 'SSH',
        'ip_protocol': 'tcp',
        'from_port': '22',
        'to_port': '22',
    },
    'smtp': {
        'name': 'SMTP',
        'ip_protocol': 'tcp',
        'from_port': '25',
        'to_port': '25',
    },
    'dns': {
        'name': 'DNS',
        'ip_protocol': 'tcp',
        'from_port': '53',
        'to_port': '53',
    },
    'http': {
        'name': 'HTTP',
        'ip_protocol': 'tcp',
        'from_port': '80',
        'to_port': '80',
    },
    'pop3': {
        'name': 'POP3',
        'ip_protocol': 'tcp',
        'from_port': '110',
        'to_port': '110',
    },
    'imap': {
        'name': 'IMAP',
        'ip_protocol': 'tcp',
        'from_port': '143',
        'to_port': '143',
    },
    'ldap': {
        'name': 'LDAP',
        'ip_protocol': 'tcp',
        'from_port': '389',
        'to_port': '389',
    },
    'https': {
        'name': 'HTTPS',
        'ip_protocol': 'tcp',
        'from_port': '443',
        'to_port': '443',
    },
    'smtps': {
        'name': 'SMTPS',
        'ip_protocol': 'tcp',
        'from_port': '465',
        'to_port': '465',
    },
    'imaps': {
        'name': 'IMAPS',
        'ip_protocol': 'tcp',
        'from_port': '993',
        'to_port': '993',
    },
    'pop3s': {
        'name': 'POP3S',
        'ip_protocol': 'tcp',
        'from_port': '995',
        'to_port': '995',
    },
    'ms_sql': {
        'name': 'MS SQL',
        'ip_protocol': 'tcp',
        'from_port': '1433',
        'to_port': '1433',
    },
    'mysql': {
        'name': 'MYSQL',
        'ip_protocol': 'tcp',
        'from_port': '3306',
        'to_port': '3306',
    },
    'rdp': {
        'name': 'RDP',
        'ip_protocol': 'tcp',
        'from_port': '3389',
        'to_port': '3389',
    },
}
```

```
echo "WSGIApplicationGroup %{GLOBAL}" >> /etc/httpd/conf.d/openstack-dashboard.conf
systemctl restart httpd.service memcached.service
```

Truy cập vào trang chủ với địa chỉ `http://192.168.10.50` với user/pass: `admin/Viethung1008`

## 2.6. Cài Cinder

*Thực hiện trên node controller*

```
mysql -u root -pViethung1008@
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'Viethung1008';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'Viethung1008';
exit
```

```
source /root/admin-openrc
openstack user create --domain default --password Viethung1008 cinder

openstack role add --project service --user cinder admin

openstack service create --name cinderv2 \
--description "OpenStack Block Storage" volumev2

openstack service create --name cinderv3 \
--description "OpenStack Block Storage" volumev3


openstack endpoint create --region RegionOne \
volumev2 public http://192.168.10.50:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne \
volumev2 internal http://192.168.10.50:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne \
volumev2 admin http://192.168.10.50:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne \
volumev3 public http://192.168.10.50:8776/v3/%\(project_id\)s

openstack endpoint create --region RegionOne \
volumev3 internal http://192.168.10.50:8776/v3/%\(project_id\)s

openstack endpoint create --region RegionOne \
volumev3 admin http://192.168.10.50:8776/v3/%\(project_id\)s
```

```
yum install -y openstack-cinder lvm2 device-mapper-persistent-data targetcli python-keystone
```

```
systemctl enable lvm2-lvmetad.service
systemctl start lvm2-lvmetad.service
```

### Add thêm 1 disk 100G sau đó thực hiện lệnh sau

```
pvcreate /dev/vdb
vgcreate cinder-volumes /dev/vdb
```

### Sửa file /etc/lvm/lvm.conf tại dòng 141 bỏ comment

```
filter = [ "a|.*/|" ]
```

### Backup cấu hình 

```
mv /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak
```

```
cat << EOF >> /etc/cinder/cinder.conf
[DEFAULT]
transport_url = rabbit://openstack:Viethung1008@192.168.10.50
auth_strategy = keystone
my_ip = 192.168.10.50
enabled_backends = lvm
glance_api_servers = http://192.168.10.50:9292
enable_v3_api = True
[backend]
[backend_defaults]
[barbican]
[brcd_fabric_example]
[cisco_fabric_example]
[coordination]
[cors]
[database]
connection = mysql+pymysql://cinder:Viethung1008@192.168.10.50/cinder
[fc-zone-manager]
[healthcheck]
[key_manager]
[keystone_authtoken]
www_authenticate_uri = http://192.168.10.50:5000
auth_url = http://192.168.10.50:5000
memcached_servers = 192.168.10.50:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = Viethung1008
[nova]
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[oslo_versionedobjects]
[privsep]
[profiler]
[sample_castellan_source]
[sample_remote_file_source]
[service_user]
[ssl]
[vault]
[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = lioadm
EOFcat << EOF >> /etc/cinder/cinder.conf
[DEFAULT]
transport_url = rabbit://openstack:Viethung1008@192.168.10.50
auth_strategy = keystone
my_ip = 192.168.10.50
enabled_backends = lvm
glance_api_servers = http://192.168.10.50:9292
enable_v3_api = True
[backend]
[backend_defaults]
[barbican]
[brcd_fabric_example]
[cisco_fabric_example]
[coordination]
[cors]
[database]
connection = mysql+pymysql://cinder:Viethung1008@192.168.10.50/cinder
[fc-zone-manager]
[healthcheck]
[key_manager]
[keystone_authtoken]
www_authenticate_uri = http://192.168.10.50:5000
auth_url = http://192.168.10.50:5000
memcached_servers = 192.168.10.50:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = Viethung1008
[nova]
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[oslo_versionedobjects]
[privsep]
[profiler]
[sample_castellan_source]
[sample_remote_file_source]
[service_user]
[ssl]
[vault]
[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = lioadm
EOF
```

```
su -s /bin/sh -c "cinder-manage db sync" cinder
systemctl restart openstack-nova-api.service
systemctl restart openstack-nova-api.service

systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service openstack-cinder-volume.service target.service
systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service openstack-cinder-volume.service target.service
```


