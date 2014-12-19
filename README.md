# Multinode Ceph on Vagrant

This workshop walks users through setting up a 3-node [Ceph](http://ceph.com) cluster and mounting a block device, using a CephFS mount, and storing a blob oject.

It follows the following Ceph user guides:

* [Preflight checklist](http://ceph.com/docs/master/start/quick-start-preflight/)
* [Storage cluster quick start](http://ceph.com/docs/master/start/quick-ceph-deploy/)
* [Block device quick start](http://ceph.com/docs/master/start/quick-rbd/)
* [Ceph FS quick start](http://ceph.com/docs/master/start/quick-cephfs/)
* [Install Ceph object gateway](http://ceph.com/docs/master/install/install-ceph-gateway/)
* [Configuring Ceph object gateway](http://ceph.com/docs/master/radosgw/config/)
* [Cephfilesystem](http://ceph.com/category/ceph-filesystem/)

Note that after many commands, you may see something like:

```
Unhandled exception in thread started by
sys.excepthook is missing
lost sys.stderr
```

I'm not sure what this means, but everything seems to have completed successfully, and the cluster will work.

## Install prerequisites

Install [Vagrant](http://www.vagrantup.com/downloads.html) and a provider such as [VirtualBox](https://www.virtualbox.org/wiki/Downloads).

We'll also need the [vagrant-cachier](https://github.com/fgrehm/vagrant-cachier) and [vagrant-hostmanager](https://github.com/smdahlen/vagrant-hostmanager) plugins:

```console
$ vagrant plugin install vagrant-cachier
$ vagrant plugin install vagrant-hostmanager
```

## Add your Vagrant key to the SSH agent

Since the admin machine will need the Vagrant SSH key to log into the server machines, we need to add it to our local SSH agent:

```console
$ ssh-add -k ~/.vagrant.d/insecure_private_key
```

## Start the VMs

This instructs Vagrant to start the VMs and install `ceph-deploy` on the admin machine.

```console
$ vagrant up
```

## Create the cluster

We'll create a simple cluster and make sure it's healthy. Then, we'll expand it.

First, we need to get an interactive shell on the admin machine:

```console
$ vagrant ssh ceph-admin
```

The `ceph-deploy` tool will write configuration files and logs to the current directory. So, let's create a directory for the new cluster:

```console
vagrant@ceph-admin:~$ mkdir test-cluster && cd test-cluster
```

Let's prepare the machines:

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy new ceph-server-1 ceph-server-2 ceph-server-3
```

Now, we have to change a default setting. For our initial cluster, we are only going to have two [object storage daemons](http://ceph.com/docs/master/man/8/ceph-osd/). We need to tell Ceph to allow us to achieve an `active + clean` state with just two Ceph OSDs. Add `osd pool default size = 2` to `./ceph.conf`. Now, it should look similar to:

```
[global]
fsid = c4e2f4d3-5c2b-4421-80a2-042527977269
mon_initial_members = ceph-server-1, ceph-server-2, ceph-server-3
mon_host = 172.21.12.11,172.21.12.12,172.21.12.13
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
filestore_xattr_use_omap = true
osd pool default size = 2
```

## Install Ceph

We're finally ready to install!

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy install ceph-admin ceph-server-1 ceph-server-2 ceph-server-3
```

## Configure monitor and OSD services

Next, we add a monitor node:

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy mon create-initial
```

And our two OSDs. For these, we need to log into the server machines directly:

```console
vagrant@ceph-admin:~/test-cluster$ ssh ceph-server-2 sudo mkdir /var/local/osd0
```

```console
vagrant@ceph-admin:~/test-cluster$ ssh ceph-server-3 sudo mkdir /var/local/osd1
```

Now we can prepare and activate the OSDs:

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy osd prepare ceph-server-2:/var/local/osd0 ceph-server-3:/var/local/osd1
vagrant@ceph-admin:~/test-cluster$ ceph-deploy osd activate ceph-server-2:/var/local/osd0 ceph-server-3:/var/local/osd1
```

## Configuration and status

We can copy our config file and admin key to all the nodes, so each one can use the `ceph` CLI.

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy admin ceph-admin ceph-server-1 ceph-server-2 ceph-server-3
```

We also should make sure the keyring is readable:

```console
vagrant@ceph-admin:~/test-cluster$ sudo chmod +r /etc/ceph/ceph.client.admin.keyring
vagrant@ceph-admin:~/test-cluster$ ssh ceph-server-1 sudo chmod +r /etc/ceph/ceph.client.admin.keyring
vagrant@ceph-admin:~/test-cluster$ ssh ceph-server-2 sudo chmod +r /etc/ceph/ceph.client.admin.keyring
vagrant@ceph-admin:~/test-cluster$ ssh ceph-server-3 sudo chmod +r /etc/ceph/ceph.client.admin.keyring
```

Finally, check on the health of the cluster:

```console
vagrant@ceph-admin:~/test-cluster$ ceph health
```

You should see something similar to this once it's healthy:

```console
vagrant@ceph-admin:~/test-cluster$ ceph health
HEALTH_OK
vagrant@ceph-admin:~/test-cluster$ ceph -s
    cluster 18197927-3d77-4064-b9be-bba972b00750
     health HEALTH_OK
     monmap e2: 3 mons at {ceph-server-1=172.21.12.12:6789/0,ceph-server-2=172.21.12.13:6789/0,ceph-server-3=172.21.12.14:6789/0}, election epoch 6, quorum 0,1,2 ceph-server-1,ceph-server-2,ceph-server-3
     osdmap e9: 2 osds: 2 up, 2 in
      pgmap v13: 192 pgs, 3 pools, 0 bytes data, 0 objects
            12485 MB used, 64692 MB / 80568 MB avail
                 192 active+clean
```

Notice that we have two OSDs (`osdmap e9: 2 osds: 2 up, 2 in`) and all of the [placement groups](http://ceph.com/docs/master/rados/operations/pg-states/) (pgs) are reporting as `active+clean`.

Congratulations!

## Expanding the cluster

To more closely model a production cluster, we're going to add one more OSD daemon and a [Ceph Metadata Server](http://ceph.com/docs/master/man/8/ceph-mds/). We'll also add monitors to all hosts instead of just one.

### Add an OSD
```console
vagrant@ceph-admin:~/test-cluster$ ssh ceph-server-1 sudo mkdir /var/local/osd2
```

Now, from the admin node, we prepare and activate the OSD:
```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy osd prepare ceph-server-1:/var/local/osd2
vagrant@ceph-admin:~/test-cluster$ ceph-deploy osd activate ceph-server-1:/var/local/osd2
```

Watch the rebalancing:

```console
vagrant@ceph-admin:~/test-cluster$ ceph -w
```

You should eventually see it return to an `active+clean` state, but this time with 3 OSDs:

```console
vagrant@ceph-admin:~/test-cluster$ ceph -w
    cluster 18197927-3d77-4064-b9be-bba972b00750
     health HEALTH_OK
     monmap e2: 3 mons at {ceph-server-1=172.21.12.12:6789/0,ceph-server-2=172.21.12.13:6789/0,ceph-server-3=172.21.12.14:6789/0}, election epoch 30, quorum 0,1,2 ceph-server-1,ceph-server-2,ceph-server-3
     osdmap e38: 3 osds: 3 up, 3 in
      pgmap v415: 192 pgs, 3 pools, 0 bytes data, 0 objects
            18752 MB used, 97014 MB / 118 GB avail
                 192 active+clean
```

### Add metadata server

Let's add a metadata server to server1:

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy mds create ceph-server-1
```

## Add more monitors

We add monitors to servers 2 and 3.

```console
vagrant@ceph-admin:~/test-cluster$ ceph-deploy mon create ceph-server-2 ceph-server-3
```

Watch the quorum status, and ensure it's happy:

```console
vagrant@ceph-admin:~/test-cluster$ ceph quorum_status --format json-pretty
```

## Install Ceph Object Gateway

To be able to install FastCGI the multiverse repository must be enabled in Ubuntu
Remove the commented rows where multiverse is specified in /etc/apt/sources.list. You might ignore the backport lines. Or if you are lazy just run this command.

```ssh ceph-server-1 sudo sed -i'' 's/\# \(.*trusty.\{0,10\}multiverse\)/\1/g' /etc/apt/sources.list```

###Install Apache and FastCGI

```
ssh ceph-server-1 sudo apt-get update
ssh ceph-server-1 sudo apt-get install apache2 libapache2-mod-fastcgi
```

###Install Ceph optimized versions of Apache and FastCGI

```
ssh ceph-server-1 'wget -q -O- https://raw.github.com/ceph/ceph/master/keys/autobuild.asc | sudo apt-key add -
ssh ceph-server-1 'echo deb http://gitbuilder.ceph.com/apache2-deb-$(lsb_release -sc)-x86_64-basic/ref/master $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph-apache.list'
ssh ceph-server-1 'echo deb http://gitbuilder.ceph.com/libapache-mod-fastcgi-deb-$(lsb_release -sc)-x86_64-basic/ref/master $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph-fastcgi.list'
ssh ceph-server-1 sudo apt-get update && sudo apt-get install apache2 libapache2-mod-fastcgi
```

### Configure Apache

Set a server name in Apache and enable FastCGI. SSL can be configured, but we skip it in this test environment.

```
ssh ceph-server-1 'echo "ServerName $(hostname -f)" | sudo tee -a /etc/apache2/apache2.conf'
ssh ceph-server-1 sudo a2enmod rewrite
ssh ceph-server-1 sudo a2enmod fastci
ssh ceph-server-1 sudo service apache2 restart
```

### Add bucket hostname to DNS
Add a wildcard hostname in your DNS. In dnsmasq you add this line to dnsmasq.conf. 
I didn't get this to work in dnsmasq.conf running on OpenWrt (14.07) so I manually added bucket.ceph-server-1 and ceph-server-1 to point to correct file

```
IP_ADDRESS=$(ifconfig eth1 | awk -F ' *|:' '/inet addr/{print $4}')

address=/.ceph-server-1/$(IP_ADDRESS)
```



### Install Ceph Object Gateway
Install the gateway deamon and agent. 
```
ssh ceph-server-1 sudo apt-get install radosgw
```

For federated architectures, the synchronization agent, radosgw-agent, provides data and metadata synchronization between zones and regions. In this setup we will use a simple configuration and not a federated configuration so we skip the agent.

### Configure Ceph Object Gateway
We configure a simple setup. 
Create a keyring, add capabilities and add key to Ceph Storage Cluster.

```
sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.radosgw.keyring
sudo chmod +r /etc/ceph/ceph.client.radosgw.keyring
sudo ceph-authtool -n client.radosgw.gateway --cap osd 'allow rwx' --cap mon 'allow rwx' /etc/ceph/ceph.client.radosgw.keyring
sudo ceph -k /etc/ceph/ceph.client.admin.keyring auth add client.radosgw.gateway -i /etc/ceph/ceph.client.radosgw.keyring

```

### Distribute key to gateway instance

```
cat /etc/ceph/ceph.client.radosgw.keyring | ssh ceph-server-1 'cat - | sudo tee /etc/ceph/ceph.client.radosgw.keyring'
```

### Add gateway instance's config
To run on the ceph-admin host
```
cd ~/test-cluster
echo "
[client.radosgw.gateway]
host = ceph-server-1
keyring = /etc/ceph/ceph.client.radosgw.keyring
rgw dns name = ceph-server-1
rgw socket path = /var/run/ceph/ceph.radosgw.gateway.fastcgi.sock
log file = /var/log/radosgw/client.radosgw.gateway.log" | sudo tee -a ceph.conf
```
Push the configuration to servers
```
ceph-deploy --overwrite-conf config push ceph-server-{1,2,3} ceph-admin ceph-client
```

### Add a Ceph Object Gateway script
```
echo '#!/bin/sh
exec /usr/bin/radosgw -c /etc/ceph/ceph.conf -n client.radosgw.gateway' | ssh ceph-server-1 sudo tee /var/www/html/s3gw.fcgi
ssh ceph-server-1 sudo chmod +x /var/www/html/s3gw.fcgi
```

### Create data directory
```
ssh ceph-server-1 sudo mkdir -p /var/lib/ceph/radosgw/ceph-radosgw.gateway
```

### Create a Gateway Configuration
```
ssh ceph-server-1 'echo "FastCgiExternalServer /var/www/s3gw.fcgi -socket /var/run/ceph/ceph.radosgw.gateway.fastcgi.sock

<VirtualHost *:80>

	ServerName $(hostname -f)
	ServerAlias *.$(hostname -f)
	ServerAdmin root
	DocumentRoot /var/www
	RewriteEngine On
	RewriteRule  ^/(.*) /s3gw.fcgi?%{QUERY_STRING} [E=HTTP_AUTHORIZATION:%{HTTP:Authorization},L]

	<IfModule mod_fastcgi.c>
   	<Directory /var/www>
			Options +ExecCGI
			AllowOverride All
			SetHandler fastcgi-script
			Order allow,deny
			Allow from all
			AuthBasicAuthoritative Off
		</Directory>
	</IfModule>

	AllowEncodedSlashes On
	ErrorLog /var/log/apache2/error.log
	CustomLog /var/log/apache2/access.log combined
	ServerSignature Off

</VirtualHost>" | sudo tee /etc/apache2/sites-available/rgw.conf'
```
### Set gateway site as main site
Enable gateway site and disable the default site.
```
ssh ceph-server-1 sudo a2ensite rgw.conf
ssh ceph-server-1 sudo a2dissite 000-default
```

### Verify the gateway is running

Browser to http://ceph-server-1 and you should get the following result:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
  <Owner>
    <ID>anonymous</ID>
    <DisplayName></DisplayName>
  </Owner>
  <Buckets></Buckets>
</ListAllMyBucketsResult>
```

And when you browse to http://bucket.ceph-server-1 you get the following result:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Error>
  <Code>NoSuchBucket</Code>
</Error>
```

### Create a user
```
ssh ceph-server-1 radosgw-admin user create --uid=johndoe --display-name="John Doe" --email=john@example.com
```
Response (Your keys will be differnt):
```json
{ "user_id": "johndoe",
  "display_name": "John",
  "email": "john@example.com",
  "suspended": 0,
  "max_buckets": 1000,
  "auid": 0,
  "subusers": [],
  "keys": [
        { "user": "johndoe",
          "access_key": "14FSHI58HI54PW4LX9JS",
          "secret_key": "oMcJ8l1q8nxXkq0XfTZFPFFm39aeUT1TtTlQ3bdH"}],
  "swift_keys": [],
  "caps": [],
  "op_mask": "read, write, delete",
  "default_placement": "",
  "placement_tags": [],
  "bucket_quota": { "enabled": false,
      "max_size_kb": -1,
      "max_objects": -1},
  "user_quota": { "enabled": false,
      "max_size_kb": -1,
      "max_objects": -1},
  "temp_url_keys": []}
```

For more information about what you can do with users check out the [documentation](http://docs.ceph.com/docs/master/radosgw/admin/).

### Time for using the storage
To test Ceph Object Gateway we can use a nice tool named DragonDisk. It's freeware and available here, [http://www.dragondisk.com](http://www.dragondisk.com). Download and unzip it.

Start DragonDisk with the start script, dragondisk. Then add an account via the File-menu. Change the Provider to "Other S3 compatible service" and enter ceph-server-1 as Service Endpoint. Fill in the username, access key and secret key and save the account.


## Play around!

Now that we have everything set up, let's actually use the cluster. We'll use the ceph-client machine for this.

### Create a block device

TODO

## Create a mount with Ceph FS

We will add the Ceph FS to the ceph-client machine. 

### First install ceph on client
```
ceph-deploy install ceph-client
```

### Create a separate pool for Ceph FS
```
rados mkpool cephfs
```

### Create a keyring for Ceph FS that the client will use
```
sudo ceph auth get-or-create client.cephfs mon 'allow r' osd 'allow rwx pool=cephfs' -o /etc/ceph/client.cephfs.keyring
```

### Extract secret from keyring
```
ceph-authtool -p -n client.cephfs /etc/ceph/client.cephfs.keyring | sudo tee /etc/ceph/client.cephfs > /dev/null
```

#### Copy the secret file to client
This allow filesystem to mount when cephx authentication is enabled
```
cat /etc/ceph/client.cephfs | ssh ceph-client sudo tee /etc/ceph/client.cephfs > /dev/null
```

### Add mount to fstab on client
The IP number of ceph-server-1, where the MDS runs is added to fstab.
```
echo "172.21.12.12:6789:/ /cephfs ceph name=cephfs,secretfile=/etc/ceph/client.cephfs,noatime 0 2" | ssh tee -a /etc/fstab
```

### Create the mount directory and mount it
```
ssh ceph-client sudo mkdir /cephfs
ssh ceph-client sudo mount /cephfs
```

### Verify that CephFS is mounted
```
ssh ceph-client df
```
The output should look something like this
```
Filesystem          1K-blocks      Used Available Use% Mounted on
/dev/sda1            41251136   1149000  38366224   3% /
none                        4         0         4   0% /sys/fs/cgroup
udev                   245924        12    245912   1% /dev
tmpfs                   50180       352     49828   1% /run
none                     5120         0      5120   0% /run/lock
none                   250888         0    250888   0% /run/shm
none                   102400         0    102400   0% /run/user
vagrant             476552216 471357628   5194588  99% /vagrant
vagrant-cache       476552216 471357628   5194588  99% /tmp/vagrant-cache
172.21.12.12:6789:/ 123752448  26193920  97558528  22% /cephfs
```

### Test the file system and redundancy
First we create a directory for the vagrant user and copy a file to CephFS. 
```
ssh ceph-client sudo mkdir testdata
ssh ceph-client sudo chown vagrant:vagrant testdata
scp /usr/lib/libxapian.so.22 ceph-client:/cephfs/testdata
```
Then we shutdown the second server.

**Note!** This command shall be run on the host machine that runs VirtualBox.
```
vagrant halt ceph-server-2
```
Copy the file from CephFS and verify it
```
scp ceph-client:/cephfs/testdata/libxapian.so.22 /tmp
diff /tmp/libxapian.so.22 /usr/lib/libxapian.so.22
```

Start up the second server again.

**Note!** This command shall be run on the host machine that runs VirtualBox.
```
vagrant up ceph-server-2
```

### Store a blob object

TODO

## Cleanup

When you're all done, tell Vagrant to destroy the VMs.

```console
$ vagrant destroy -f
```
