# CI/CD Pipeline

This document describes a practical CI/CD pipeline: Git → Jenkins → Maven → Docker → Docker Hub → Kubernetes

## 1. Project structure

project/
├── src/main/java/com/demo/App.java
├── pom.xml
├── Dockerfile
├── deployment.yaml

## 2. Minimal Java app (Spring Boot)

package com.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.*;

@SpringBootApplication
@RestController
public class App {

    @GetMapping("/")
    public String home() {
        return "CI/CD Pipeline Working!";
    }

    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}

## 3. pom.xml (example)

<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>
  </parent>
  <groupId>com.demo</groupId>
  <artifactId>app</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>

## 4. Dockerfile

FROM eclipse-temurin:17-jdk

WORKDIR /app

COPY target/app-0.0.1-SNAPSHOT.jar app.jar

ENTRYPOINT ["java","-jar","app.jar"]

## 5. Kubernetes deployment (deployment.yaml)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: yourname/app:latest
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  type: NodePort
  selector:
    app: app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30009

## 6. Jenkins pipeline (Jenkinsfile)

pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "yourname/app:latest"
    }

    stages {

        stage('Clone Repository') {
            steps {
                git branch: 'main',
                url: 'https://github.com/your/repo.git'
            }
        }

        stage('Build with Maven') {
            steps {
                bat 'mvn clean package'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat "docker build -t %DOCKER_IMAGE% ."
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    bat """
                    docker login -u %DOCKER_USER% -p %DOCKER_PASS%
                    docker push %DOCKER_IMAGE%
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                bat 'kubectl apply -f deployment.yaml'
            }
        }
    }
}

## 7. Common commands

- Initialize git and push:

  git init
  git add .
  git commit -m "initial commit"
  git branch -M main
  git remote add origin <repo-url>
  git push -u origin main

- Maven build:

  mvn clean package

- Docker:

  docker build -t yourname/app:latest .
  docker push yourname/app:latest

- Kubernetes:

  kubectl apply -f deployment.yaml
  kubectl get pods
  kubectl get svc
  kubectl logs <pod-name>

## 8. Verification

Open in browser:

http://127.0.0.1:30009

## 9. Common errors & fixes

- Git not recognized: ensure Git is installed and on PATH.
- Maven dependency/cache errors: run `mvn -U clean package`.
- Dockerfile jar mismatch: confirm JAR name in `target/` and `COPY` path.
- Port already allocated: change `nodePort` or stop conflicting service.
- CrashLoopBackOff: inspect `kubectl logs` and fix app startup errors.
- Kubernetes auth issues: ensure kubeconfig and context are correct.

---

Document assembled from existing project notes.
