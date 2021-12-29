---
layout: post
title: Tableau Three-Node HA Cluster in AWS
author: Angel Mary Babu
date: 2021-12-29 11:00:00 +0800
categories: [AWS]
tags: [AWS, devops, ]
math: true
mermaid: true
description: Tableau Three-Node HA Cluster in AWS

---

#### Points to note:
* All the EC2 nodes in the cluster should have the same Tableau server version.
* All the EC2 nodes in the cluster should have the same Linux version.
* Purchase an Sever Activation code which starts with TS-XXX-XXX-XXX
* 14-days trial license is available however, it is not possible to configure a three-node HA cluster with trial.

The exact requirement information can be found from the Tableau Official  Website [Link](https://help.tableau.com/current/server-linux/en-us/distrib_requ.htm)

##### To setup Initial Node:

1. Login to the EC2 instance as sudo user and create a directory to store the installer.
   ``` 
   mkdir angel
   cd angel
   ```
2. Download the Tableau server installer in the server (Latest releases can be found in the following [Link](https://www.tableau.com/support/releases))
   ```
   wget https://downloads.tableau.com/esdalt/2021.2.2/tableau-server-2021-2-2.x86_64.rpm
   ```
3. Now, we have to install the tableau server,
   ```
   sudo yum update
   sudo yum install tableau-server-2020-4-0.x86_64.rpm
   ```
   The default location will be `/opt/tableau/tableau_server`
 4. we have to initialize TSM service in the initial node.
    ```
    cd /opt/tableau/tableau_server/packages/scripts.<version_code>/
    sudo ./initialize-tsm --accepteula
    ```
5. Sign in to the TSM web interface with port and IP address. We have to allow the port 8850 in order to access the TSM interface.
6. It will ask to fill the information to sign up like your name,address etc.
7. Also, requires License to activate the Tableau server.
8. You can choose `Local` for Identity store.
9. Initization will be completed at this stage.


#### To add Primary and Secondary Nodes

Generate a bootstrap file for the additional nodes
1. Open TSM in a browser:
```https://<tsm-computer-name>:8850```

2. Click the Configuration tab, and in the Add a Node box, click Download Bootstrap File. We can copy this file into the primary node in a location. 
3. Perform following to initilize TSM
```
wget https://downloads.tableau.com/esdalt/2021.2.2/tableau-server-2021-2-2.x86_64.rpm
sudo yum update
sudo yum install tableau-server-<version>.x86_64.rpm
cd /opt/tableau/tableau_server/packages/scripts.<version_code>/
sudo ./initialize-tsm -b /path/to/<bootstrap>.json --accepteula
```

You can follow the same steps to configure the secondary node.

Now we have to add the nodes from the TSM GUI to complete the process.

1. Open TSM in a browser:
```https://<tsm-computer-name>:8850```
2. Click the Configuration tab. A message should tell you that new nodes were added.
3. Click Continue to dismiss the message.
4. Click Pending Changes at the top of the page:
5. Click Apply Changes and Restart and Confirm to confirm a restart of Tableau Server.

When Tableau Server restarts, the nodes are included with the minimum topology necessary.

##### Deploy a Coordination Service ensemble

The following steps illustrate how to deploy a new Coordination Service ensemble on an existing three-node Tableau Server cluster.

1. On the initial node, open a terminal session.
2. Stop Tableau Server:
```tsm stop```
3. Confirm there are no pending changes:
```tsm pending-changes list```
4. If there are pending changes, you need to either discard the changes or apply them. Applying pending changes will take some time:
Discard the changes:
```tsm pending-changes discard```
or
Apply the changes:
```tsm pending-changes apply```
Wait until the command completes and you are returned to the system prompt.

5. Get the node IDs for each node in the cluster:
```tsm topology list-nodes -v```
6. Use the tsm topology deploy-coordination-service command to add a new Coordination Service ensemble by adding the Coordination Service to specified nodes.
```tsm topology deploy-coordination-service -n node1,node2,node3```
Wait until the command completes and you are returned to the system prompt.
7. Start Tableau Server:
```tsm start```

##### To Configure Client File Services (CFS) on additional nodes
1. On the initial node, open a terminal session.
2. Find the node ID for the node you are adding CFS to:
```tsm topology list-nodes -v```
3. Add CFS on the node by specifying the node, the process, and a single instance.
For example,:
```tsm topology set-process -n node2 -pr clientfileservice -c 1```
To add CFS to additional nodes, repeat this step for each node.
4. Apply the changes:
```tsm pending-changes apply```

Now you can configure processes in Primary nodes and Secondary nodes.

Ref: https://help.tableau.com/current/server-linux/en-us/distrib_ha_install_3node.htm

##### Some Useful commands
###### To Sign in to Tableau Services Manager Web UI

To view the user accounts in the TSM authorisation group, run the following command in the Bash shell.
```
grep tsmadmin /etc/group
```
If the user account is not in the group, run the following command to add the user to the tsmadmin group:
```
sudo usermod -G tsmadmin -a <username>
```
###### To Create the Tableau Server administrator account

Create the Tableau Server administrator account.
If you are using LDAP for authentication, then the account that you specify here must be a user in the directory.

Run the following command:
```
tabcmd initialuser --server 'localhost:80' --username '<AD-user-name>'
```
On the other hand, if you are running Tableau Server with local authentication, the username and password that you specify here will be used to create the administrative account. Enter a strong password for this account.

Run the following command:
```
tabcmd initialuser --server 'localhost:80' --username 'admin'
```
###### To remove tableau server:
```sudo ./tableau-server-obliterate -a -y -y -y```
