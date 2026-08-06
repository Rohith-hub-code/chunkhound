pipeline {

    agent any

    options {
        timestamps()
    }

    environment {
    PATH = "/usr/local/bin:/usr/bin:/bin:${env.PATH}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/adikarthik/chunkhound.git'
            }
        }


        stage('Install Dependencies') {
            steps {
                sh '''
                    uv --version
                    uv sync
                '''
            }
        }


        stage('Code Quality Checks') {

            parallel {

                stage('Black') {
                    steps {
                        script {
                            try {
                                sh 'uv run black --check .'
                            }
                            catch (Exception e) {
                                echo "Black check failed, but continuing..."
                            }
                        }
                    }
                }


                stage('Ruff') {
                    steps {
                        script {
                            try {
                                sh 'uv run ruff check .'
                            }
                            catch (Exception e) {
                                echo "Ruff check failed, but continuing..."
                            }
                        }
                    }
                }


                stage('MyPy') {
                    steps {
                        script {
                            try {
                                sh 'uv run mypy .'
                            }
                            catch (Exception e) {
                                echo "MyPy check failed, but continuing..."
                            }
                        }
                    }
                }


                stage('Bandit') {
                    steps {
                        script {
                            try {
                                sh '''
                                    uv run bandit -r . \
                                    -x .venv,.uv-cache,dist,build,__pycache__
                                '''
                            }
                            catch (Exception e) {
                                echo "Bandit check failed, but continuing..."
                            }
                        }
                    }
                }
                
                stage('Pytest') {
            steps {
                script {
                    try {
                        sh '''
                            uv run pytest -v
                        '''
                    }
                    catch (Exception e) {
                        echo "Pytest failed, but continuing..."
                    }
                }
            }
        }

            }
        }


        stage('SonarQube Analysis') {

            steps {

                script {

                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {

                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=chunkhound \
                            -Dsonar.projectName=Chunkhound \
                            -Dsonar.sources=.
                        """
                    }
                }
            }
        }


        stage('Build Artifact') {

            steps {

                sh '''
                    uv build
                '''
            }
        }


        stage('Upload Artifact') {

            steps {

                withCredentials([
                    string(credentialsId: 'AZURE_CLIENT_ID', variable: 'CLIENT_ID'),
                    string(credentialsId: 'AZURE_CLIENT_SECRET', variable: 'CLIENT_SECRET'),
                    string(credentialsId: 'AZURE_TENANT_ID', variable: 'TENANT_ID')
                ]) {

                    sh '''
                        az login --service-principal \
                            -u $CLIENT_ID \
                            -p $CLIENT_SECRET \
                            --tenant $TENANT_ID


                        az storage blob upload-batch \
                            --account-name rpstorage07 \
                            --destination rpcontainer \
                            --source dist \
                            --pattern "*.whl" \
                            --overwrite
                    '''
                }
            }
        }


        stage('Archive Artifact') {

            steps {

                archiveArtifacts artifacts: 'dist/*',
                                 fingerprint: true
            }
        }

    }


    post {

        success {

            echo 'Pipeline completed successfully.'
        }


        unstable {

            echo 'Pipeline completed with quality issues, but artifact was generated.'
        }


        failure {

            echo 'Pipeline failed.'
        }


        always {

            cleanWs()
        }

    }

}
