pipeline {
    agent { label 'k8s' }

    environment {
        REGION = 'us-west-2'
        ECR_REPO = '036965198866.dkr.ecr.us-west-2.amazonaws.com/sasken/kenfest'
        SONAR_HOST_URL = 'http://43.205.103.118:9000'
        SONAR_PROJECT_KEY = 'jenkins'
    }

    stages {
        stage('Cloning from repo') {
            steps {
                echo "Cloning repository..."
                git url: 'https://github.com/saskendevops/Githubactions-CICD.git', branch: 'main'
                script {
                    def branch = sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                    def commit = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    echo "Branch: ${branch}"
                    echo "Commit: ${commit}"
                }
            }
        }

        stage('Build') {
            steps {
                echo "Building the code..."
                sh 'mvn clean package'
            }
        }

        stage('SonarQube Scan') {
            steps {
                script {
                    try {
                        echo "Running SonarQube analysis..."
                        withCredentials([string(credentialsId: 'sonar-token', variable: 'SonarQube')]) {
                            sh """
                            mvn clean verify sonar:sonar \
                              -Dsonar.projectKey=Sasken-KenFest-Demo \
                              -Dsonar.host.url=http://43.204.43.201:9000 \
                              -Dsonar.login=sqp_19ea17860bbf15a7c5709fe1fba476ab246f6bde
                            """
                        }
                        echo "SonarQube analysis completed successfully."
                    } catch (Exception e) {
                        echo "SonarQube analysis failed: ${e.message}"
                        currentBuild.result = 'FAILURE'
                        error("Exiting pipeline due to failure in SonarQube Analysis stage.")
                    }
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                echo "Running Trivy scan..."
                sh """
                trivy fs --security-checks vuln,config . || exit 1
                """
            }
        }

        stage('Docker Build') {
            steps {
                echo "Building Docker image..."
                sh "docker build -t pranay356/demo:latest ."
            }
        }

        stage('SBOM Generation') {
            steps {
                echo "Generating SBOM file using Trivy..."
                sh "trivy image --format cyclonedx --output sbom.cyclonedx.json pranay356/demo:latest"
                archiveArtifacts artifacts: 'sbom.cyclonedx.json', fingerprint: true
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Docker', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    echo "Pushing Docker image to repository..."
                    sh """
                    echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                    docker push pranay356/demo:latest
                    """
                }
            }
        }

        stage('Image push to ECR') {
            steps {
                echo "Pushing image to ECR..."
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS-ECR']]) {
                    sh """
                    aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                    aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                    aws configure set region $REGION
                    aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ECR_REPO
                    docker tag pranay356/demo:latest $ECR_REPO:latest
                    docker push $ECR_REPO:latest
                    """
                }
            }
        }

        stage('Deployment') {
            steps {
                echo "Deployment in K8s Cluster is in progress..."
                sh '''
                    echo "Current directory:"
                    pwd
                    kubectl apply -f deployment-service.yaml
                    kubectl rollout restart deployment boardgame-deployment
                '''
            }
        }

        
    }
}
