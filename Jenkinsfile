pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "hapi-fhir-server:latest"
        DOCKERFILE_PATH = "./Dockerfile"
        GITHUB_REPO = "https://github.com/Ayataaki/Devops-hapi-fhir-jpaserver.git"
        GITHUB_CREDENTIALS = "github-token"
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checkout du code...'
                git branch: 'main',
                    url: "${GITHUB_REPO}",
                    credentialsId: "${GITHUB_CREDENTIALS}"
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Construction de l’image Docker...'
                sh """
                    docker build -t ${DOCKER_IMAGE} -f ${DOCKERFILE_PATH} .
                """
            }
        }

        stage('Run Container for Tests') {
            steps {
                echo 'Lancement du conteneur pour tests...'
                sh """
                    docker run --rm -d --name hapi-fhir-test -p 9099:8080 ${DOCKER_IMAGE}
                """
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Analyse SonarQube...'
                withSonarQubeEnv('SonarQube') {
                    sh """
                        # Run analysis inside the container or via Maven if needed
                        docker run --rm ${DOCKER_IMAGE} \
                          mvn sonar:sonar -Dsonar.host.url=${SONAR_HOST_URL} -Dsonar.login=${SONAR_AUTH_TOKEN}
                    """
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline terminé avec succès'
        }
        failure {
            echo '❌ Pipeline échoué'
        }
    }
}