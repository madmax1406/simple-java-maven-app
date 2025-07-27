# Jenkins Pipeline Project

This README documents the setup and implementation of a Jenkins pipeline using a master-agent architecture on AWS EC2 instances. It includes raw, honest observations, tips, blockers, and lessons learned from building a functional CI/CD pipeline.This repo is forked from jenkins tutorial page and i do not own the code

---

## 1. Jenkins Master Node Setup

* **Instance type:** T2 Micro (used due to AWS Free Tier). *Not recommended* due to CPU utilization issues. A larger instance would perform better.
* **OS:** Ubuntu
* **Software Installed:** Latest JDK and Jenkins (via APT per official Jenkins docs).
* **Jenkins Running Mode:** Started via command line (not best practice). Can be improved by running Jenkins as a `systemd` service.
* **Ports Opened in Security Group:**

  * 22 (SSH)
  * 25 (SMTP)
  * 8080 (Jenkins UI)

---

## 2. Dev and Prod Agent Node Setup

* **Instance type:** Same as master (T2 Micro)
* **OS:** Ubuntu
* **Software Installed:** JDK
* **Agent Connection Method:** Connected to Jenkins controller using `java -jar agent.jar -jnlpUrl ...`
* **Remote Working Directory:** `/home/ubuntu/jenkins`
* **Node Labels:** `devserver`, `prodserver`
* **Challenges:**

  * Initially didn’t keep agent JAR process running, causing disconnects.
  * Solved using `systemd` and later with `nohup`.
  * Faced permission issues for `/home/ubuntu/jenkins`.

---

## 3. Jenkins File Setup - Parameters and Tools

```groovy
pipeline {
    tools {
        maven 'mymaven'
    }

    parameters {
        choice choices: ['Dev', 'Prod'], description: 'Select an Environment to proceed with deployment.', name: 'Environment'
    }
```

* **`mymaven`:** Defined in **Global Tool Configuration** in Jenkins.
* **Parameters:** Used a `choice` parameter so the user can choose between Dev and Prod during `Build with Parameters`.

---

## 4. Build and Test Stage Breakdown

### Build Stage:

```groovy
stage('Build') {
    steps {
        sh 'mvn clean package -DskipTests=true'
    }
}
```

* Used `-DskipTests=true` so tests can run in a separate stage.

### Test Stage (Parallel):

```groovy
stage('Test') {
    parallel {
        stage('Test-A') {
            agent { label 'devserver' }
            steps {
                echo 'This is Test A'
                sh 'mvn test'
            }
        }

        stage('Test-B') {
            agent { label 'devserver' }
            steps {
                echo 'This is Test B'
                sh 'mvn test'
            }
        }
    }
    post {
        success {
            dir("target/") {
                stash includes: '*.jar', name: 'maven-build'
            }
        }
    }
}
```

* Tests run in parallel as a demo setup.
* Stash used to save JAR only if tests succeed.

---

## 5. Deploy Stage (Dev & Prod)

### Deploy to Dev

```groovy
stage('Deploy_Dev') {
    when {
        expression { params.Environment == 'Dev' }
        beforeAgent true
    }
    steps {
        dir("${env.WORKSPACE}/artifacts/dev") {
            unstash 'maven-build'
            sh 'java -jar my-app-1.0-SNAPSHOT.jar'
        }
    }
}
```

### Deploy to Prod

```groovy
stage('Deploy_Prod') {
    when {
        expression { params.Environment == 'Prod' }
        beforeAgent true
    }
    agent { label 'prodserver' }
    steps {
        timeout(time: 5, unit: 'DAYS') {
            input message: 'Deployment approved?'
        }
        dir("${env.WORKSPACE}/artifacts/prod") {
            unstash 'maven-build'
            sh 'java -jar my-app-1.0-SNAPSHOT.jar'
        }
    }
}
```

* Used `when` conditions to route to proper deploy stage.
* Initially faced issues with inaccessible directories.
* Solved by using subdirectories under Jenkins workspace.
* Approval gate added for prod deployment.
* Misnaming JAR in `sh` caused initial failures.

---

## 6. Docker and Kubernetes (Planned)

Though not yet implemented, plan is:

* **Dockerfile:** In project root, using a minimal Java image, copying the JAR and running it.
* **Push to DockerHub:** Jenkins stage using credentials.
* **K8s Deployment:** YAML with deployment and service using image from DockerHub.
* **CI/CD Goal:** Trigger image build and deploy via single `Build Now` action.

---

## 7. Blockers and Learnings

* Agent disconnects due to not running `agent.jar` in background → Solved using `systemd` / `nohup`.
* Directory access issues (permissions).
* Jar mismatch between filename generated and referenced.
* Unstash failed due to inaccessible dirs.
* Solutions:

  * Followed official docs
  * Used ChatGPT
  * Adjusted directory structure
  * Declarative syntax via Jenkins UI helped generate accurate code.

### If Rebuilding Again:

* Would include security improvements.

### Final Tips:

> “Don't stop learning. If things don’t work, try debugging, use trial and error. Stay open-minded.”

---


If you found this helpful, feel free to fork or star ⭐ the repository.
