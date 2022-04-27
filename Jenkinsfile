pipeline {
  agent any

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

      stage('Vulnerability Scan - Docker ') {
          steps {
            sh "mvn dependency-check:check"
          }
      }

      stage('Docker Build and Push') {
            steps {
              // This step should not normally be used in your script. Consult the inline help for details.
              withDockerRegistry([credentialsId: 'hatamoniDockerhub', url: ""]) {
                sh 'printenv'
                sh 'docker build -t hatamoni/numeric-app:""$GIT_COMMIT"" .'
                sh 'docker push hatamoni/numeric-app:""$GIT_COMMIT""'
              }
            }
      }

      stage('Kubernetes Deployment - DEV') {
            steps {
              withKubeConfig([credentialsId: 'kubeconfig']) {
                sh "sed -i 's#replace#hatamoni/numeric-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
                sh "kubectl apply -f k8s_deployment_service.yaml"
              }
            }
      }

      post {
        always {
          junit 'target/surefire-reports/*.xml'
          jacoco execPattern: 'target/jacoco.exec'
          pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
          dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
        }

    }
  }
}