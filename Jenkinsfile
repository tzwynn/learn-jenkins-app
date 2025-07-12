pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'e6449064-00bb-4021-85dc-5219e10bfb9e'
        NETLIFY_AUTH_TOKEN = credentials ('netlify-token')

    }

    stages {
       
        
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                
                }
            }
            steps {
                sh '''
                    echo 'Small change'
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }
        stage('Stage Tests') {
            parallel {
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        echo "Test Stage"
                        sh '''
                            #test -f build/index.html
                            npm test
                
                        '''
                        
                    }
                    post {
                            always {
                                junit 'test-results/junit.xml'
                                
                            }
                        }                  
                }
                stage('E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps {

                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                
                        '''
                        
                    }
                    post {
                        always {
                            
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }               

            }
        }

        stage('Deploy staging') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                   npm install netlify-cli
                   node_modules/.bin/netlify --version
                   echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                   node_modules/.bin/netlify status
                   node_modules/.bin/netlify deploy --dir=build --no-build
                '''
            }
        }

        stage('Approval') {
            timeout(30) {
                input cancel: 'Not approve', message: 'Would you like to Approve this deployment?', ok: 'Approve'
            }       
        }
        
        stage('Deploy') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                   npm install netlify-cli
                   node_modules/.bin/netlify --version
                   echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                   node_modules/.bin/netlify status
                   node_modules/.bin/netlify deploy --dir=build --prod --no-build
                '''
            }
        }

        stage('Prod E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {

                CI_ENVIRONMENT_URL = 'https://bejewelled-crisp-965ab1.netlify.app'
            }

            steps {

                sh '''
                    node_modules/.bin/serve -s build &
                    sleep 10
                    npx playwright test --reporter=html
        
                '''
                
            }
            post {
                always {
                    
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Post Deployment Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }    
    }
    
}
