### Web server
* Install httpd
  * On all nodes
    ```
    yum -y install httpd

    # Update /etc/httpd/conf.d/status.conf
    cat <<EOF>> /etc/httpd/conf.d/status.conf
    <Location /server-status>
        SetHandler server-status
        Order Deny,Allow
        Deny from all
        Allow from 127.0.0.1
    </Location>
    EOF
    ```
* Start httpd 
  * On all nodes
  ```
  sudo systemctl enable httpd

  sudo systemctl start httpd
  ```

### Pacemaker

* Install Pacemaker
  * On all nodes
  ```
  # disable selinux
  setenforce 0
  
  # install pacemaker
  yum -y install pacemaker pcs

  # enable and start Pacemaker 
  systemctl enable pcsd
  
  systemctl start pcsd
  ```
* Update passwords
  * On all nodes
  ```
  echo "password" | passwd --stdin hacluster 
  ```

* Configure pacemaker
  * On one *master* node 
  ```
  # Enable auth
  sudo pcs cluster auth <master> <slave1> <slave2> -u <username> -p <password>

  # setup cluster
  sudo pcs cluster setup --name cluster <master> <slave1> <slave2>
  ```
* Start Pacemaker Services
  * On one *master* node
  ```
  # start pacemaker services
  sudo pcs cluster start --all

  # enable corosync and pacemaker
  sudo systemctl enable corosync.service
  sudo systemctl enable pacemaker.service

  ```
* Disable pacemaker configuration
  * On one *master* node
  ```
  # disable stonith
  sudo pcs property set stonith-enabled=false
  
  # disable quorum
  sudo pcs property set no-quorum-policy=ignore

  ```
* Configure resource agents
  * On one *master* node
  ```
  # Configure virtual IP resource agent
  pcs resource create Cluster_VIP ocf:heartbeat:IPaddr2 ip=<VIP> cidr_netmask=24 op monitor interval=20s
  
  # Configure apache web server
  sudo pcs resource create WebServer ocf:heartbeat:apache statusurl="http://127.0.0.1/server-status" op monitor interval=20s
  ```
* Configure colocation constraints
  * On one *master* node
  ```
  # colocate both of them
  sudo pcs constraint colocation add WebServer Cluster_VIP INFINITY
  
  # order resources
  sudo pcs constraint order set Cluster_VIP WebServer
  ```

(More info)[https://www.digitalocean.com/community/tutorials/how-to-set-up-an-apache-active-passive-cluster-using-pacemaker-on-centos-7] 