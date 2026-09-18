pipeline {

    agent any

    tools {
        nodejs 'NodeJS-26'
    }

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        EC2_USER = 'deploy'
        EC2_HOST = 'ec2-44-222-125-115.compute-1.amazonaws.com'
        DEPLOY_ROOT = '/home/deploy/cicd-demo/node'
    }

    stages {

        // ==========================================
        // 1. CHECKOUT
        // ==========================================

        stage('Checkout') {

            steps {
                checkout scm
            }

        }


        // ==========================================
        // 2. INSTALL DEPENDENCIES
        // ==========================================

        stage('Install Dependencies') {

            steps {
                sh 'npm ci'
            }

        }


        // ==========================================
        // 3. LINT
        // ==========================================

        stage('Lint') {

            steps {
                sh 'npm run lint'
            }

        }


        // ==========================================
        // 4. TESTS
        // ==========================================

        stage('Tests') {

            steps {
                sh 'npm run test:ci'
            }

            post {

                always {

                    junit testResults: 'reports/junit/junit.xml',
                          allowEmptyResults: true

                }

            }

        }


        // ==========================================
        // 5. BUILD
        // ==========================================

        stage('Build') {

            steps {
                sh 'npm run build'
            }

        }


        // ==========================================
        // 6. PACKAGE
        // ==========================================

        stage('Package') {

            steps {

                sh '''
                    set -e

                    tar -czf node-demo-${BUILD_NUMBER}.tar.gz \
                        dist package.json package-lock.json

                '''

                archiveArtifacts artifacts: "node-demo-${BUILD_NUMBER}.tar.gz",
                                 fingerprint: true

            }

        }


        // ==========================================
        // 7. TEST EC2 CONNECTION
        // ==========================================

        stage('Test EC2 Connection') {

            when {
                branch 'main'
            }

            steps {

                sshagent(credentials: ['ec2-deployment-key']) {

                    sh '''

                        set -e

                        echo "Testing SSH connection to EC2..."

                        ssh \
                            -o StrictHostKeyChecking=accept-new \
                            "$EC2_USER@$EC2_HOST" \
                            "echo Connected to EC2 && whoami && hostname"

                    '''

                }

            }

        }


        // ==========================================
        // 8. DEPLOY TO EC2
        // ==========================================

        stage('Deploy') {

            when {
                branch 'main'
            }

            steps {

                sshagent(credentials: ['ec2-deployment-key']) {

                    sh '''

                        set -e

                        ARTIFACT="node-demo-${BUILD_NUMBER}.tar.gz"

                        echo "Uploading artifact to EC2..."

                        scp \
                            -o StrictHostKeyChecking=accept-new \
                            "$ARTIFACT" \
                            "$EC2_USER@$EC2_HOST:/tmp/$ARTIFACT"


                        echo "Deploying application to EC2..."


                        ssh \
                            -o StrictHostKeyChecking=accept-new \
                            "$EC2_USER@$EC2_HOST" \
                            "BUILD_NUMBER=$BUILD_NUMBER DEPLOY_ROOT=$DEPLOY_ROOT bash -s" <<'REMOTE_SCRIPT'


                            set -e


                            ARTIFACT="node-demo-${BUILD_NUMBER}.tar.gz"

                            RELEASE_DIR="$DEPLOY_ROOT/releases/$BUILD_NUMBER"


                            echo "Creating release directory..."

                            mkdir -p "$RELEASE_DIR"


                            echo "Extracting artifact..."

                            tar -xzf "/tmp/$ARTIFACT" \
                                -C "$RELEASE_DIR"


                            cd "$RELEASE_DIR"


                            echo "Installing production dependencies..."

                            npm ci --omit=dev


                            echo "Stopping previous application..."


                            if [ -f "$DEPLOY_ROOT/app.pid" ]; then

                                OLD_PID=$(cat "$DEPLOY_ROOT/app.pid" || true)


                                if [ -n "$OLD_PID" ] && \
                                   kill -0 "$OLD_PID" 2>/dev/null; then


                                    echo "Stopping process: $OLD_PID"

                                    kill "$OLD_PID" || true


                                    sleep 2


                                fi

                            fi


                            echo "Updating current release symlink..."


                            ln -sfn "$RELEASE_DIR" \
                                "$DEPLOY_ROOT/current"


                            echo "Starting new application..."


                            cd "$DEPLOY_ROOT/current"


                            APP_VERSION="$BUILD_NUMBER" \
                            PORT=3000 \
                            nohup node dist/server.js \
                            > "$DEPLOY_ROOT/app.log" 2>&1 &


                            echo $! > "$DEPLOY_ROOT/app.pid"


                            echo "Application PID:"

                            cat "$DEPLOY_ROOT/app.pid"


                            echo "Cleaning temporary artifact..."

                            rm -f "/tmp/$ARTIFACT"


                            echo "Deployment completed successfully!"


REMOTE_SCRIPT

                    '''

                }

            }

        }


        // ==========================================
        // 9. SMOKE TEST
        // ==========================================

        stage('Smoke Test') {

            when {
                branch 'main'
            }

            steps {

                sshagent(credentials: ['ec2-deployment-key']) {

                    sh '''

                        set -e

                        echo "Running remote smoke test..."


                        ssh \
                            -o StrictHostKeyChecking=accept-new \
                            "$EC2_USER@$EC2_HOST" \
                            "sleep 2 && curl -f http://127.0.0.1:3000/health"


                        echo "Smoke test passed!"

                    '''

                }

            }

        }

    }


    // ==========================================
    // POST ACTIONS
    // ==========================================

    post {

        success {

            echo 'PIPELINE SUCCESSFUL'

        }

        failure {

            echo 'PIPELINE FAILED - deployment was blocked'

        }

    }

}