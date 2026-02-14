pipeline {
    agent any

    environment {
        appname      = "backend"
        tag          = "v1"
        argocdPassUAT = credentials('argocd-pass-uat')
        DockerHubUser = credentials('dockerhub-user')
        branch       = "${GIT_BRANCH}"
        DockerRegistryUrl = "nettur19"
    }

    stages {

        stage('Build') {
            steps {
                sh '''
                    docker run --rm -v "$PWD:/app" -w /app maven:3.9.9-eclipse-temurin-21 mvn clean package -DskipTests
                '''
            }
        }


        stage('SonarQube') {
            steps {
                withSonarQubeEnv('sonar_server') {
                    sh '''#!/bin/bash
                        docker run --rm -u 0 -v "$PWD:/usr/src" -w /usr/src sonarsource/sonar-scanner-cli:latest sonar-scanner -Dsonar.projectKey=backend -Dsonar.sources=src/main -Dsonar.java.binaries=target/classes -Dsonar.host.url=${SONAR_HOST_URL} -Dsonar.login=${SONAR_AUTH_TOKEN} -Dsonar.branch.name=${GitVersion_BranchName} -Dsonar.qualitygate.wait=false
                    '''
                }
            }
        }


        stage('SCA trivy FS scan') {
            when {
                allOf {
                    expression { env.branch == 'main' || (env.branch).startsWith('release') }
                    environment name: 'IsRun', value: 'true'
                }
            }
            steps {
                script {
                    sh "trivy fs . --format template --template \"@/usr/local/share/trivy/templates/html.tpl\" -o trivyfsreport.html"
                    sh "ls -lart"
                }
            }

        }

        stage('trivy FS SBOM Generation and Scan') {
            when {
                allOf {
                    expression { env.branch == 'main' || (env.branch).startsWith('release') }
                    environment name: 'IsRun', value: 'true'
                }
            }
            steps {
                script {
                    sh "trivy fs . --format cyclonedx --timeout 600s -o trivyfsbomreport.json"
                    if (env.TrivyGateEnabled == "True") {
                        sh "trivy sbom trivyfsbomreport.json --exit-code 1 --severity HIGH,CRITICAL"
                    }
                }
            }
        }

        stage('Docker Image build') {
          steps {
                script {
                    sh "echo $DOCKER_PASS | docker login --username ${DockerHubUser} --password-stdin ${DockerRegistryUrl}"
                    if ((env.branch).startsWith('release')) {
                        env.Docker_registry = "${DockerRegistryUrl}/${appname}"
                        sh "docker build -t ${Docker_registry}:${GitVersion_SemVer} ."
                        sh "docker tag ${Docker_registry}:${GitVersion_SemVer} ${Docker_registry}:uat"
                    } else if (env.branch == 'main') {

                        sh "docker build -t ${Docker_registry}:${GitVersion_SemVer} ."
                        sh "docker tag ${Docker_registry}:${GitVersion_SemVer} ${Docker_registry}:prod"
                    }
                }
            }
        }

        stage('trivy image SBOM Generate') {
            when {
                allOf {
                    expression { env.branch == 'main' || (env.branch).startsWith('release') }
                    environment name: 'IsRun', value: 'true'
                }
            }
            steps {
                script {
                    sh "trivy image --format spdx-json -o result.json ${Docker_registry}:${GitVersion_SemVer}"
                }
            }
        }

        stage('trivy image scan') {
            when {
                allOf {
                    expression { env.branch == 'main' || (env.branch).startsWith('release') }
                    environment name: 'IsRun', value: 'true'
                }
            }
            steps {
                script {
                    sh "trivy image --timeout 20m --format template --template \"@/usr/local/share/trivy/templates/html.tpl\" -o trivyreport.html ${Docker_registry}:${GitVersion_SemVer} --timeout 25m "
                    if (env.TrivyGateEnabled == "True") {
                        sh "trivy image --no-progress --exit-code 1 --severity HIGH,CRITICAL ${Docker_registry}:${GitVersion_SemVer}"
                    }
                }
            }
            post {
                always {
                    emailext subject: 'Trivy-DigiLedge-MCA-Test-BackEnd-Report',
                             body: 'Trivy DigiLedge-MCA-Test-BackEnd report.',
                             attachmentsPattern: '**/trivyreport.html',
                             to: '$DEFAULT_RECIPIENTS'

                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: false,
                        reportDir: '',
                        reportFiles: 'trivyreport.html',
                        reportName: "Trivy Image Scan Report"
                    ])
                }
            }
        }

        stage('trivy image secrets scan') {
            when {
                allOf {
                    expression { env.branch == 'main' || (env.branch).startsWith('release') }
                    environment name: 'IsRun', value: 'true'
                }
            }
            steps {
                script {
                    sh "trivy image --format template --template \"@/usr/local/share/trivy/templates/html.tpl\" -o trivysecretreport.html ${Docker_registry}:${GitVersion_SemVer} --timeout 30m --no-progress --scanners secret --exit-code 1 --severity HIGH,CRITICAL" //--no-progress --exit-code 1 --severity HIGH,CRITICAL
                }
            }
             post {
                 always {
            //         emailext subject: 'Trivy-DigiLedge-MCA-Test-BackEnd-Secret-Report',
            //                  body: 'Trivy DigiLedge-MCA-Test-BackEnd Secret report.',
            //                  attachmentsPattern: '**/trivysecretreport.html',
            //                  to: '$DEFAULT_RECIPIENTS'
                       publishHTML(target: [
                         allowMissing: true,
                         alwaysLinkToLastBuild: true,
                         keepAll: false,
                         reportDir: '',
                         reportFiles: 'trivysecretreport.html',
                         reportName: "Trivy Image Secret Scan Report"
                     ])
                 }
             }
        }

        stage('Docker Image Push') {
            steps {
                script {
                    sh "echo $DOCKER_PASS | docker login --username AWS --password-stdin ${DockerRegistryUrl}"
                    sh "docker push ${Docker_registry}:${GitVersion_SemVer} || true"
                    sh "docker rmi -f ${Docker_registry}:${GitVersion_SemVer}"

                    if ((env.branch).startsWith('release')) {
                        sh "docker push ${Docker_registry}:uat || true"
                        sh "docker rmi -f ${Docker_registry}:uat"
                    }  else if (env.branch == 'main') {
                        sh "docker push ${Docker_registry}:prod || true"
                        sh "docker rmi -f ${Docker_registry}:prod"
                    }
                }
            }
            post {
                failure {
                    script {
                        emailext attachLog: true, body: '''$DEFAULT_CONTENT
    Job Details:
    Application: ${JOB_NAME}
    Branch Name: ${BRANCH_NAME}
    ---------------------------
    Docker image-build was failed
    Please check the Dockerfile
    --------------------------
    Code changes:
    ${CHANGES}''', subject: '$DEFAULT_SUBJECT', to: '$DEFAULT_RECIPIENTS'
                    }
                }
            }
        }


        stage('K8s UAT-Deploy') {
            when { branch '*release/**' }
            steps {
              sh "yes | argocd login ${ArgoCDUatUrl} --username admin --password ${argocdPassUAT}"
              sh "argocd app actions run MCA-Test-BackEnd restart --kind Rollout --resource-name ams-server-v1"
            }
            post {
                always {
                    jiraSendDeploymentInfo site: 'mintoak.atlassian.net',
                                           environmentId: 'App-cluster',
                                           environmentName: 'UAT',
                                           environmentType: 'testing'
                }
                success {
                    script {
                        emailext attachLog: true, body: '''$DEFAULT_CONTENT
    Job Details:
    Application: ${JOB_NAME}
    Branch Name: ${BRANCH_NAME}
    Deployment Server: app-cluster
    ---------------------------
    SonarQube Report generated (code analysis + code coverage)
    Sonar Quality Gate is Passed, check URL: ${SONAR_URL}

    Junit Test-Cases:
    Total tests = $TEST_COUNTS
    Passed = ${TEST_COUNTS,var="pass"}
    Failed = ${TEST_COUNTS,var="fail"}
    Test cases Results:
    ${FAILED_TESTS}
    --------------------------
    Code changes:
    ${CHANGES}''', subject: '$DEFAULT_SUBJECT', to: '$DEFAULT_RECIPIENTS'
                    }
                }
            }
        }



        stage('Clean Workspace After build') {
            when { not { branch 'main' } }
            steps {
                sh "ls -all"
                cleanWs()
                sh "ls -all"
                sh "pwd"
            }
        }
    }
}
