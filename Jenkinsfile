pipeline {
    agent { label 'K8s' }

    environment {
        REGION = 'us-west-2'
        ECR_REPO = '036965198866.dkr.ecr.us-west-2.amazonaws.com/sasken/kenfest'
    }

    stages {
        stage('Cloning from repo') {
            steps {
                echo "Cloning repository..."
                git url: 'https://github.com/saskendevops/Githubactions-CICD.git', branch: 'main'
            }
        }
        stage('Build') {
            steps {
                echo "Building the code..."
                sh 'mvn clean package'
            }
        }
    //     stage('SonarQube Scan') {
    //     steps {
    //      script {
    //       try {
    //         echo "Running SonarQube analysis..."
    //         def sonarEnabled = true // Example condition
    //         if (sonarEnabled) {
    //           sh """
    //           mvn clean verify sonar:sonar \
    //            -Dsonar.projectKey=jenkins \
    //            -Dsonar.host.url=http://43.205.103.118:9000 \
    //            -Dsonar.login=sqp_0f8142335919bc4b518d159167e4dbb3fb96ebba 
				// """

    //           echo "SonarQube analysis completed successfully."
    //         } else {
    //           echo "Skipping SonarQube analysis as it is disabled."
    //         }
    //       } catch (Exception e) {
    //         echo "SonarQube analysis failed: ${e.message}"
    //         currentBuild.result = 'FAILURE'
    //         error("Exiting pipeline due to failure in SonarQube Analysis stage.")
    //       }
    //     }
    //   }
    //  }
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
                    aws configure set region us-west-2
                    aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin 036965198866.dkr.ecr.us-west-2.amazonaws.com
                    docker tag pranay356/demo:latest 036965198866.dkr.ecr.us-west-2.amazonaws.com/sasken/kenfest:latest
                    docker push 036965198866.dkr.ecr.us-west-2.amazonaws.com/sasken/kenfest:latest
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
                    cd ../..  # Change to the desired directory
                    echo "Changed to directory:"
                    pwd
                    kubectl apply -f deploy.yaml
                    kubectl rollout restart deployment boardgame-deployment
                '''
    }
}
      /*  
        stage('Clean Workspace') {
            steps {
                echo "Cleaning workspace"
                cleanWs()
            }
        }
        */
    }

   
}
