@Library('slack') _

pipeline {
  agent any

  environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "hatamoni/numeric-app:${GIT_COMMIT}"
    applicationURL = "http://devsecops-hix-demo.westeurope.cloudapp.azure.com"
    applicationURI = "/increment/99"
  }

  stages {
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar'
            }
      }

      stage('Unit Test - JUnit and Jacoco') {
            steps {
              sh "mvn test"
            }
      }

      stage('Mutation Tests - PIT') {
            steps {
              sh "mvn org.pitest:pitest-maven:mutationCoverage"
            }
      }

      stage('SonarQube - SAST') {
          steps {
              withSonarQubeEnv('SonarQubeAzure') {
                sh "mvn sonar:sonar -Dsonar.projectKey=DevSecOpsDemoAzure -Dsonar.host.url=http://devsecops-hix-demo.westeurope.cloudapp.azure.com:9000"
              }
          
              timeout(time: 2, unit: 'MINUTES') {
                  script {
                    waitForQualityGate abortPipeline: true
                  }
              }
          }
      }

      //    stage('Vulnerability Scan - Docker ') {
      //      steps {
      //         sh "mvn dependency-check:check"   
      //        }
      // }

      stage('Vulnerability Scan - Docker') {
        steps {
          parallel(
            "Dependency Scan": {
              sh "mvn dependency-check:check"
            },
            "Trivy Scan": {
              sh "bash trivy-docker-image-scan.sh"
            },
            "OPA Conftest": {
              sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego Dockerfile'
            }
          )
        }
      }

      stage('Docker Build and Push') {
            steps {
              // This step should not normally be used in your script. Consult the inline help for details.
              withDockerRegistry([credentialsId: 'hatamoniDockerhub', url: ""]) {
                sh 'printenv'
                sh 'sudo docker build -t hatamoni/numeric-app:""$GIT_COMMIT"" .'
                sh 'docker push hatamoni/numeric-app:""$GIT_COMMIT""'
              }
            }
      }

      // stage('Vulnerability Scan - Kubernetes') {
      //   steps {
      //     sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
      //   }
      // }

      stage('Vulnerability Scan - Kubernetes') {
        steps {
          parallel(
            "OPA Scan": {
              sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
            },
            "Kubesec Scan": {
              sh "bash kubesec-scan.sh"
            }
          )
        }
      }

      // stage('Kubernetes Deployment - DEV') {
      //       steps {
      //         withKubeConfig([credentialsId: 'kubeconfig']) {
      //           sh "sed -i 's#replace#hatamoni/numeric-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
      //           sh "kubectl apply -f k8s_deployment_service.yaml"
      //         }
      //       }
      // }

      stage('K8S Deployment - DEV') {
        steps {
          parallel(
            "Deployment": {
              withKubeConfig([credentialsId: 'kubeconfig']) {
                sh "bash k8s-deployment.sh"
              }
            },
            "Rollout Status": {
              withKubeConfig([credentialsId: 'kubeconfig']) {
                sh "bash k8s-deployment-rollout-status.sh"
              }
            }
          )
        }
      }

      stage('Integration Tests - DEV') {
        steps {
          script {
            try {
              withKubeConfig([credentialsId: 'kubeconfig']) {
                sh "bash integration-test.sh"
              }
            } catch (e) {
              withKubeConfig([credentialsId: 'kubeconfig']) {
                sh "kubectl -n default rollout undo deploy ${deploymentName}"
              }
              throw e
            }
          }
        }
      }

      // stage('OWASP ZAP - DAST') {
      //   steps {
      //     withKubeConfig([credentialsId: 'kubeconfig']) {
      //       sh 'bash zap.sh'
      //     }
      //   }
      // }
  }

  post {
      always {
        junit 'target/surefire-reports/*.xml'
        jacoco execPattern: 'target/jacoco.exec'
        pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
        dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
        //publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'owasp-zap-report', reportFiles: 'zap_report.html', reportName: 'OWASP ZAP HTML Report', reportTitles: 'OWASP ZAP HTML Report'])
      
        sendNotification currentBuild.result

      }
  }
  
}