# OpenShift Container Platform as a Service
This repo provides Ansible playbooks to provision OpenShift as a service.  It's intended to be used with CloudForms and Ansible Tower in a service catalog 

NOTE: This requires Ansible 2.3 (from epel)!  Make certain Tower is at 2.3 for this to work

## Setup EC2 dynamic inventory
If you're testing this with Ansible core (outside of Tower), you will need to setup dynamic inventory to EC2.  To do this, use the following process: 

```
cd /etc/ansible/
sudo wget https://raw.github.com/ansible/ansible/devel/contrib/inventory/ec2.py
sudo wget https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/ec2.ini

# Add you credentials to the credentials section at the bottom of the ec2.ini file
cat << EOF >> /etc/boto.cfg

aws_access_key_id = XXXXXXXXXXXXXXX
aws_secret_access_key = XXXXXXXXXXXXXXXXXXX
EOF
chmod 755 /etc/ansible/ec2.py

# Python boto is required for this.  
sudo dnf install python2-boto3.noarch python3-boto3.noarch
ansible -i /etc/ansible/ec2.py -e ansible_user=ec2-user -m ping all

# Takes about 30 seconds to pull everything

# Add ignore host key checks in /etc/ansible/ansible.cfg
[defaults]
host_key_checking = False

# Ping test (optional)
ansible -i /etc/ansible/ec2.py -e ansible_user=ec2-user -m ping all
```

## Steps to deploy OpenShift on AWS 
These steps are used to manually execute via Ansible core.  

```
# Prep AWS (Create VPC, Subnet, Internet Gateway, etc)
ansible-playbook awsprep.yml
# Provision EC2 Instances 
ansible-playbook ec2_instances.yml
# Register to RHN, Setup correct repos, yum update, install docker, etc...
ansible-playbook -i /etc/ansible/ec2.py openshift-prep.yml
# Deploy OpenShift
ansible-playbook -i /etc/ansible/ec2.py openshift-deploy.yml

# To watch the progress of the openshift deploy
ssh ec2-user@bastion1.<cluster-name>.<domain-name>
tail -f /home/ec2-user/ansible.log
```

Atomic openshift master failed to start... 
Couldn't create retry file... 

## Quick Validation
```
   # Would be nice to put this in Ansible too 
# From bastion1 
ssh ec2-user@master1.<cluster-name>.internal
sudo su - 

oc get pods 
oc get all --all-namespaces=true
   # You should see a registry, router, logging, and hawkular all setup 

oc get pv 	# You should see PVCs for logging and metrics


# Create a user to test UI access (Note if using multi-master this must be done on all nodes)
htpasswd /etc/origin/master/htpasswd admin
oadm policy add-cluster-role-to-user cluster-admin admin

# Connect to dashboard as your 'admin' user 
# https://master.<cluster>.<domain>	# master.ocp1.example.com

# Connect to Metrics to confirm the hawkular page 
# https://hawkular-metrics.<cloudapps_suffix>/hawkular/metrics    # hawkular-metrics.apps.ocp1.example.com

# Connect to logging to confirm Kibana
# https://kibana.<cloudapps_suffix>	# kibana.apps.ocp1.example.com

# Connect to registry console
# oc get routes
# https://registry-console-default.<cloudapps_suffix>

# Should probably add a test app here... Something like spinning up hello-openshift... 
```

## Known Issues
* In Ansible 2.3, route 53 entries are only create or delete.  They are not idempotent.  If you rerun ansible and it provisions different hosts it will not correct the entries.  Ansible 2.4 makes these modules idempotent
* On reruns of openshift-prep, it will unregister / reregister every time and redo all the repos.  Some work is needed to get this to run idempotent...
* Delete local SSH key (in openshift_prep.yaml) fails due to it running 5 times at once (based on 5 hosts).  Refactor this somehow (add a wait?)
* openshift_prep.yaml should move tasks into roles... not keep in the overall playbook

* Setting hostnames breaks AWS cloud config (openshift_set_hostname=True).  As we want dynamic provisioning of AWS volumes, I've disabled setting hostnames 
https://github.com/openshift/origin/issues/6747

NEXT STEPS:
* Add retirement of RHN subscription (ie unsubscribe) for each node
* Need way to retire volumes
  * Delete on retire on instance creation? 
  * Or tag and delete tagged instances? 
* Need way to retire dymanic pvcs.  Are they tagged? 
* Load balancer retirement takes a long time... why?
* Break out individual tasks in my top level playbook into the respective roles.  
* Add secrets to  vault
* Split deployment across AZs similar to the refarch



## Resources

Security Hardened playbooks are from Ansible Lockdown: 
https://github.com/ansible/ansible-lockdown

Registry AWS S3 example inventory: 
https://github.com/sborenst/ansible_agnostic_deployer/blob/master/ansible/configs/opentlc-shared/files/hosts_template.j2

A possibility for project request from CloudForms:
https://github.com/kevensen/cfme-ocp/tree/master/docs
   I think it's in ruby though.  We'd just do that in Ansible..
OR this: configs/opentlc-shared/post_software.yml ... For some ideas on config...
   configs/opentlc-shared/env_vars.yml

This goes into all the other OpenShift + CloudForms config - should point me to scheduling image scans and basic chargeback if needed: 
https://github.com/redhat-cop/openshift-playbooks/blob/master/playbooks/operationalizing/cloudforms.adoc


## Tower Workflow 
1. Add OCPaaS playbooks as a project (either through manual, or via git)
2. Settings -> Credentials -> +ADD 
   Add your AWS Credentials 
   Name: AWS 
   Description: AWS Access
   Organization: Default
   Type: Amazon Web Services
   Access Key: ...
   Secret Key: ... 
#Add ec2.ini ... 
#cd /var/lib/awx/venv/tower/lib/python2.7/site-packages/awx/plugins/inventory
#curl -o ec2.ini https://raw.githubusercontent.com/ansible/ansible/devel/contrib/inventory/ec2.ini
#
## Add your AWS secret key / access key to the bottom of the file (under credentials)
#cat << EOF >> ec2.ini
#
#aws_access_key_id = XXXXXXXXXXXXXXX
#aws_secret_access_key = XXXXXXXXXXXXXXXXXXX
#EOF
#chown awx:awx ec2.ini

IMPORTANT - YOUR TIME MUST BE IN SYNC OR THE INVENTORY SYNC WILL FAIL!!!!! 
IMPORTANT IMPORTANT IMPORTANT

#Test as follows:
#cd /var/lib/awx/venv/tower/lib/python2.7/site-packages/awx/plugins/inventory
#
#In the end, I modified the script as follows:
#Either in /var/lib/awx/venv/tower/lib/python2.7/site-packages/awx/plugins/inventory
#3Or touch ~/.aws/credentials like this: 
#[default]
#aws_access_key_id = YOUR_ACCESS_KEY
#aws_secret_access_key = YOUR_SECRET_KEY

3. Create AWS Inventory
   Inventories -> Create inventory
   Org: Default
   Click 'Save' 
   Click 'Add Group' for this inventory
   Name: aws
   Source: Amazon EC2
   Cloud Credential: AWS (that you created earlier) 
   Regions: Choose All 
   Check Overwrite (to remove instances from inventory that are no longer on the provider
   Check update on launch  (so inventory is updated any time a job is launched )
   Click Save 
   In inventories, Click 'AWS' and run the sync on the aws group 

4. Create credentials for the EC2 machines 
   Settings -> Credentials -> +ADD
   Name: aws_ec2-user
   Org: Default
   Type: Machine 
   User: ec2-user
   Priv Escalation: Sudo
   Paste your private key

4. Create job templates for each individual component of the deployment

ocp-awsprep:
   Inventory: localhost	# Just a blank inventory as it's not needed
   Project: ocpaas 
   Playbook: awsprep.yml
   Machine Credential: Can be anything... it's not used
   Cloud Credential: AWS
	NOTE: If you do not specify a credential the playbook will fail with handler errors
	This isn't passed from var_files.  You Must specify in each play (if not using the cloud credentials) 

ocp-ec2_instances:
   Inventory: localhost # Just a blank inventory as it's not needed
   Project: ocpaas
   Playbook: ec2_instances.yml
   Machine Credential: Can be anything... it's not used
   Cloud Credential: AWS

ocp-openshift-prep
   Inventory: AWS
   Project: ocpaas
   Playbook: openshift_prep-yml
   Machine Creds: aws_ec2-user
   Cloud Creds: AWS 

ocp-openshift-deploy
   Inventory: AWS
   Project: ocpaas
   Playbook: openshift_prep-yml
   Machine Creds: aws_ec2-user
   Cloud Creds: AWS

5. Create a workflow to tie these altogether
   Add -> Workflow Job Template
   Name: ocp-deploy
   Org: Default
   Save
   Workflow Editor
   Start -> ocp-awsprep -> ocp-ec2_instances -> ocp-openshift-prep -> ocp-openshift-deploy

Test by launching ocp-deploy ... 

## Tower CLI 

```
yum localinstall https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum --enablerepo=epel install python2-pip
pip install ansible-tower-cli 
```


