main doc: https://sysdig.com/blog/deploy-openshift-aws/

// seems my key file was already in ~/.ssh/id_rsa

   option: minishift - https://github.com/minishift/minishift

   CloudFormation template: https://gist.github.com/mateobur/9435b803b912f3e980aacfb0151670b6
      update:
		eu-central-1:
		  CentOS7: "ami-e7ae7088"	    
        ....
        t2.large
		
aws s3 cp openshift_aws_base_01.yml s3://gabriel-personal/temp/openshift_aws_base_01.yml

```
aws cloudformation validate-template \
 --region eu-central-1 \
 --template-url "https://s3.eu-central-1.amazonaws.com/gabriel-personal/temp/openshift_aws_base_01.yml"


aws cloudformation create-stack \
 --region eu-central-1 \
 --stack-name openshifttest01 \
 --template-url "https://s3.eu-central-1.amazonaws.com/gabriel-personal/temp/openshift_aws_base_01.yml" \
 --parameters \
   ParameterKey=AvailabilityZone,ParameterValue=eu-central-1a \
   ParameterKey=KeyName,ParameterValue=aws_keypair \
 --capabilities=CAPABILITY_IAM		
```

master:  master.os-dev.sse-consulting.eu 
worker1: worker1.os-dev.sse-consulting.eu 
worker2: worker2.os-dev.sse-consulting.eu 
 
git clone https://github.com/openshift/openshift-ansible.git
cd openshift-ansible/
git checkout origin/release-3.6
cd ..

vi hosts
 - replace master-host-name
 - replace worker-host-name
``` 
vi preconfig.yml
 
test connection:
  ansible -i hosts -m ping --key-file aws_keypair_clear.pem nodes
 
ansible-playbook -i hosts preconfig.yml --key-file aws_keypair_clear.pem
 
ssh -l centos master
  sudo hostnamectl set-hostname master.os-dev.sse-consulting.eu --static
ssh -l centos worker1
  sudo hostnamectl set-hostname worker1.os-dev.sse-consulting.eu --static
ssh -l centosworker2
  sudo hostnamectl set-hostname worker2.os-dev.sse-consulting.eu --static
 ``` 
 
ansible-playbook -c paramiko -i hosts openshift-ansible/playbooks/byo/config.yml --key-file aws_keypair_clear.pem

SSH in the host you designated as master
  
```
/etc/origin/master/openshift-master.kubeconfig -> server: master.os-dev.sse-consulting.eu

  sudo vi /etc/origin/master/master-config.yaml
    change AWS url to master.os-dev.sse-consulting.eu
  sudo systemctl restart origin-master.service
  
  oc get nodes
  sudo htpasswd -b /etc/openshift/openshift-passwd admin <your_pass>
  https://master.os-dev.sse-consulting.eu:8443
  ```



 ssh master

copy images to local docker repo

```
    oc get nodes
	aws ecr get-login --no-include-email --region eu-central-1
	sudo docker login -u AWS -p "generated_password" https://535544306598.dkr.ecr.eu-central-1.amazonaws.com 
	
	sudo docker pull 535544306598.dkr.ecr.eu-central-1.amazonaws.com/aws-apim:apim-rdbms-kubernetes-2.1.0
	sudo docker pull 535544306598.dkr.ecr.eu-central-1.amazonaws.com/aws-apim:wso2am-kubernetes-2.1.0
	sudo docker pull 535544306598.dkr.ecr.eu-central-1.amazonaws.com/aws-apim:wso2am-analytics-kubernetes-2.1.0

	sudo docker tag 535544306598.dkr.ecr.eu-central-1.amazonaws.com/aws-apim:apim-rdbms-kubernetes-2.1.0 docker.wso2.com/apim-rdbms-kubernetes:2.1.0
	sudo docker tag 535544306598.dkr.ecr.eu-central-1.amazonaws.com/aws-apim:wso2am-kubernetes-2.1.0 docker.wso2.com/wso2am-kubernetes:2.1.0
	sudo docker tag 535544306598.dkr.ecr.eu-central-1.amazonaws.com/aws-apim:wso2am-analytics-kubernetes-2.1.0 docker.wso2.com/wso2am-analytics-kubernetes:2.1.0 
	 
	sudo docker save -o apim-rdbms-kubernetes-2.1.0.tgz docker.wso2.com/apim-rdbms-kubernetes:2.1.0 
	sudo docker save -o wso2am-kubernetes-2.1.0.tgz docker.wso2.com/wso2am-kubernetes:2.1.0
	sudo docker save -o wso2am-analytics-kubernetes-2.1.0.tgz docker.wso2.com/wso2am-analytics-kubernetes:2.1.0
    
	sudo chown centos:centos *.tgz
	scp apim-rdbms-kubernetes-2.1.0.tgz centos@ip-10-0-0-11.eu-central-1.compute.internal:/home/centos
	scp wso2am-kubernetes-2.1.0.tgz centos@ip-10-0-0-11.eu-central-1.compute.internal:/home/centos
	scp wso2am-analytics-kubernetes-2.1.0.tgz centos@ip-10-0-0-11.eu-central-1.compute.internal:/home/centos
	scp apim-rdbms-kubernetes-2.1.0.tgz centos@ip-10-0-0-14.eu-central-1.compute.internal:/home/centos
	scp wso2am-kubernetes-2.1.0.tgz centos@ip-10-0-0-14.eu-central-1.compute.internal:/home/centos
	scp wso2am-analytics-kubernetes-2.1.0.tgz centos@ip-10-0-0-14.eu-central-1.compute.internal:/home/centos
	
	(worker1) ssh centos@ip-10-0-0-11
	sudo docker load -i apim-rdbms-kubernetes-2.1.0.tgz
	sudo docker load -i wso2am-analytics-kubernetes-2.1.0.tgz
	sudo docker load -i wso2am-kubernetes-2.1.0.tgz
	
	(worker2) ssh centos@ip-10-0-0-14
	sudo docker load -i apim-rdbms-kubernetes-2.1.0.tgz
	sudo docker load -i wso2am-analytics-kubernetes-2.1.0.tgz
	sudo docker load -i wso2am-kubernetes-2.1.0.tgz

```	
	
cd projects
git clone https://rd_gabriel_vince@bitbucket.org/koenge/kubernetes-apim.git
cd kubernetes-apim

NFS:

```
  sudo mkdir -p /mnt/data/wso2
  useradd -u 1000000000 wso2user
  sudo chown 1000000000 /mnt/data/wso2

 useradd -u 1000000000 wso2user
 (edit persistent-volumes.yaml)
mkdir -p /mnt/data/wso2/pattern-2/apim1
mkdir -p /mnt/data/wso2/pattern-2/apim2
mkdir -p /mnt/data/wso2/pattern-2-pv-3
mkdir -p /mnt/data/wso2/pattern-2/apim3
```
 
 
 NFS: 10.0.0.4
 path:  /mnt/data/wso2
 
 

```

oc login -u system:admin
# oc create user admin --full-name=admin
oc adm policy add-cluster-role-to-user cluster-admin admin

oc new-project wso2 --description="WSO2 API Manager 2.1.0" --display-name="wso2"
oc create serviceaccount wso2svcacct 
oc adm policy add-scc-to-user anyuid -z wso2svcacct -n wso2
oc policy add-role-to-user view system:serviceaccount:wso2:wso2svcacct -n wso2
oc adm policy add-scc-to-user privileged  -z wso2svcacct


deploy_openshift.sh


sudo mkdir -p /mnt/data/exports/pattern-2/apim1
sudo mkdir -p /mnt/data/exports/pattern-2/apim2
sudo mkdir -p /mnt/data/exports/pattern-2/apim3
sudo mkdir -p /tmp/data/pattern-2-pv-3
```


        volumeMounts:
        - name: apim-km-storage-volume
          mountPath: "/home/wso2user/wso2am-2.1.0/repository/deployment/server"
		  
   volumes:
      - name: apim-km-storage-volume
        persistentVolumeClaim:
          claimName: apim-km-volume-claim

```

