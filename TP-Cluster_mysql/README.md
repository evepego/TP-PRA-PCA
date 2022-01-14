## Cluster SQL



L'objectif de ce TP est de monter une infrastructure MySQL en cluster.

MySQL Cluster utilise la technologie de stockage NDB qui permet de synchroniser les données entre les nœuds.

La best practice pour monter un MySQL Cluster est d'utiliser un "shared nothing environment", ce qui veut dire qu'aucun des composants du cluster ne partage le même hardware.

 ![Why Is This 30-Year-Old Website Architecture So Popular in 2017? - Hybrid  Cloud Management and Automation | Morpheus](https://coschedule.s3.amazonaws.com/52932/2abba76a-66f3-42d8-bdd6-60054b6fb5f4/shared-nothing-comparison.jpg)

Pour simplifier le TP et compte tenu de la limitation des ressources nous allons configurer 4 machines :

- 2 MySQL data nodes (ndbd), qui vont faire la redondance des données.
- 1 serveur qui aura la rôle Cluster Manager (ndb_mgmd).
- 2 serveur MySQL server/client (mysqld et mysql).

#### ![MySQL Cluster](D:\Téléchargements\MySQL Cluster.png)

| Properties | Management node         | Datanode 1              | Datanode 2              | Sql node                | Sql Node 2              |
| ---------- | ----------------------- | ----------------------- | ----------------------- | ----------------------- | ----------------------- |
| OS         | CentOS 7                | CentOS 7                | CentOS 7                | CentOS 7                | CentOS 7                |
| vCPU       | 1                       | 1                       | 1                       | 1                       | 1                       |
| Memory     | 1 GB                    | 1 GB                    | 1 GB                    | 1 GB                    | 1 GB                    |
| Disk       | 20 GB                   | 20 GB                   | 20 GB                   | 20 GB                   | 20 GB                   |
| Disk 1     | 1 GB                    | 10 GB                   | 10 GB                   |                         |                         |
| Hostname   | mgm_node                | data_node1              | data_node2              | sql_node                | sql_node                |
| FQDN       | mysql_mngmt.example.com | mysql_data1.example.com | mysql_data2.example.com | mysql_node1.example.com | mysql_node2.example.com |
| IP Address | 10.0.2.20               | 10.0.2.21               | 10.0.2.22               | 10.0.2.25               | 10.0.2.26               |



## OPTIONNEL :

Le vagrantfile pour ce TP est le suivant :

```sh
Vagrant.configure("2") do |config|
  
  # Configuration commune à toutes les machines
  config.vm.box = "centos7-custom"

  # Ajoutez cette ligne afin d'accélérer le démarrage de la VM (si une erreur 'vbguest' est levée, voir la note un peu plus bas)
  #config.vbguest.auto_update = false
  # Désactive les updates auto qui peuvent ralentir le lancement de la machine
  config.vm.box_check_update = false 
  # La ligne suivante permet de désactiver le montage d'un dossier partagé (ne marche pas tout le temps directement suivant vos OS, versions d'OS, etc.)
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.provider "virtualbox" do |vb|
    vb.gui = true
  end

  # Config une première VM "mgm_node"
  config.vm.define "mgm_node" do |mgm_node|
    mgm_node.vm.network "private_network", ip: "10.0.0.11"
    mgm_node.vm.provision "shell",
      inline: "sudo hostnamectl set-hostname mgm_node"
  end

  # Config une  VM "sql_node"
  config.vm.define "sql_node" do |sql_node|
    sql_node.vm.network "private_network", ip: "10.0.0.12"
    sql_node.vm.provision "shell",
      inline: "sudo hostnamectl set-hostname sql_node"
  end
  # Config une  VM "sql_node2"
  config.vm.define "sql_node2" do |sql_node2|
    sql_node2.vm.network "private_network", ip: "10.0.0.13"
    sql_node2.vm.provision "shell",
      inline: "sudo hostnamectl set-hostname sql_node2"
  end
  
  # Config une  VM "data_node1"
  config.vm.define "data_node1" do |data_node1|
    data_node1.vm.network "private_network", ip: "10.0.0.21"
    data_node1.vm.provision "shell",
      inline: "sudo hostnamectl set-hostname data_node1"
  end

  # Config une  VM "data_node2"
  config.vm.define "data_node2" do |data_node2|
    data_node2.vm.network "private_network", ip: "10.0.0.22"
    data_node2.vm.provision "shell",
      inline: "sudo hostnamectl set-hostname data_node2"
  end
end
```

### Installation du Management Node (mgm_node)

```sh
sudo su
cd ~
yum -y install wget
wget http://dev.mysql.com/get/Downloads/MySQL-Cluster-7.4/MySQL-Cluster-gpl-7.4.10-1.el7.x86_64.rpm-bundle.tar
tar -xvf MySQL-Cluster-gpl-7.4.10-1.el7.x86_64.rpm-bundle.tar
yum -y remove mariadb-libs
yum -y install perl-Data-Dumper
yum install -y libaio.x86_64 libaio-devel.x86_64 net-tools
rpm -Uvh MySQL-Cluster-client-gpl-7.4.10-1.el7.x86_64.rpm
rpm -Uvh MySQL-Cluster-server-gpl-7.4.10-1.el7.x86_64.rpm
rpm -Uvh MySQL-Cluster-shared-gpl-7.4.10-1.el7.x86_64.rpm
```

### Sur le management node (mgm_node) : configuration du MySQL Cluster, cette configuration va permettre au serveur de management de détecter tous les nœuds de notre cluster

```sh
mkdir -p /var/lib/mysql-cluster
cd /var/lib/mysql-cluster
mkdir /data
vim config.ini
```

```sh
[ndb_mgmd default]
# Directory for MGM node log files
DataDir=/data
 
[ndb_mgmd]
#Management Node db1
HostName=10.0.0.11
 
[ndbd default]
NoOfReplicas=2      # Number of replicas
DataMemory=256M     # Memory allocate for data storage
IndexMemory=128M    # Memory allocate for index storage
#Directory for Data Node
DataDir=/data

[ndbd]
#Data Node db1
HostName=10.0.0.21
 
[ndbd]
#Data Node db2
HostName=10.0.0.22

[mysqld]
#MySQL Node
HostName=10.0.0.12

[mysqld]
#MySQL Node2
HostName=10.0.0.13
```

On démarre le management node :

```sh
ndb_mgmd --config-file=/var/lib/mysql-cluster/config.ini
```

```sh
[root@mgm_node mysql-cluster]# ndb_mgm

ndb_mgm> show

Connected to Management Server at: localhost:1186
Cluster Configuration
---------------------
[ndbd(NDB)]     2 node(s)
id=2 (not connected, accepting connect from 10.0.0.21)
id=3 (not connected, accepting connect from 10.0.0.22)

[ndb_mgmd(MGM)] 1 node(s)
id=1    @10.0.0.11  (mysql-5.6.28 ndb-7.4.10)

[mysqld(API)]   2 node(s)
id=4 (not connected, accepting connect from 10.0.0.12)
id=5 (not connected, accepting connect from 10.0.0.13)

```

### Nous allons maintenant configurer les MySQL Cluster Data Nodes (sur data_node1 et data_node2) :

On configure les nodes pour qu'ils puissent contacter le management node :

```sh
sudo su
cd ~
yum -y install wget
wget http://dev.mysql.com/get/Downloads/MySQL-Cluster-7.4/MySQL-Cluster-gpl-7.4.10-1.el7.x86_64.rpm-bundle.tar
tar -xvf MySQL-Cluster-gpl-7.4.10-1.el7.x86_64.rpm-bundle.tar
yum -y remove mariadb-libs
yum -y install perl-Data-Dumper
yum install -y libaio.x86_64 libaio-devel.x86_64 net-tools
rpm -Uvh MySQL-Cluster-client-gpl-7.4.10-1.el7.x86_64.rpm
rpm -Uvh MySQL-Cluster-server-gpl-7.4.10-1.el7.x86_64.rpm
rpm -Uvh MySQL-Cluster-shared-gpl-7.4.10-1.el7.x86_64.rpm


vim /etc/my.cnf
```

```sh
[mysqld]
ndbcluster
ndb-connectstring=10.0.0.11     # IP address of Management Node
 
[mysql_cluster]
ndb-connectstring=10.0.0.11     # IP address of Management Node
```

```sh
mkdir -p /var/lib/mysql-cluster
mkdir /data

ndbd

2020-10-29 14:57:36 [ndbd] INFO     -- Angel connected to '10.0.0.11:1186'
2020-10-29 14:57:36 [ndbd] INFO     -- Angel allocated nodeid: 3
```



### On configure les nodes SQL pour qu'ils puissent contacter le management node :

```sh
sudo su
cd ~
yum -y install wget
wget http://dev.mysql.com/get/Downloads/MySQL-Cluster-7.4/MySQL-Cluster-gpl-7.4.10-1.el7.x86_64.rpm-bundle.tar
tar -xvf MySQL-Cluster-gpl-7.4.10-1.el7.x86_64.rpm-bundle.tar
yum -y remove mariadb-libs
yum -y install perl-Data-Dumper
yum install -y libaio.x86_64 libaio-devel.x86_64 net-tools
rpm -Uvh MySQL-Cluster-client-gpl-7.4.10-1.el7.x86_64.rpm
rpm -Uvh MySQL-Cluster-server-gpl-7.4.10-1.el7.x86_64.rpm
rpm -Uvh MySQL-Cluster-shared-gpl-7.4.10-1.el7.x86_64.rpm

vim /etc/my.cnf
```

```sh
[mysqld]
ndbcluster
#ndb-connectstring=10.0.0.11     # IP address for server management node
default_storage_engine=ndbcluster     # Define default Storage Engine used by MySQL
 
[mysql_cluster]
ndb-connectstring=10.0.0.11     # IP address for server management node
```

On peut maintenant lancer le service mysql sur les deux machines (**cela peut prendre quelques minutes**) :

```sh
mysqld --user=root

#votre shell sera bloqué donc je vous recommande de lancer un autre shell sur les deux serveurs pour administrer les bases
```

Nous allons maintenant configurer les bases MySQL :

```sh
sudo su
cd ~
cat .mysql_secret #permet de récupérer le mot de passe généré aléatoirement pour l'utilisateur root du MySQL

sql_node : SGM1RxnHHMrKqtDs #ceci est l'exemple du mot de passe que j'ai récupéré sur le sql_node1

sql_node2 : pBY9EXZtxy0vD8Uu #ceci est l'exemple du mot de passe que j'ai récupéré sur le sql_node2
```

On peut lancer le mysql_secure_boot sur les deux nodes SQL :

```sh
mysql_secure_installation #permet de sécuriser le mysql 

You already have a root password set, so you can safely answer 'n'.

Change the root password? [Y/n] Y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..

Change the root password? [Y/n] Y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!

Remove anonymous users? [Y/n] Y 
 ... Success!

Disallow root login remotely? [Y/n] n
 ... Success!

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reload privilege tables now? [Y/n] Y
 ... Success!

All done!  If you've completed all of the above steps, your MySQL
installation should now be secure.

Thanks for using MySQL!


```

On peut maintenant se connecter au mysql avec la nouveau mot de passe que l'on a défini précédemment :

```sh
mysql -u root -p
mysql> select user, host, password from mysql.user;
+------+-----------+-------------------------------------------+
| user | host      | password                                  |
+------+-----------+-------------------------------------------+
| root | localhost | *44367FB67F0548551DA6A5A6732A69A340694B71 |
| root | 127.0.0.1 | *44367FB67F0548551DA6A5A6732A69A340694B71 |
| root | ::1       | *44367FB67F0548551DA6A5A6732A69A340694B71 |
| root | %         | *44367FB67F0548551DA6A5A6732A69A340694B71 |
+------+-----------+-------------------------------------------+

#on récupère le mot de passe que l'on a dans la liste ci-dessus pour donner tous les privilèges sur le MySQL
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY PASSWORD '*570C286CB1F0B445A804AEA5A0133F065D08CC35' WITH GRANT OPTION;
Query OK, 0 rows affected (0.00 sec)

```

```sh
[root@mgm_node mysql-cluster]# ndb_mgm
-- NDB Cluster -- Management Client --
ndb_mgm> show
Connected to Management Server at: localhost:1186
Cluster Configuration
---------------------
[ndbd(NDB)]     2 node(s)
id=2    @10.0.0.21  (mysql-5.6.28 ndb-7.4.10, Nodegroup: 0, *)
id=3    @10.0.0.22  (mysql-5.6.28 ndb-7.4.10, Nodegroup: 0)

[ndb_mgmd(MGM)] 1 node(s)
id=1    @10.0.0.11  (mysql-5.6.28 ndb-7.4.10)

[mysqld(API)]   2 node(s)
id=4    @10.0.0.12  (mysql-5.6.28 ndb-7.4.10)
id=5    @10.0.0.13  (mysql-5.6.28 ndb-7.4.10)
```

Créez une database sur le sql node :

```sql
CREATE DATABASE sql_cluster;
```

Sur le node sql 2 :

```sql
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| ndbinfo            |
| performance_schema |
| sql_cluster        |
+--------------------+
5 rows in set (0.00 sec)
```

Commandes utiles :

```sh
ndb_mgm #permet de voir l'état du cluster
ndb_mgm -e "all report memory" #permet d'avoir l'utilisation sur les nodes
```



## Questions :



Quel impact aura la perte du management node ? 

Est-ce que le management node à un rôle critique dans le fonctionnement de mon cluster ? Quelle solution de secours peut-on imaginer ? 

Quel est l'impact de la perte de mon DataNode1 ?

Quel est l'impact de la perte de mon SQL node 2 ?



| Evenement | Evenement attendu | Evenement obtenu |
| --------- | ----------------- | ---------------- |
|           |                   |                  |
|           |                   |                  |

