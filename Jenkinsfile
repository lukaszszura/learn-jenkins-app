pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '270a1086-89f6-48cf-a574-95b2ff8ccb8b'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }

    stages {

        stage('Install Dependencies') {
            agent {
                docker {
                    image 'node:18-alpine'
                }
            }
            steps {
                sh '''
                    npm install
                    npm install --save-dev @babel/plugin-proposal-private-property-in-object
                '''
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Tests') {
            agent {
                docker {
                    image 'node:18-alpine'
                }
            }
            steps {
                sh '''
                    npm test -- --detectOpenHandles
                '''
            }
            post {
                always {
                    junit 'jest-results/junit.xml'
                }
            }
        }

        stage('Deploy staging') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'STAGING_URL_TO_BE_SET'
            }

            steps {
                sh '''
                    # Install required dependencies
                    npm install netlify-cli node-jq

                    # Verify Netlify CLI version
                    node_modules/.bin/netlify --version

                    # Deploy to staging
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json

                    # Extract the staging URL
                    export CI_ENVIRONMENT_URL=$(node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json)
                    if [ -z "$CI_ENVIRONMENT_URL" ]; then
                        echo "Error: Failed to extract staging URL from deploy-output.json"
                        exit 1
                    fi
                    echo "Staging URL: $CI_ENVIRONMENT_URL"

                    # Run Playwright tests
                    npx playwright test --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        stage('Approval') {
            steps {
                script {
                    input message: 'Do you approve the deployment to production?', ok: 'Proceed to Production'
                }
            }
        }

        stage('Deploy prod') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                }
            }

            environment {
                CI_ENVIRONMENT_URL = '270a1086-89f6-48cf-a574-95b2ff8ccb8b'
            }

            steps {
                sh '''
                    node --version
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                    npx playwright test --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}