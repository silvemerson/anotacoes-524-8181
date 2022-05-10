## Primeiro Pipeline Declarativo

```groovy
pipeline {
  agent any
  
  stages{
    stage('Varaiveis de Ambiente'){
      steps{
        sh 'printenv'  
      }    
    }
  }    
  post{
      always {
          echo "Pipeline finalizado."
      }
      success {
          echo "Pipeline finalizado com Sucesso"
      }
      
      failure{
          echo "Pipeline finalizado com falha"
      }
  }    
}

```

## Pipeline com Váriaveis, options e parameters

```groovy
pipeline {
    agent any
    
    options{
        timeout(time: 5, unit: 'SECONDS')
        timestamps()
    }
    
    stages {
        stage('OS Release'){
            steps{
                sh 'cat /etc/*release'
                echo "$WORKSPACE"
                echo "$BUILD_ID"
                echo "$BUILD_DISPLAY_NAME"
            }
            
        }
        
        stage('Git Checkout gitTest'){
            steps{
                git 'http://192.168.56.20:3000/root/gitTest.git'
            }
        }
    }
}

```

# Docker Build

```groovy

pipeline{
    agent any

    environment {
            IMAGE_NAME="courseCatalog"
       } 

    stages{

        stage('Image Build'){
            steps{
                {
                    image = docker.build("$IMAGE_NAME")
                }
            }
       } 
    }
}

```

# teste unitários

```groovy
pipeline{
    agent any

    environment {
            IMAGE_NAME="course-catalog"
       } 

    stages{

        stage('Image Build'){
            steps{
                {
                    image = docker.build("$IMAGE_NAME")
                }
            }

       }
        stage('Executando Unit Test'){
            steps{
                {  
                    image.inside("-v ${WORKSPACE}:/courseCatalog"){
                        sh "nosetests --with-xunit --with-coverage --cover-package=project test_users.py"
                    }
                 }

            }
        }
    }
}
```

# SonarQube

```groovy
pipeline{
    agent any

    environment {
            IMAGE_NAME="course-catalog"
       } 

    stages{

        stage('Image Build'){
            steps{##
                {
                    image = docker.build("$IMAGE_NAME")
                }
            }

       }
        stage('Executando Unit Test'){
            steps{
                {  
                    image.inside("-v ${WORKSPACE}:/courseCatalog"){
                        sh "nosetests --with-xunit --with-coverage --cover-package=project test_users.py"
                    }
                 }

            }
        }
        stage('SonarQube'){
            steps{
                {
                    def scannerPath = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube'){
                        sh "${scannerPath}/bin/sonar-scanner -Dsonar.projectKey=courseCatalog -Dsonar.sources=."
                    }
                }
           }
       }
       stage('SonarQube Analysis Result'){
           steps{
                timeout(time: 30, unit: 'SECONDS'){
                    waitForQualityGate abortPipeline: true
                }
          }
      }
       
       
    }
    post{
      success {
        junit 'nosetests.xml'
       }
   }    
}

```

# Push Nexus

```groovy
pipeline{
    agent any

    environment {
            IMAGE_NAME="course-catalog"
            IMAGE_TAG="0.0.${BUILD_ID}"
            NEXUS_SERVER="192.168.56.20:8082"
            HTTP_PROTO="http://"
            JENKINS_CREDEN="710f295d-1bfb-43f4-a83c-2bbfe1ea61f1"
       } 

    stages{

        stage('Image Build'){
            steps{
                {
                    image = docker.build("$IMAGE_NAME:${IMAGE_TAG}")
                }
            }

       }
        stage('Executando Unit Test'){
            steps{
                {  
                    image.inside("-v ${WORKSPACE}:/courseCatalog"){
                        sh "nosetests --with-xunit --with-coverage --cover-package=project test_users.py"
                    }
                 }

            }
        }
/*        
        stage('SonarQube'){
            steps{
                {
                    def scannerPath = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube'){
                        sh "${scannerPath}/bin/sonar-scanner -Dsonar.projectKey=courseCatalog -Dsonar.sources=."
                    }
                }
           }
       }
       stage('SonarQube Analysis Result'){
           steps{
                timeout(time: 30, unit: 'SECONDS'){
                    waitForQualityGate abortPipeline: true
                }
          }
      }*/

       stage('Docker Push Nexus'){
           steps{
               {
                   docker.withRegistry("http://192.168.56.20:8082", "710f295d-1bfb-43f4-a83c-2bbfe1ea61f1"){
                   image.push()
               }
           }
       }
       
       
    }
}
    post{
      success {
        junit 'nosetests.xml'
       }
   }    
}
```

## Deploy em Homolo

```groovy
pipeline{
    agent any

    environment {
            IMAGE_NAME="course-catalog"
            IMAGE_TAG="0.0.${BUILD_ID}"
            NEXUS_SERVER="192.168.56.20:8082"
            HTTP_PROTO="http://"
            JENKINS_CREDEN="710f295d-1bfb-43f4-a83c-2bbfe1ea61f1"
            DOCKER_REGISTRY="${HTTP_PROTO}${NEXUS_SERVER}"
            DOCKER_HOMOLOG="tcp://192.168.56.30:2375"
       } 

    stages{

        stage('Image Build'){
            steps{
                {
                    image = docker.build("$IMAGE_NAME:${IMAGE_TAG}")
                }
            }

       }
        stage('Executando Unit Test'){
            steps{
                {  
                    image.inside("-v ${WORKSPACE}:/courseCatalog"){
                        sh "nosetests --with-xunit --with-coverage --cover-package=project test_users.py"
                    }
                 }

            }
        }
/*        
        stage('SonarQube'){
            steps{
                {
                    def scannerPath = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube'){
                        sh "${scannerPath}/bin/sonar-scanner -Dsonar.projectKey=courseCatalog -Dsonar.sources=."
                    }
                }
           }
       }
       stage('SonarQube Analysis Result'){
           steps{
                timeout(time: 30, unit: 'SECONDS'){
                    waitForQualityGate abortPipeline: true
                }
          }
      }*/

       stage('Docker Push Nexus'){
           steps{
               {
                   docker.withRegistry("http://192.168.56.20:8082", "710f295d-1bfb-43f4-a83c-2bbfe1ea61f1"){
                   image.push()
               }
           }
       }
       
       
    }
    stage('Deploy em Homologação'){
       steps{
           {
               docker.withServer("${DOCKER_HOMOLOG}"){
                   docker.withRegistry("${HTTP_PROTO}${NEXUS_SERVER}","${JENKINS_CREDEN}"){
                       image = docker.image("${NEXUS_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}")
                           image.pull()
                                sh """
                                        sed 's|IMAGE|${NEXUS_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}|g' docker-compose.yml > docker-compose-homolog.yml
                                   """
                                    sh "docker stack deploy -c docker-compose-homolog.yml course-catalog"                           
                    }
                }
            }
        }
    }
}

    post{
      success {
        junit 'nosetests.xml'
       }
   }    
}

```