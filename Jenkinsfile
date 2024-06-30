pipeline{
    agent any
    environment{
        SONAR_HOME = tool "Sonar"
    }
    stages{
        
        stage("code"){
            steps{
                git url:"https://github.com/sanchit1498/cicdnodejsApp" , branch:"master"
                echo "cloned sucess"
            }
        }
        
        stage("Sonarqube analysis"){
            steps{
              withSonarQubeEnv("Sonar"){
                 sh "$SONAR_HOME/bin/sonar-scanner -Dsonar.projectName=nodetodo -Dsonar.projectKey=nodetodo -X"
              }
            }
        }
        
        stage("SonarQube Quality Gates"){
            steps{
              timeout(time: 1, unit:"MINUTES"){
                waitForQualityGate abortPipeline: false
              }
            }
        }
        
        stage("OWASP"){
            steps{
               dependencyCheck additionalArguments: '--scan ./' , odcInstallation: 'OWASP'
               dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage("build and test"){
            steps{
                sh 'docker build -t node-app-batch-6:latest .'
               echo "build sucess"
            }
        }
         stage("Trivy"){
            steps{
              sh "trivy image node-app-batch-6"
            }
        }
     
        
        stage("Push to Private Docker Hub Repo"){
            steps{
                withCredentials([usernamePassword(credentialsId:"DockerHubCreds",passwordVariable:"dockerPass",usernameVariable:"dockerUser")]){
                sh "docker login -u ${env.dockerUser} -p ${env.dockerPass}"
                sh "docker tag node-app-batch-6:latest ${env.dockerUser}/node-app-batch-6:latest"
                sh "docker push ${env.dockerUser}/node-app-batch-6:latest"
                }
            }
        }
        
        stage("deploy"){
            steps{
               sh "docker compose down && docker compose up -d"
                 echo "deployed sucessfully"
            }
        }
    }
}
