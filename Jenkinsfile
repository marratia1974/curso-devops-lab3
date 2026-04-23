// Una funcion en un jenkins file Se escribe como cualquier funcion. Tiene un nombre, 
// y parametros y comienza con la palabra reservada def. Esta funcion se llama tagAndPush
// y sirve para resumir la logica de upload de imagenes sin repetir el que teniamos antes.

def tagAndPush(String localImage, String repo, String registry, String credential) {

    docker.withRegistry(registry, credential) {
        sh "docker tag ${localImage} ${repo}:latest"
        sh "docker tag ${localImage} ${repo}:${env.BUILD_NUMBER}"
        sh "docker tag ${localImage} ${repo}:${env.APP_SEMANTIC_VERSION}"
        sh "docker push ${repo}:latest"
        sh "docker push ${repo}:${env.BUILD_NUMBER}"
        sh "docker push ${repo}:${env.APP_SEMANTIC_VERSION}"
    }

}

pipeline {
    agent any
    // Aca podemos declarar variables que luego podemos acceder como variables de ambiente dentro del pipeline
    // usando "env.". Estas variables solo existen en este pipeline.
    environment {
        IMAGE_NAME = "curso-devops-lab3"
        DH_REPO    = "marratia1974/curso-devops-lab3"
        GHCR_REPO  = "ghcr.io/marratia1974/curso-devops-lab3"
        K8S_NAMESPACE  = "marratia"
        K8S_DEPLOYMENT = "curso-devops-deployment"
        K8S_CONTAINER  = "contenedor-curso-devops-lab3"



    }
    stages {
        stage("Integracion continua") {
            agent {
                docker {
                    image "node:24"
                    reuseNode true
                }
            }
            stages {
                stage("CI de la aplicacion - version") {
                    steps {
                        script {
                            env.APP_SEMANTIC_VERSION = sh(
                                script: 'npm pkg get version | tr -d \'"\'',
                                returnStdout: true
                            ).trim()
                            echo "Version semantica detectada: ${env.APP_SEMANTIC_VERSION}"
                        }
                    }
                }
                stage("CI de la aplicacion - dependencias") {
                    steps {
                        sh "npm install"
                    }
                }
                stage("CI de la aplicacion - lint") {
                    steps {
                        sh "npm run lint"
                    }
                }
                stage("CI de la aplicacion - test") {
                    steps {
                        sh "npm run test:cov"
                    }
                }
                stage("CI de la aplicacion - build") {
                    steps {
                        sh "npm run build"
                    }
                }
            }
        }
        stage("Quality Assurance"){
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli'
                    args '--network=devops-infra_default'
                    reuseNode true
                }
            }
            stages{
                stage("validacion de codigo"){
                    steps{
                        withSonarQubeEnv('sonarqube'){
                            sh 'sonar-scanner'
                        }
                    }
                }
                // stage('validacion quality gate'){
                //     steps{
                //         script{
                //             def  qualityGate = waitForQualityGate() // esperar por el resultado del qualitygate en un endpoint de jenkins, que se gatilla desde sonar via webhook.
                //             if(qualityGate.status != 'OK'){
                //                 error "La puerta de calidad ha fallado : ${qualityGate.status}"
                //             }
                //         }
                //     }
                // }
            }
        }
        stage("CD de la aplicacion - build dockerfile") {
            steps {
                sh "docker build -t ${env.IMAGE_NAME} ."
                script {
                    if (!env.APP_SEMANTIC_VERSION?.trim()) {
                        error("APP_SEMANTIC_VERSION no definida en el stage anterior")
                    }
                    // Aca llamamos a la funcion que definimos al principio , y ya esta funcion 
                    // hace login en dockerhub y github con docker.withRegistry y sube ambas imagenes
                    tagAndPush(env.IMAGE_NAME, env.DH_REPO, "https://index.docker.io/v1/", "credencial-dh")
                    tagAndPush(env.IMAGE_NAME, env.GHCR_REPO, "https://ghcr.io", "credencial-gh")
                }
            }
        }
 
        stage("CD - Despliegue continuo en develop") {
            agent {
                docker {
                    image 'alpine/k8s:1.34.6'
                    reuseNode true
                }
            }
            steps {
                withCredentials([file(credentialsId: 'credencial-11', variable: 'KUBECONFIG')]) {
                    sh """
                        kubectl -n ${env.K8S_NAMESPACE} set image deployment/${env.K8S_DEPLOYMENT} ${env.K8S_CONTAINER}=${env.GHCR_REPO}:${env.BUILD_NUMBER}
                        kubectl -n ${env.K8S_NAMESPACE} rollout status deployment/${env.K8S_DEPLOYMENT}
                    """
                }
            }
        }
    }
}

                        // kubectl -n ${env.K8S_NAMESPACE} set image deployment/${env.K8S_DEPLOYMENT} ${env.K8S_CONTAINER}=${env.DH_REPO}:${env.APP_SEMANTIC_VERSION}
                        // kubectl -n ${env.K8S_NAMESPACE} rollout status deployment/${env.K8S_DEPLOYMENT}
                        //image 'alpine/k8s:1.34.6'