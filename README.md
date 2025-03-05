# GitLab Maven Project

To build a Maven project, run a SonarQube scan, build a Docker image, push the image to Docker Hub, and finally deploy and run the image on an Amazon EKS (Elastic Kubernetes Service) cluster via a GitLab CI/CD pipeline, you can follow these steps.

___

# Prerequisites

1. GitLab Project: You need a GitLab project where the Maven project code resides.
2. SonarQube: You need to have SonarQube set up and a SonarQube token (can be saved as GitLab CI/CD variables).
3. Docker Hub Account: Ensure you have Docker Hub credentials stored in GitLab CI/CD variables.
4. EKS Cluster: Ensure you have an Amazon EKS cluster up and running, and you can connect to it using kubectl.
5. AWS IAM: Ensure your GitLab CI runner has appropriate permissions to interact with AWS EKS (via IAM roles and policies).

## First create EC2 instance and install the required software on it - 

```
Install apache maven >> sudo apt install maven -y

Install Docker engine >> sudo apt install docker.io -y

Install Openjdk >> sudo apt install openjdk-11-jdk -y

Install gitlab runner >>

curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner -y

Install AWS CLI >>

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" 
sudo apt install unzip
unzip awscliv2.zip 
sudo ./aws/install
aws --version

Run the sonarqube docker container >> docker run -d -p 9000:9000 sonarqube

```

## Option -1 with Gitlab account 

Go to https://gitlab.com/users/sign_up to create Gitlab account. 

Once the account is created, you can use for CI-CD pipeline deployment with various stages, tools and plugins. 

Create project on the created gitlab account to configure Ci-CD jobs and pipelines. 

Once the gitlab account is created, create a new project 

## Register gitlab runner with the above created EC2 instance 
 
1. First got to gitlab account >> project >> setting >> CI- CD >> search for runners >> expant and click on new runner

2. Select runner platform Linux >> give name of the runner >> click on create runner

3. You will get gitlab token for runner registretion, follow the commands on the Ec2 instance in order to regiter it with gitlab as a runner worker

```
To register runner >> gitlab-runner register  --url https://gitlab.com  --token glrt-tYK95Ynj6qTBsLtw6vbM

Select the executor when it will prompt for >> shell

To run the runner once registration >> gitlab-runner run

```

## Option -2 with self hosted Gitlab on VM / EC2 system

1. Createa Ubuntu VM / EC2 instance and install the docker engine - run sudo apt install docker.io
2. Run the below mentioned command to pull gitlab community edition docker image and to spinup the container 

```
docker run --detach \
  --hostname gitlab.example.com \
  --publish 443:443 --publish 80:80 --publish 22:22 \
  --name gitlab \
  --restart always \
  --volume /srv/gitlab/config:/etc/gitlab \
  --volume /srv/gitlab/logs:/var/log/gitlab \
  --volume /srv/gitlab/data:/var/opt/gitlab \
  gitlab/gitlab-ce:latest

```

3. Run the below mentioned command to pull gitlab runner docker image and to spinup the container

```
docker run -d --name gitlab-runner --network host \
  --volume /srv/gitlab-runner/config:/etc/gitlab-runner \
  gitlab/gitlab-runner:latest

```

4. Register gitlab runner

```
docker exec -it gitlab-runner gitlab-runner register

You'll need the following information to register the runner:

GitLab URL: The address of your GitLab instance (e.g., http://gitlab.example.com). >> Use http://localhost:80
GitLab registration token: Obtain it from your GitLab project under Settings > CI/CD > Runners.
Description: The name/description of the runner.
Tags: Optional tags to associate with the runner.
Executor: Choose an executor (e.g., docker).

```

5. Verify that the containers are running

```
sudo docker ps -a
```

6. Wait for GitLab to initialize (may take a few minutes)

```
echo "Waiting for GitLab to initialize..." sleep 60 # Adjust sleep time if needed
```
   
8. Retrieve GitLab root user password

```
if [ -f "/srv/gitlab/config/initial_root_password" ]; then
    echo "GitLab Root Login Credentials:"
    echo "Username: root"
    echo "Password: $(sudo cat /srv/gitlab/config/initial_root_password | grep Password | awk '{print $2}')"
else
    echo "Password file not found! Resetting root password..."
    sudo docker exec -it gitlab gitlab-rails console <<EOF
user = User.find_by_username('root')
user.password = 'NewSecurePassword'
user.password_confirmation = 'NewSecurePassword'
user.save!
exit
EOF
    echo "Password reset to: NewSecurePassword"
fi

```
9. Display GitLab login URL

```

echo "Access GitLab at: http://$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)/" /### Replcae the IP address from above
```

## Clone the Gitlab project reposiroty to upload the project 

Once the Ec2 instance is ready to work as a gitlab runner worker,

then clone the gitlab project with your local system from where you need to upload your project data for CI-CD piple build. 

Copy the gitlab project url >> Got to Gitlab project >> Find the Clone with HTTPS option >> copy the URL and clone with your EC2 instance or local system from where you want to upload the project data. 

```

To clone the gitlab repository >> git clone https://gitlab.com/anand40090/springboot.git
It will craete the springboot folder in your system, save your project data into it and keep it ready to be uploaded on gitlab reposiroty.

To add your project data to commit on gitlab reposiorty >> go to project springboot project folder >> git add *

To commit the project data on gitlab repository >> git commit -m '1st commit'

To push the data to gitlab repository >> git push >> input username password when it will prompt for authentication 
 
```

## Configure sonarqube to intigrate with Gitlab 

1. At very bigining we have already downloaded the sonarqube docker image and spined the container of port 9000, by running commnd "docker run -d -p 9000:9000 sonarqube"
2. Run docker ps -a coommand on the Ubuntu system to check the docker container status of Sonarqube
3. Use the EC2 Ubuntu instance ip address:9000 in browser to access the sonarqube dashboard
4. Default username and password for sonarqube is admin / admin
5. Generate the token from sonarqube for integration with gitlab >> Login sonarqube >> Go to My Account >> Security >> Generate Token
6. Save the generated token to be hardcode in Gitlab variable

## Intigrate Sonarqube with Gitlab 

1. Login to gitlab account >> go to project >> go to setting >> CI-CD >> Find for variables >> Expand
2. Create two variable - 1] SONAR_TOKEN >> Input the token generated from sonqrqube 2] SONAR_HOST_URL >> Input your sonarqube URL (http://13.126.187.216:9000/)

## Create AWS ECR and IAM User with Secret and Access key to push to Docker Image on ECR

1. Create ECR reposiroty >> Go to AWS Dashboard >> Go to ECR >> Create Reposiorty
2. Create AWS User to authenticate the ECR and push docker image to ECR >> Go to AWS Dashboard >> IAM User >> Create User >> Download the Secret Key and Access Key of the user
3. Create Variable in Gitlab to call during CI-CD pipeline >>  Go to Gitlab account >>  go to project >> go to setting >> CI-CD >> Find for variables >> Expand
4. Create 1] AWS_Access_KEY_ID 2] AWS_Default_Region 3] AWS_Secret_access_key

## Create Variables in the Gitlab 

Before creating the .gitlab-ci.yml, add the following variables in GitLab project settings (Settings > CI / CD > Variables):

1. SONARQUBE_URL: The URL of your SonarQube instance (e.g., http://sonarqube.local).
2. SONARQUBE_TOKEN: Your SonarQube token.
3. DOCKER_REGISTRY: The Docker registry (e.g., docker.io).
4. DOCKER_USERNAME: Your Docker Hub username.
5. DOCKER_PASSWORD: Your Docker Hub password or access token.
6. AWS_ACCESS_KEY_ID: AWS Access Key ID for EKS access.
7. AWS_SECRET_ACCESS_KEY: AWS Secret Access Key for EKS access.
8. AWS_DEFAULT_REGION: AWS region where your EKS cluster is hosted (e.g., us-east-1).
9. ECR_REPO_NAME: The ECR repository name where the image will be pushed (optional if pushing to DockerHub).
10. KUBECONFIG: The kubeconfig file to access the EKS cluster (can be configured using aws eks update-kubeconfig).

## Create .gitlab-ci.yml file in the gitlab project reposiroty to run the CI-CD pipeline 

Got to gitlab project >> create the .gitlab-ci-.yml file 

Once this file is created and commited over the gitlab project, it will trigger the CI-CD pipeline and start the stages as mentioned in the file. 

Here we are doing 4 stage CI-CD pipelien - 

1. Build the maven project
2. Test the built maven project with sonarscanner
3. Build the docker image from the JAR file which is created while maven project build
4. Upload the docker image to AWS ECR

```
stages:
  - build
  - sonar_scan
  - docker_build
  - docker_push
  - deploy

before_script:
  - echo "Setting up Docker and AWS CLI"
  - apk add --no-cache curl git bash python3 py3-pip
  - pip3 install awscli
  - aws --version
  - echo "$AWS_ACCESS_KEY_ID" | aws configure set aws_access_key_id
  - echo "$AWS_SECRET_ACCESS_KEY" | aws configure set aws_secret_access_key
  - echo "$AWS_DEFAULT_REGION" | aws configure set region
  - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

# Build the Maven project
build_maven:
  image: maven:3.8.5-jdk-11  
  stage: build
  script:
    - echo "Building Maven project"
    - mvn clean install $MAVEN_CLI_OPTS

# Run SonarQube analysis
sonar_scan:
  stage: sonar_scan
  script:
    - echo "Running SonarQube scan"
    - mvn sonar:sonar -Dsonar.host.url=$SONARQUBE_URL -Dsonar.login=$SONARQUBE_TOKEN

# Build the Docker image
docker_build:
  stage: docker_build
  script:
    - echo "Building Docker image"
    - docker build -t $DOCKER_USERNAME/$DOCKER_IMAGE_NAME:$CI_COMMIT_REF_NAME .

# Push the Docker image to Docker Hub
docker_push:
  stage: docker_push
  script:
    - echo "Pushing Docker image to Docker Hub"
    - docker push $DOCKER_USERNAME/$DOCKER_IMAGE_NAME:$CI_COMMIT_REF_NAME
# Deploy the Docker image to EKS
deploy_to_eks:
  stage: deploy
  script:
    - echo "Setting up Kubernetes configuration for EKS"
    - aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_DEFAULT_REGION
    - kubectl get svc
    - echo "Deploying Docker image to EKS"
    - kubectl set image deployment/$KUBERNETES_DEPLOYMENT_NAME $KUBERNETES_DEPLOYMENT_NAME=$DOCKER_USERNAME/$DOCKER_IMAGE_NAME:$CI_COMMIT_REF_NAME
    - kubectl rollout status deployment/$KUBERNETES_DEPLOYMENT_NAME
  only:
    - main  # Run this job only on the main branch (adjust if needed)    



```


