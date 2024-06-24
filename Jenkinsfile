def imageName='xurse9/panda-frontend'
def dockerTag=''
def dockerRegistry=''
def registryCredentials='dockerhub'

pipeline {
    agent {
        label 'agent'
    }
 
    environment {
        PIP_BREAK_SYSTEM_PACKAGES = 1
        scannerHome = tool 'SonarQube'
    }
 
    stages {
        stage('Download code from Git') {
            steps {
                git branch: 'master', url: 'https://github.com/xurse9/Frontend'
            }
        }
 
        stage('Tests') {
            steps {
                sh 'pip3 install -r requirements.txt'
                sh 'python3 -m pytest --cov=. --cov-report xml:test-results/coverage.xml --junitxml=test-results/pytest-report.xml'
            }
        }


        stage('SonarQube') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '${scannerHome}/bin/sonar-scanner'
                }
            }
        } 
 

        stage('Build docker image') {
            steps {
                script {
                    dockerTag = "RC-${env.BUILD_ID}"
                    customImage = docker.build("$imageName:$dockerTag")
                }
            }
        }
        
        stage('Push docker image') {
            steps {
                script {
                    docker.withRegistry("$dockerRegistry", "$registryCredentials") 
                    {
                        customImage.push()
                        customImage.push('latest')
                    }
                }
            }
        }

        stage ('Push to Repo') {
            steps {
                dir('ArgoCD') {
                    withCredentials([gitUsernamePassword(credentialsId: 'git', gitToolName: 'Default')]) {
                        git branch: 'master', url: 'https://github.com/xurse9/argocd.git'
                        sh """ cd frontend
                        sed -i "s#$imageName.*#$imageName:$dockerTag#g" deployment.yaml
                        git commit -am "Set new $dockerTag tag."
                        git push origin master
                        """
                    }
                }
            }
        }
    }
        post {
            always {
                junit testResults: "test-results/*.xml"
                cleanWs()
            }
            success {
                build wait: false, job: 'app_of_apps', parameters: [string(name: 'frontendDockerTag', value: "$dockerTag")]
            }
        }
}
