# CI for Spark Applications

This repository intends to demonstrate the usage of a
Continuous Integration Pipeline for Scala Apps to run on Apache spark

![Continuous Integration](diagrams/cicd-pipeline.png)

## Installing Continuous Integration Software

Jenkins was the CI tool choose to run these exercise.


## Adding Jenkins Pipeline

```
pipeline {
  agent { dockerfile true }
  stages {
    stage('Build') {
      steps {
        sh 'sbt clean compile'
      }
    }
    stage('Unit') {
      steps {
        sh 'sbt test'
      }
    }
    stage('Build Artifact') {
      steps {
        sh 'sbt package'
        archiveArtifacts artifacts: 'target/scala-2.11/*.jar', fingerprint: true
      }
    }
    stage('Integration Test') {
      steps {
        sh 'aws emr create-cluster --name "Add Spark Step Cluster" --release-label emr-5.6.0 --applications Name=Spark --ec2-attributes KeyName=myKey --instance-type m3.xlarge --instance-count 3 --steps Type=CUSTOM_JAR,Name="basicspark_${env.BUILD_NUMBER}",Jar="target/scala-2.11/basicspark_2.11-1.0.jar",ActionOnFailure=CONTINUE,Args=[spark-example,SparkPi,10] --use-default-roles '
      }
    }
  }
}

```
TO test the Jenkinsfile run:

```
JENKINS_CRUMB=`curl -u admin:admin "$JENKINS_URL/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,\":\",//crumb)"`
curl -u admin:admin  -X POST -H $JENKINS_CRUMB -F "jenkinsfile=<Jenkinsfile" $JENKINS_URL/pipeline-model-converter/validate
```


## Installing Docker on Jenkins

Add Jenkis to the Docker Group
```
sudo usermod -aG docker jenkins
docker ps # check if it works
```

Configure Docker Service
```
vi (/usr/lib | /etc) /systemd/system/docker.service
ExecStart=/usr/bin/docker daemon -H unix:// -H tcp://localhost:2375
systemctl daemon-reload
systemctl restart docker
sudo service jenkins restart
```

## Configure AWS cli

Jenkins must be able to execute aws cicd-pipeline

```
sudo pip install awscli
aws Configure
```

## AWS EMR Spark

For running Spark as a service on AWS, make sure that you run:

```
aws emr create-default-roles
```

## Jenkins Jobs

### Backup Jobs
How to backup a Job configuration to an xml file
```
curl -s http://<user>:<password>@<jenkins_address>:<port>/job/<JOBNAME>/config.xml > jenkins-job.xml
```

### Import Job

How to create a Job from from a Backup
```
CRUMB=$(curl -s -u admin:admin "$JENKINS_URL/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,\":\",//crumb)")
curl -H "$CRUMB" -X POST  -u admin:admin "$JENKINS_URL/createItem?name=CICDSPARK" --header "Content-Type: application/xml" -d @jenkins-job.xml
