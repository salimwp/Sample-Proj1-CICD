This is an example project for using Git/Jenkins for configuring a Fortigate appliance
with VLAN and firewall policies and then setting up 2 instances of Wordpress. This example is essentially just a smoke test to ensure the CICD is functional.

The CICD example that the 2 CentOS 7.7 servers are installed with just the minimal configuration at 10.1.1.80 for dev and 10.1.2.80 for prod. Jenkins will also need to have its SSH key installed on those 2 servers for deployments.

First, launch the ansible playbook in proj1-cicd folder to create to Git repos on the
CICD server. This will need a SSH key located in proj1-cicd/keys/myKey to be created.

Under Linux or Mac, use ssh-keygen -f myKey to create the SSH key followed by ssh-add myKey to add the key to your keychain. In later examples, we will use a centralized authentication system to help manage the SSH keys.

Run the ansible playbook in the proj1-cicd folder.
```shell
$ cd proj1-cicd
$ mkdir keys
$ cd keys
$ ssh-keygen -f myKey
$ ssh-add myKey
$ ansible-playbook -i hosts -k -K cicd.yaml
```
Once the playbook has completed there will be 2 Git repos on the CICD server. 

Log into the Jenkins servers WebUI https://192.168.50.30 and select 'Credentials'. You will see a list of existing Credentials. Under to Domain, click on one of the '(global)' links and then on 'Add Credentials'

You will now need to create an account for Jenkins to access your newly created Git repoes. Under 'Kind' select 'SSH Username with Private Key'. For username use 'git-proj1app'. For 'Private Key', select 'Enter directly' and click on 'Add'. Open up your 'myKey' file and copy and paste the contents. Click on 'OK'. Repeat this process for 'git-proj1net'.

Navigate back to the home screen on Jenkins and click on 'New Item'. Enter in 'proj1-net' for the item name and select 'Pipeline'. Click on 'OK' to create this item. Navigate to 'Build Triggers' and check 'Trigger builds remotely (e.g. from scripts)' and enter in an 'Authentication Token' and note the calling URL. 

Navigate down to 'Pipeline'. From Definition, select 'Pipeline script from SCM'. Select 'Git' from the SCM type. The repository URL is the URL of the Git repo clone string. For our example it will be ssh://git-proj1net@192.168.50.30/srv/git-proj1net/proj1net.git. Under credentials, select 'git-proj1net'. Everything else can stay as default and click on 'Save'.

You will need to log into your Git repo and configure the hook to communicate with Jenkins.

```sh
$ sudo -u git-proj1net vi /srv/git-proj1net/proj1net.git/hooks/post-receive
#!/bin/bash
curl -k --user <login>:<pass> https://192.168.50.30/job/proj1-net/build?token=<token>
$ sudo chown git-proj1net.git-proj1net /srv/git-proj1net/proj1net.git/hooks/post-receive
$ sudo chmod +x  /srv/git-proj1net/proj1net.git/hooks/post-receive
```

We can now test out our Git/Jenkins setup by going into the proj1-net directory and configuring it for pushing to our Git/CICD server.

```sh
$ cd proj1-net
$ git init
$ git remote add origin ssh://git-proj1net@192.168.50.30/srv/git-proj1net/proj1net.git
$ git add .
$ git commit -m 'Initial commit'
$ git push origin master
```
You can watch the progress of the pipeline on Jenkins.

Now we can setup the CICD for the Wordpress installations. For our setup, jenkins will need to have its ssh key copied to the ansible user on the dev server and prod server to automate the application deployment without the need for passwords. This can be done by logging onto the CICD server and running the following command:

```sh
sudo -u jenkins ssh-copy-id ansible@10.1.1.80 # for the dev
sudo -u jenkins ssh-copy-id ansible@10.1.2.80 # for the prod
```

Now, repeat the steps above where you commited the network configurations to the git repo but substite in git-proj1app/projapp where you see proj1net.

```sh
$ cd proj1-app
$ git init
$ git remote add origin ssh://git-proj1app@192.168.50.30/srv/git-proj1app/proj1app.git
$ git add .
$ git commit -m 'Initial commit'
$ git push origin master
```

