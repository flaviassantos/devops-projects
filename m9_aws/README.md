#  AWS Services

#### Github repo

[]()
[]()


## 1. Deploy Web Application on EC2 Instance (manually)

#### Technologies used:
AWS, Docker, Linux

#### Project Description:
Create and configure an EC2 Instance on AWS Install Docker on remote EC2 Instance
Deploy Docker image from private Docker repository on EC2 Instance


### 1.1 Created Admin IAM User

Best Practice:
- Assign the permission (policy) to the Role, rather than on the User directly
- Give User the least privilege they need

![_](img/admin.png)

### 1.2 EC2 Instance created

![_](img/createec2.png)

### 1.3 Stored Private Key in .ssh folder
Also change permission of the file.

![_](img/chmod.png)

Install Docker in the server

``` bash
$ ssh -i ~/.ssh/docker-server.pem ec2-user@3.64.127.118
$ whoami
$ sudo yum update
$ sudo yum install docker
$ sudo service docker start
$ ps aux | grep docker
$ sudo usermod -aG docker $USER
$ groups
$ exit
$ ssh -i ~/.ssh/docker-server.pem ec2-user@3.64.127.118
$ groups
```
### 1. Built and pushed Docker Image to your private DockerHub

``` bash
$ docker build -t react-nodejs-app:1.0 .
$ docker tag react-nodejs-app:1.0 flaviassantos/react-nodejs-app:1.0
$ docker login
$ docker push flaviassantos/react-nodejs-app:1.0
```

![_](img/dockerhub.png)

### 1. Docker installed and run Docker Image

``` bash
[ec2-user@ip-172-31-40-231 ~]$ docker login
$ ls -a
$ docker pull flaviassantos/react-nodejs-app:1.0
$ docker run -d -p 3000:3080 flaviassantos/react-nodejs-app:1.0 
```
While building app Docker Image I've faced the following issue:
"WARNING: The requested image's platform (linux/arm64) does not match the detected host platform (linux/amd64) and no specific platform was requested" 
As a result container fails and can't be started.
The issue is connected to the fact that docker image was built on MacOS M1 
and solution to overcome the issue is changing the Dockerfile in the following way:
```
-FROM node:10 AS ui-build
+FROM --platform=linux/amd64 node:10 AS ui-build
...
-FROM node:10 AS server-build
+FROM --platform=linux/amd64 node:10 AS server-build
```
![_](img/image.png)

### 1.Security Group configured: Opened Firewall to access Web App from Browser

![_](img/firewall.png) 

![_](img/apprunning.png)

## 2. CD - Deploy Application from Jenkins Pipeline to EC2 Instance (automatically with docker)

#### Technologies used:
AWS, Jenkins, Docker, Linux, Git, Java, Maven, Docker Hub

#### Project Description:
Prepare AWS EC2 Instance for deployment (Install Docker)
Create ssh key credentials for EC2 server on Jenkins
Extend the previous CI pipeline with deploy step to ssh into the
remote EC2 instance and deploy newly built image from Jenkins
server
Configure security group on EC2 Instance to allow access to our
web application

### 2.1 Installed SSH agent plugin on Jenkins and created ssh credentials type for EC2 on Jenkins

![_](img/ssh.png)

❏ Configured Jenkinsfile to use the sshAgent and execute docker run command
on EC2

![_](img/syntax.png)

``` groovy
def deployApp() {
    sshagent(['ec2-server-key']) {
        def dockerCmd = "docker run -p 3080:3080 -d flaviassantos/react-nodejs-app:2.0"
        sh "ssh -o StrictHostKeyChecking=no ec2-user@3.64.127.118 ${dockerCmd}"
}
}

return this
```
❏ Docker Login to DockerHub or your other private Docker Repository (if you
haven’t already)
❏ Security Group configured: Added Jenkins IP Address and opened port to
access Web App from Browser

![_](img/ip.png)

❏ Deploy Webapp on EC2 Instance by executed Multi-Branch Pipeline

![_](img/pipeline.png)

❏ Access Application on port 3080 in the browser

![_](img/app.png)

### 2.2 Deploy Java Maven App via Jenkins Pipeline on EC2 Instance:

❏ Configured Jenkinsfile to build and deploy on EC2 Instance

![_](img/file.png)
 
❏ Executed Multi-Branch Pipeline on Jenkins

![_](img/multi.png) 

## 3. CD - Deploy Application from Jenkins Pipeline on EC2 Instance (automatically with docker-compose)

#### Technologies used:
AWS, Jenkins, Docker, Linux, Git, Java, Maven, Docker Hub

### 3.1 Installed Docker-Compose on EC2 Instance

``` bash
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
$ docker-compose --version

```
❏ Created docker-compose.yaml file

``` 
version: '3.8'
services:
    java-maven-app:
      image: flaviassantos/my-repo:jma-3.0
      ports:
        - 8080:8080
    postgres:
      image: postgres:13
      ports:
        - 5432:5432
      environment:
        - POSTGRES_PASSWORD=my-pwd
```
❏ Configured Jenkinsfile (via script.groovy) to execute docker-compose command

```
def deployApp() {
    sshagent(['ec2-server-key']) {
        echo 'deploying docker image to EC2...'
        
        def dockerComposeCmd = "docker-compose -f docker-compose.yaml up --detach"
        sh "scp docker-compose.yaml ec2-user@3.64.127.118:/home/ec2-user"
        sh "ssh -o StrictHostKeyChecking=no ec2-user@3.64.127.118 ${dockerComposeCmd}"
}
}
return this
```

❏ Executed Jenkins Pipeline and deploy to AWS EC2 Instance

![_](img/deploy.png) 

❏ Improvement: Extract to Shell Script

![_](img/improv.png) 


## 4. Complete the CI/CD Pipeline (Docker-Compose, Dynamic versioning)

#### Technologies used:
AWS, Jenkins, Docker, Linux, Git, Java, Maven, Docker Hub

### 4.1 Adjusted Jenkinsfile and script.groovy to include dynamic versioning

- script.groovy:
``` groovy
def incrementVersion(){
    echo 'incrementing app version...'
    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
    def version = matcher[0][1]
    env.IMAGE_NAME = "flaviassantos/my-repo:$version-$BUILD_NUMBER"
}
return this

def commitVersionUpdate(){
    script {
        withCredentials([usernamePassword(credentialsId: 'github-token-as-pwd', passwordVariable: 'GITHUB_TOKEN', usernameVariable: 'USER')]) {
            // git config here for the first time run
            sh 'git config --global user.email "jenkins@example.com"'
            sh 'git config --global user.name "jenkins"'

            // using github token (as Jenkins username-password type)
            sh "git remote set-url origin https://github.com/${USER}/java-maven-app.git"
            sh "git config credential.helper 'store --file ~/.git-credentials'"
            sh "echo \"https://github.com:${GITHUB_TOKEN}@github.com\" >> ~/.git-credentials"

            sh 'git add .'
            sh 'git commit -m "ci: version bump"'
            sh 'git push origin HEAD:master'
        }
    }
}
return this
```

- Jenkinsfile:

``` groovy
...
        stage('increment version') {
            steps {
                script {
                    gv.incrementVersion()
                }
            }
        }
...
        stage('commit version update'){
            steps {
                script {
                    gv.commitVersionUpdate()
                }
            }
        }
```

### 4.2 Executed Jenkins Pipeline and deploy to AWS EC2 Instance

![_](img/newjenkins.png)  

## 5. Interacting with AWS CLI

#### Technologies used:
AWS, Linux

#### Project Description:

- Install and configure AWS CLI tool to connect to our AWS account
- Create EC2 Instance using the AWS CLI with all necessary configurations like Security Group
- Create SSH key pair
- Create IAM resources like User, Group, Policy using the AWS CLI
- List and browse AWS resources using the AWS CLI

### 5.1 Using EC2 commands:

❏ Created Security Group

``` bash
## List all available security-group ids
$ aws ec2 describe-security-groups

## create new security group
$ aws ec2 describe-vpcs
$ aws ec2 create-security-group --group-name my-sg --description "My security group" --vpc-id [vpc-0a76966266f11dcea]

## this will give output of created my-sg with its id, so we can do:
$ aws ec2 describe-security-groups --group-ids sg-0e6b934da2fa7a947

## add firewall rule to the group for port 22
$ aws ec2 authorize-security-group-ingress --group-id sg-0e6b934da2fa7a947 --protocol tcp --port 22 --cidr 92.221.241.143/32
$ aws ec2 describe-security-groups --group-ids sg-0e6b934da2fa7a947
```

❏ Created SSH key pair

``` bash
# Use an existing key-value pair or if you want, create and use a new key-pair. 'KeyMaterial' gives
us an unencrypted PEM encoded RSA private key.
$ aws ec2 create-key-pair --key-name MyKeyPair --query 'KeyMaterial' --output text > MyKeyPair.pem
```

❏ Created EC2 Instance

![_](img/ami.png)  

``` bash
# launch ec2 instance in the specified subnet of a VPC
$ aws ec2 describe-subnets
$ aws ec2 describe-instances -> will give us ami-imageid, we will use the same one
$ aws ec2 run-instances --image-id ami-0adbcf08fdd664fed --count 1 --instance-type t2.micro --key-name MyKeyPair --security-group-ids sg-0e6b934da2fa7a947 --subnet-id subnet-051dbf1284868ef89

# get the public IP address
$ aws ec2 describe-instances --instance-ids i-0250a8fe0d5ff7197
```

❏ SSHed into newly created EC2 Instance

``` bash
# ssh into the ec2 instance with the new key pem after creating it - public IP will be returned as json, so query it
$ chmod 400 ~/MyKeyPair.pem
$ ssh -i MyKeyPair.pem ec2-user@public-ip
```
![_](img/myinstance.png)  

❏ Used filter and query options

### 5.2 Using IAM commands:

❏ Created User and created Group

❏ Added User to Group

``` bash
# same way as ec2 had a bunch of commands for components relevant for ec2 instances, iam does too
$ aws iam create-group --group-name MyIamGroup
$ aws iam create-user --user-name MyUser
$ aws iam add-user-to-group --user-name MyUser --group-name MyIamGroup

# verify that my-group contains the my-user
$ aws iam get-group --group-name MyIamGroup
```

❏ Assigned policy to Group

``` bash
# attach policy to group
## this is the command so we need the policy-ARN
$ aws iam attach-user-policy --user-name MyUser --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
$ aws iam attach-group-policy --group-name MyIamGroup --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess

# validate policy attached to group or user
$ aws iam list-attached-group-policies --group-name MyIamGroup 
$ aws iam list-attached-user-policies --user-name MyUser
```

❏ Created credentials for new User

``` bash
# Now that user needs access to the command line and UI, but we didn't give it any credentials. So
let's do that as well!
## UI access
$ aws iam create-login-profile --user-name MyUser --password MyUser1Login8P@ssword --password-reset-required
# MyUserLoginP@ssword

$ aws iam get-user --user-name MyUser

```

❏ Created a new Policy and assigned to newly created Group

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iam:GetAccountPasswordPolicy",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "iam:ChangePassword",
      "Resource": "arn:aws:iam::account-id:user/${aws:username}"
    }
  ]
}
```

``` bash
# Create test policy
$ aws iam create-policy --policy-name changePwd --policy-document file://./changePwdPolicy.json
$ aws iam attach-group-policy --group-name MyIamGroup --policy-arn arn:aws:iam::716187863110:policy/changePwd
```

❏ Logged in with new User in AWS UI

![_](img/ui.png)   

❏ Created access keys for newly created User

``` bash

# cli access
$ aws iam create-access-key --user-name MyUser
-> you will see the access keys and use them in the next step

## Now let's ssh into the EC2 instance with this user
## 'aws configure' with new user creds overwriting the existing config
$ aws configure set aws_access_key_id default_access_key
$ aws configure set aws_secret_access_key default_secret_key

# or just for trying some things for MyUser in the current session:
$ export AWS_ACCESS_KEY_ID=AKIA2NQA7RBDA2T3QATD
$ export AWS_SECRET_ACCESS_KEY=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
$ export AWS_DEFAULT_REGION=eu-central-1

# test: user not allowed
$ aws iam create-user --user-name test
```