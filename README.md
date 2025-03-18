CI/CD Pipeline with Gitlab, Jenkins, Terraform and Development and Staging Server

This project outlines the setting up CI/CD Pipeline using Gitlab, Jenkins, Terraform,  Development Server, Staging server. Goal is to provision AWS infrastructure automatically using a fully automated pipeline.

Components (Main Server, Staging server , and Gitlab)
Main Server:
Install docker and docker-compose
# Update the package index
sudo apt-get update
# Install Docker
sudo apt-get install docker.io
sudo usermod -aG docker $USER
sudo systemctl enable docker --now 
                                    #Install Docker-Compose
sudo apt-get install docker-compose-v2
Install ngrok for exposing jenkins to the gitlab
#Install  Ngrok
curl -sSL https://ngrok.com/download | sudo unzip -d /usr/local/bin
ngrok http 8080
Install Jenkins on a Docker Container
#Install  Jenkins on a Docker Container
                        docker run -d -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /home/ubuntu/.ssh:/var/jenkins_home/.ssh \
  --user root \
  --name jenkins_container \
  --memory="2g" --cpus="1.5" \
  jenkins/jenkins:lts
(Note- specifying memory as 2g , and vcpu 1.5 means Jenkins server doesn’t overuse Host resource. This setup can be easily scaled up by adding more staging servers. Also, could even containerize the staging environment using Docker Compose or Kubernetes for larger infrastructures i.e we can run staging server or its services in a docker container using docker compose or kubernetes for better scalability, resource-management and consistency.Test from Jenkins Container: If needed, exec into the container and test  manually:
           docker exec -it jenkins_container_name /bin/bash 
           ssh <staging-server-username>@<staging-server-ip>
Docker will manage it’s own volume ‘jenkins_home’ which is mapped to the container’s /var/jenkins_home. This volume lives outside of the container and it is usually found under /var/lib/docker/volumes on the host. 
sudo -i
ls /var/lib/docker/volumes (gives jenkins_home directory)
drwx-----x  3 root root  4096 Mar 14 10:57 jenkins_home

ls /var/lib/docker/volumes/jenkins_home/_data 
(gives all the Jenkins Data, also contains  .ssh folder, because that folder is mounted to the container’s /var/jenkins_home.)

Inside the container, 
docker exec -it jenkins_container bash
ls /var/jenkins_home/
(gives all the Jenkins files and folders , and also the .ssh folder because the .ssh folder being mounted as a volume to /var/jenkins_home.)
 Jenkins inside Docker can access SSH keys under /var/jenkins_home/.ssh/.
 docker inspect jenkins_container | grep "User"
            "UsernsMode": "",
            "User": "root", 
The default ‘User’ is Jenkins. If we installed Jenkins with no –user root parameter, then the default User would be Jenkins who has no Permission to access Private Keys to make SSH  with the Staging Server.           






Now, Jenkins inside Docker can access SSH keys under /var/jenkins_home/.ssh/.
  docker inspect jenkins_container | grep "User"
            "UsernsMode": "",
            "User": "root",     
 


Staging Server:
Install docker and docker-compose-v2
Install SSH Server for jenkins to ssh into staging machine
sudo apt install openssh-server -y 
sudo systemctl enable –now openssh-server
                        sudo ss -tuln | grep 22 (ensure port 22 is open)
            sudo ufw allow ssh; sudo ufw enable; sudo ufw status (if port 22 is not              open)
                        #enable the following in /etc/ssh/sshd_config
                                    Port 22
                                    PermitRootLogin yes
                                    PubkeyAuthentication yes
                                    PasswordAuthentication yes
                                    sudo systemctl restart ssh




Gitlab:
Create a new project(terraform-eks) in gitlab 
Generate Personal Access Token: Click Profile > Preferences > Access Token > Generate Personal Access Token , (select expiration date also), select scopes as api > click ‘Create personal access token’. Save that token somewhere safe because we need it later while configuring the Jenkins in the main server.
Clone that repository into your main VM.
# On the Main (Development) VM
mkdir -p ~/projects
cd ~/projects
           git clone  https://gitlab.com/serverlessprojectgroup/terraform-eks.git
cd terraform-eks
           # Preparing Terraform Configuration
mkdir Terraform
cd Terraform
create  0-s3.tf file
# Push Code to gitlab
git add .
git commit -m "Add S3 bucket Terraform configuration"
git push origin main

 Install Terraform, AWS CLI, and GIT on the containers running in docker in staging server.



 Connection between Gitlab and Main VM
Generate SSH Key on Main VM
            ssh-keygen -t rsa -b 4096 
It generates a private key (~/.ssh/id_rsa) and a public key (~/.ssh/id_rsa.pub).
Copy the public key (cat ~/.ssh/id_rsa.pub) and paste it to your Gitlab’s profile > Preferences > SSH key  > and (paste) save that key. This setup is for authentication with the Gitlab.
Test the SSH Connection
ssh -T git@gitlab.com
You should see the output: Welcome to Gitlab, <username>  


Access Jenkins UI and configure Jenkins Server
Open  http://localhost:8080 or http://192.168.123.128:8080
Usually, initialAdminPassword is unlocked by sudo cat /var/lib/jenkins/secrets/initialAdminPassword command, however here Jenkins is installed via the jenkins-image , therefore the initialAdminPassword retrieving command sudo  /var/lib/jenkins/secrets/initialAdminPassword does not work in this case. Either retrieve the password from the Jenkins Server or from the container.
Retrieve the initialAdminPassword form the Jenkins Server
I have used jenkins-image to install Jenkins on my Main VM and mounted a volume which is mapped to a container. I can access the volume data via:
Inspect the Volume
docker inspect jenkins_home
#Produces output
[
  {
       "CreatedAt": "2025-03-14T10:57:53+11:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/jenkins_home/_data",
        "Name": "jenkins_home",
        "Options": null,
        "Scope": "local"
    }
]
Act as a root and retrieve an initialAdminPassword from the mounted volume
sudo -i
cat /var/lib/docker/volumes/jenkins_home/_data/secrets/initialAdminPassword

Retrieve initialAdminPassword from the container
docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED       STATUS       PORTS                                                                                          NAMES
78596d99f2ae   jenkins/jenkins:lts   "/usr/bin/tini -- /u…"   2 hours ago   Up 2 hours   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp, 0.0.0.0:50000->50000/tcp, [::]:50000->50000/tcp   jenkins_container
docker exec -it jenkins_container bash
                                               cat  /var/jenkins_home/secrets/initialAdminPassword
Finish the Installation of Jenkins 
After you paste your password, Follow the steps and install suggested plugins, then Create a user (username, password,confirm password, fullname, email).
Install Required Plugins
Click Manage Jenkins > Plugins > Available Plugins > search for Gitlab, Gitlab api, SSH Agent Plugin, Pipeline Stage View and install it. These plugins allow Gitlab to triggerJenkins for build and deployment.
Make Jenkins aware of the Gitlab plugin and Test the connection
Click Manage Jenkins > Configure System > Search for new section called “Gitlab” > give a connection name(gitlab-connection), gitlab host url as https://gitlab.com, and for credential create a new one, select Kind as Gitlab API token, and under API token section paste the Personal Access Token that you got from the Gitlab.If everything works well, you will see “Success” after testing a connection.




Setting up Jenkins Pipeline (Connection between Development Server and Staging Server)
 Generate keys in the main development server for accessing Staging Server
 ssh-keygen -t rsa -b 4096 -C “jenkins-staging-server-connection” -f ~/.ssh/id_rsa_jenkins
 To avoid confusion with the Gitlab private and public key, custom name the keys for jenkins as id_rsa_jenkins.pub and id_rsa_jenkins. We add a public key to the Staging server and private key to Jenkins.
Share the public key to the Staging Server
ssh-copy-id -i ~/.ssh/id_rsa_jenkins.pub <staging-server-username>:<staging-server-ip>

Verify/Test the SSH Connection to the Staging Server
ssh -i ~/.ssh/id_rsa_jenkins <staging-server-username>@<staging_server_ip>
Store the Private Key in the Jenkin’s Agent Node
It’s the Jenkins container that makes ssh requests to the staging server, so we need to first create a new agent node and configure that node and add  the generated private key to that node.

Setting up Staging VM as an Agent Node in Jenkins
Register Staging VM as an agent
Go to Jenkins Dashboard > Manage Jenkins > Manage Nodes and Clouds.
 Click "New Node", name it staging-agent, and choose Permanent Agent and create.
Description: This is an ubuntu machine agent
Number of executors:1
Remote root directory:/home/ubuntu/jenkins-agent. Create a folder jenkins-agent in the Staging Server so that it becomes the Remote root directory.
Labels:staging-agent
Launch method:Launch agents via SSH
 Host:192.168.123.129 <ip address of Staging VM>
Credentials:staging-server-ssh  + Add > 
 Kind:SSH Username with private key
 ID:staging-server-ssh
 username:ubuntu
  Private Key: < content of  ~/.ssh/id_rsa_jenkins >
  Host Key Verification Strategy:Non verifying Verification Strategy  
  Go to Dashboard> Nodes>staging-agent(node name), Click Launch Agent  now in script - agent {label "staging-agent"} 













Setting up Jenkins PipelineJob
Open http://localhost:8080, Enter a New Item(Terraform-EKS-Pipeline), select item type as ‘Pipeline’ job name and press OK.
Configure the Job.
Add description, Gitlab Connection (give gitlab connection name), and Under Trigger, tick the box- ‘Build when a change is pushed to gitlab’.


Under Advanced Section, generate a secret token (we need that token to integrate with our project ‘terraform-eks’ in Gitlab)





 Setting Gitlab Webhook with Jenkins
Integrate Gitlab with Jenkins
 Select a project in Gitlab, Click Settings > webhook > and paste that token (which you have generated above) as a secret in the webhook section and also Trigger Push events.

Paste the url transformed by the ngrok
For the url we cannot use http://192.168.123.128:8080 coz it is local, so we need to use ngrok which transforms our local ip into public ip. URL would be http://<ngrok_url>/project/Terraform-EKS-Pipeline
How to generate the ngrok url?
In main VM, nhrok http 8080
Session Status                online                                                                                                                                                    
Account                       User’s Name(Plan: Free)                                                                                                                                   
Version                       3.20.0                                                                                                                                                    
Region                        Australia (au)                                                                                                                                            
Latency                       74ms                                                                                                                                                      
Web Interface                 http://127.0.0.1:4040                                                                                                                                     
Forwarding                    https://da96-14-202-101-171.ngrok-free.app -> http://localhost:8080                                                                                       
                                                                                                                                                                                        
Connections                   ttl     opn     rt1     rt5     p50     p90                                                                                                               
                              0       0       0.00    0.00    0.00    0.00    



Note Forwarding ‘https://da96-14-202-101-171.ngrok-free.app’ and paste it into the url of in the gitlab followed by /project/<Pipeline-Job-Name>.In my case, it is
                        https://da96-14-202-101-171.ngrok-free.app/project/Terraform-EKS-Pipeline


You can write a pipeline script, then apply and save. Now, your pipeline runs when you push your code in your main VM.


Containerize the Staging Environment
Create containers which have Terraform and AWS CLI installed . Create a Dockerfile that defines the container.
Use Docker Compose to manage and execute Terraform on a container.
Connect Jenkins to Staging Server and run commands inside containers. Write a Jenkinsfile to execute Terraform in a container

Overflow
User Updates Code in Main VM → Pushes to GitLab
GitLab triggers Jenkins (Main VM, 192.168.123.128)
Jenkins pulls latest code to Staging VM (192.168.123.129)
Jenkins starts the Terraform container using docker-compose
Terraform runs inside the container (instead of directly on the VM)
Terraform provisions AWS infrastructure.





