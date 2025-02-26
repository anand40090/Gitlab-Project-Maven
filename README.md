# GitLab Maven Project
This is gitlab maven project repository

In this lab we would be doing maven project intitrgaion with gitlab. To do this lab you would need -

- Gitlab Account
- AWS EC2 Ubuntu instance (t2.micro or t2.medium)
- One working Maven project
- AWS ECR reposiroty
- Sonarqube scanner

___

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

## Create Gitlab account 

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
    - test
    - sonarqube-check
    - docker_build

build_job:
    stage: build
    script:
        - mvn clean install

test_job:
    stage: test  
    script:
        - mvn test

sonarqube-check:
  stage: sonarqube-check   
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"  # Defines the location of the analysis task cache
    GIT_DEPTH: "0"  # Tells git to fetch all the branches of the project, required by the analysis task
  cache:
    key: "${CI_JOB_NAME}"
    paths:
      - .sonar/cache
  script: 
    - mvn verify sonar:sonar -Dsonar.projectKey=devops8004609_springboot-maven-project_AZU88yLLk0-28RVuG-tY
  allow_failure: true
  only:
    - main # or the name of your main branch

docker_build:
  stage: docker_build
  script:
    - docker build -t target . 
    - docker run -itd --name app-1 -p 8080:8080 target   
```
