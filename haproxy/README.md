### GALERA Installation

* Install galera - on all nodes
    ```
    cat <<MARIADB > /etc/yum.repos.d/mariadb.repo
    [mariadb]
    name = MariaDB
    baseurl = http://yum.mariadb.org/10.1/centos7-amd64
    gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
    gpgcheck=1
    MARIADB
    
    setenforce 0
    
    yum -y install MariaDB-server MariaDB-client MariaDB-compat galera socat jemalloc vim net-tools
    ```

* Configure galera
  * On master
  ```
    # /etc/my.cnf.d/server.cnf
    [server]

    [mysqld]

    [galera]
    wsrep_on=ON
    wsrep_provider=/usr/lib64/galera/libgalera_smm.so
    wsrep_cluster_address='gcomm://192.168.66.11,192.168.66.12'
    wsrep_cluster_name='haproxy'
    wsrep_node_address='192.168.66.11'
    wsrep_node_name='haproxy1'
    wsrep_sst_method=rsync
    binlog_format=row
    default_storage_engine=InnoDB
    innodb_autoinc_lock_mode=2
    [embedded]
    [mariadb]
    [mariadb-10.1]
  ```

  * On other nodes
  ```
    [server]

    [mysqld]
    [galera]
    wsrep_on=ON
    wsrep_provider=/usr/lib64/galera/libgalera_smm.so
    wsrep_cluster_address='gcomm://192.168.66.11,192.168.66.12'
    wsrep_cluster_name='haproxy'
    wsrep_node_address='192.168.66.11'
    wsrep_node_name='haproxy1'
    wsrep_sst_method=rsync
    binlog_format=row
    default_storage_engine=InnoDB
    innodb_autoinc_lock_mode=2
    [embedded]
    [mariadb]
    [mariadb-10.1]
  ```

* Start galera
  * On master
  ``` 
  galera_new_cluster
  ```
  * On other nodes
  ```
  sudo service enable mariadb
  ```

* On all systems reboot (if all nodes failed)
  * On master
  ```
  sed -ie '/safe_to/c\safe_to_bootstrap: 1' /var/lib/mysql/grastate.dat

  galera_new cluster
  ```
  * On all nodes
  ```
  sudo systemctl restart mariadb
  ```

### KEEPALIVED 
* Configure few sysctl options
  ```
  # Load balancing in HAProxy and Keepalived at the same time also requires the ability to bind to an IP address that are nonlocal, meaning that it is not assigned to a device on the local system. This allows a running load balancer instance to bind to an IP that is not local for failover.

  sysctl net.ipv4.ip_nonlocal_bind = 1
  
  # In order for the Keepalived service to forward network packets properly to the real servers, each router node must have IP forwarding turned on in the kernel

  sysctl net.ipv4.ip_forward = 1

  sysctl -p 
  ```

* Install keepalived
  * On master and slaves
  ```
  yum -y install keepalived
  ```

* Configure keepalived
  * On master
  ```
    global_defs {
    router_id haproxy.cs
    }
    vrrp_script haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
    }
    vrrp_instance 50 {
    virtual_router_id 50
    advert_int 1
    nopreempt
    priority 101
    state MASTER
    interface eth1
    virtual_ipaddress {
        192.168.66.10 dev eth1
    }
    track_script {
        haproxy
    }
    }
  ```
  * On other nodes
  ```
    global_defs {
    router_id haproxy.cs
    }
    vrrp_script haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
    }
    vrrp_instance 50 {
    virtual_router_id 50
    advert_int 1
    priority 1
    state BACKUP
    interface eth1
    virtual_ipaddress {
        192.168.66.10 dev eth1
    }
    track_script {
        haproxy
    }
    }
  ```

* Start keepalived on all nodes
```
sudo service keepalived start
```


### HAProxy

* Install haproxy
  ```
  yum -y install haproxy
  ```
* Configure Haproxy
  ```
    global
            log 127.0.0.1   local0
            log 127.0.0.1   local1 notice
            maxconn 1024
            user haproxy
            group haproxy
            daemon

    defaults
            log     global
            mode    http
            option  tcplog
            option  dontlognull
            retries 3
            option redispatch
            maxconn 1024
            timeout connect 5000ms
            timeout client 50000ms
            timeout server 50000ms

    frontend galera
        bind	192.168.66.10:3307
        mode tcp
        default_backend	galera_cluster

    backend	galera_cluster
        balance roundrobin
        mode tcp
        option httpchk
        server haproxy1	192.168.66.11:3306 check port 9200 weight 1
        server haproxy2	192.168.66.12:3306 check port 9200 weight 1

    listen stats 192.168.66.10:8080
        mode http
        log global
        maxconn 10

        stats enable
        stats hide-version
        stats refresh 30s
        stats show-node
        stats uri /haproxy?stats
        stats auth admin:password
  ```
* Start haproxy
  ```
  sudo systemctl restart haproxy
  ```