# zomato-clone-devsecops
STEP 1:- Instance Setup
OS Image is Ubuntu 26.04
Instance Type:- m7i-flex.large
Add your existing key pair
security groups:- port:- 8080, 443, 22, 9000, 3000, 80.


STEP 2:- Cloned the Project Repo
sudo git clone https://github.com/Aj7Ay/Zomato-Clone.git
cd Zomato-Clone


STEP 3:- Installed Java + Jenkins
Created the install script using the vi text editor:
vi jenkins.sh
sudo apt update -y
#sudo apt upgrade -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb jammy main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
                  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins

Then made it executable and ran it:
bash
sudo chmod 777 jenkins.sh
./jenkins.sh

Jenkins' 2023.key GPG key had expired — switched to the current key:
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

Installed Java 17 first, but the newer Jenkins release required Java 21 — installed it and switched the default:
bash
sudo apt install temurin-21-jdk -y
sudo update-alternatives --config java

Started and enabled Jenkins:
bash
sudo systemctl start jenkins
sudo systemctl enable jenkins


STEP 4:- Installed Docker
bash
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER

Later, also added the jenkins system user to the docker group (this was the fix for a "permission denied" error when Jenkins tried to run docker build):

bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins


STEP 5:- Ran SonarQube as a Docker Container
bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

Logged in with default admin/admin, changed password, generated a token, and connected it to Jenkins.

Two SonarQube issues came up and were fixed:

vm.max_map_count too low → Elasticsearch (SonarQube's search backend) failed to index. Fixed with:
bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
docker restart sonar
Server URL missing http:// prefix in Jenkins' SonarQube server config — the scanner was failing with "Illegal character in scheme name." Fixed by setting the URL to http://<ec2-ip>:9000 in Manage Jenkins → System.


STEP 6:- Installed Trivy
bash
vi trivy.sh
sudo chmod 777 trivy.sh
./trivy.sh

Same jammy codename override needed here too, since the repo doesn't support resolute yet.


STEP 7:- Jenkins Configuration
Installed plugins: Eclipse Temurin Installer, SonarQube Scanner, NodeJS Plugin, OWASP Dependency-Check, Docker/Docker Pipeline/Docker Commons/Docker API.
Configured tools under Manage Jenkins → Tools: JDK (jdk21), NodeJS (NodeJs 16.2.0).
Added SonarQube server config and webhook (http://<jenkins-ip>:8080/sonarqube-webhook/) pointing back at Jenkins.
Added a docker credential (DockerHub username esha27 + a Personal Access Token with Read & Write scope — the first token generated had insufficient scope and had to be regenerated).


STEP 8:- Resolved a Full Disk (the biggest blocker)
The original EBS volume was only 6.7–8GB, which filled up completely and caused:

SonarQube's Elasticsearch to go read-only ("flood stage disk watermark exceeded")
Jenkins' built-in node to go offline ("Disk space is below threshold")

Fixed by:

Resizing the EBS volume from 8GB → 30GB in the AWS Console (EC2 → Volumes → Modify Volume)
Growing the partition and filesystem:
bash
lsblk
sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/nvme0n1p1
df -h
Bringing the Jenkins node back online and forcing a monitor recheck under Manage Jenkins → Nodes.


STEP 9:- Built the Full Pipeline (incrementally, stage by stage)
pipeline{
    agent any
    tools{
        jdk 'jdk21'
        nodejs 'NodeJs 16.2.0'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/Aj7Ay/Zomato-Clone.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "docker build -t zomato ."
                       sh "docker tag zomato esha27/zomato:latest "
                       sh "docker push esha27/zomato:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image esha27/zomato:latest > trivy.txt"
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name zomato -p 3000:3000 esha27/zomato:latest'
            }
        }
    }
}



Along the way you fixed:

NodeJS tool name mismatch — pipeline said node16, actual configured tool was named NodeJs 16.2.0
OWASP Dependency-Check hanging indefinitely — NVD (vulnerability database) requires an API key or it rate-limits to a crawl; temporarily removed this stage to unblock everything else
Docker push failing on insufficient token scope — regenerated the DockerHub token with Read & Write permissions
All image/tag references pointed at the tutorial author's DockerHub (sevenajay) — swapped every reference to your own account (esha27)


STEP 10:- Result
App successfully deployed and reachable at:

http://54.145.53.170:3000

Confirmed working in the browser — the Zomato clone loads correctly.

