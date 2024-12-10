pipeline  {
    agent any

    tools {
        jdk 'OpenJDK17'
    }
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }

    stages {
        stage('Clean Workspace') {
            steps {
                // Cleanup before starting the stage
                cleanWs()
            }
        }
        stage('Checkout') {
            steps {
                // Checkout the repository
                checkout scm
            }
        }
        stage('Main branch release') {
            when { 
                branch 'main' 
            }
            steps {
                echo "I am building on ${env.BRANCH_NAME}"
                sh "./gradlew clean build release -Drelease.dir=$JENKINS_HOME/repo.gecko/release/org.gecko.libraries --info --stacktrace -Dmaven.repo.local=${WORKSPACE}/.m2"
            }
        }
        stage('Snapshot branch release') {
            when { 
                branch 'snapshot'
            }
            steps  {
                echo "I am building on ${env.JOB_NAME}"
                sh "./gradlew clean release --info --stacktrace -Dmaven.repo.local=${WORKSPACE}/.m2"
                sh "mkdir -p $JENKINS_HOME/repo.gecko/snapshot/org.gecko.libraries"
                sh "rm -rf $JENKINS_HOME/repo.gecko/snapshot/org.gecko.libraries/*"
                sh "cp -r cnf/release/* $JENKINS_HOME/repo.gecko/snapshot/org.gecko.libraries"
            }
        }
    }

}
