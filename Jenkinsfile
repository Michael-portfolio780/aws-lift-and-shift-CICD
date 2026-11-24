
pipeline {
    agent any

    stages {

        stage('Verify Repository') {
            steps {
                sh '''
                    set -eu

                    test -f drupal11/composer.json
                    test -f drupal11/composer.lock
                    test -f sonar-project.properties

                    echo "Drupal repository verified."
                '''
            }
        }

        stage('Gitleaks Secret Scan') {
            steps {
                sh '''
                    set -eu

                    echo "Starting Gitleaks scan..."

                    gitleaks git . \
                        --redact \
                        --no-banner \
                        --report-format json \
                        --report-path gitleaks-report.json

                    echo "Gitleaks scan completed."
                '''
            }

            post {
                always {
                    archiveArtifacts(
                        artifacts: 'gitleaks-report.json',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('Semgrep SAST Scan') {
            steps {
                dir('drupal11') {
                    sh '''
                        set -eu

                        echo "Starting first-party Semgrep scan..."

                        mkdir -p ../reports

                        if find web/modules web/themes \
                            -type f \
                            \\( -name "*.php" \
                            -o -name "*.module" \
                            -o -name "*.install" \
                            -o -name "*.inc" \
                            -o -name "*.profile" \
                            -o -name "*.theme" \\) \
                            -print -quit | grep -q .; then

                            semgrep scan \
                                --config auto \
                                --json \
                                --output ../reports/semgrep-report.json \
                                web/modules web/themes

                            echo "Semgrep scan completed."

                        else
                            echo "No custom Drupal PHP modules or themes found."

                            echo '{"results":[],"errors":[]}' \
                                > ../reports/semgrep-report.json
                        fi
                    '''
                }
            }

            post {
                always {
                    archiveArtifacts(
                        artifacts: 'reports/semgrep-report.json',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('SonarQube Code Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        sh """
                            set -eu

                            echo "Starting SonarQube analysis..."

                            ${scannerHome}/bin/sonar-scanner \
                                -Dproject.settings=sonar-project.properties

                            echo "SonarQube analysis submitted."
                        """
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Composer Validation') {
            steps {
                dir('drupal11') {
                    sh '''
                        set -eu

                        echo "Validating Composer configuration..."

                        composer validate --no-check-publish

                        echo "Checking platform requirements..."

                        composer check-platform-reqs --lock
                    '''
                }
            }
        }

        stage('Composer Security Audit') {
            steps {
                dir('drupal11') {
                    sh '''
                        set -eu

                        echo "Auditing Composer dependencies..."

                        composer audit --locked --no-interaction
                    '''
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('drupal11') {
                    sh '''
                        set -eu

                        echo "Installing Drupal dependencies..."

                        composer install \
                            --no-interaction \
                            --prefer-dist \
                            --optimize-autoloader

                        test -f vendor/autoload.php
                        test -d web/core

                        echo "Drupal dependencies installed successfully."
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Drupal CI pipeline completed successfully.'
        }

        failure {
            echo 'Drupal CI pipeline failed. Check the stage logs and archived reports.'
        }
    }
}
