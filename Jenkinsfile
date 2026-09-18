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
                    junit(
                        testResults: 'reports/junit/junit.xml',
                        allowEmptyResults: true
                    )
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

                    ARTIFACT="node-demo-${BUILD_NUMBER}.tar.gz"

                    echo "Creating artifact: ${ARTIFACT}"

                    tar -czf "${ARTIFACT}" \
                        dist \
                        package.json \
                        package-lock.json
                '''

                archiveArtifacts(
                    artifacts: "node-demo-${BUILD_NUMBER}.tar.gz",
                    fingerprint: true
                )
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

                        echo "Resolving EC2 hostname..."

                        getent hosts "$EC2_HOST"

                        echo "Testing SSH connection to EC2..."

                        ssh -4 \
                            -o BatchMode=yes \
                            -o ConnectTimeout=10 \
                            -o ConnectionAttempts=3 \
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

                        scp -4 \
                            -o BatchMode=yes \
                            -o ConnectTimeout=10 \
                            -o StrictHostKeyChecking=accept-new \
                            "$ARTIFACT" \
                            "$EC2_USER@$EC2_HOST:/tmp/$ARTIFACT"


                        echo "Deploying application to EC2..."

                        ssh -4 \
                            -o BatchMode=yes \
                            -o ConnectTimeout=10 \
                            -o ConnectionAttempts=3 \
                            -o StrictHostKeyChecking=accept-new \
                            "$EC2_USER@$EC2_HOST" \
                            "BUILD_NUMBER=$BUILD_NUMBER DEPLOY_ROOT=$DEPLOY_ROOT bash -s" <<'REMOTE_SCRIPT'

set -e


# ==========================================
# LOAD NVM AND NODE.JS 26
# ==========================================

export NVM_DIR="$HOME/.nvm"

if [ -s "$NVM_DIR/nvm.sh" ]; then
    . "$NVM_DIR/nvm.sh"
else
    echo "ERROR: NVM is not installed for user: $(whoami)"
    exit 1
fi

nvm use 26


echo "Node version:"
node --version

echo "NPM version:"
npm --version

command -v node
command -v npm


# ==========================================
# DEPLOYMENT VARIABLES
# ==========================================

ARTIFACT="node-demo-${BUILD_NUMBER}.tar.gz"

RELEASE_DIR="$DEPLOY_ROOT/releases/$BUILD_NUMBER"


echo "=========================================="
echo "Deployment started"
echo "Build number: $BUILD_NUMBER"
echo "Release directory: $RELEASE_DIR"
echo "=========================================="


# ==========================================
# CREATE RELEASE DIRECTORY
# ==========================================

echo "Creating release directory..."

mkdir -p "$RELEASE_DIR"


# ==========================================
# EXTRACT ARTIFACT
# ==========================================

echo "Extracting artifact..."

tar -xzf "/tmp/$ARTIFACT" \
    -C "$RELEASE_DIR"


cd "$RELEASE_DIR"


# ==========================================
# INSTALL PRODUCTION DEPENDENCIES
# ==========================================

echo "Installing production dependencies..."

npm ci --omit=dev


# ==========================================
# STOP PREVIOUS APPLICATION
# ==========================================

echo "Stopping previous application..."

if [ -f "$DEPLOY_ROOT/app.pid" ]; then

    OLD_PID=$(cat "$DEPLOY_ROOT/app.pid" || true)

    if [ -n "$OLD_PID" ] && \
       kill -0 "$OLD_PID" 2>/dev/null; then

        echo "Stopping process: $OLD_PID"

        kill "$OLD_PID" || true

        sleep 2

    else

        echo "No active process found for PID: $OLD_PID"

    fi

fi


# ==========================================
# UPDATE CURRENT RELEASE SYMLINK
# ==========================================

echo "Updating current release symlink..."

ln -sfn "$RELEASE_DIR" \
    "$DEPLOY_ROOT/current"


# ==========================================
# START APPLICATION
# ==========================================

echo "Starting new application..."

cd "$DEPLOY_ROOT/current"


APP_VERSION="$BUILD_NUMBER" \
PORT=3000 \
nohup node dist/server.js \
    > "$DEPLOY_ROOT/app.log" 2>&1 &

NEW_PID=$!

echo "$NEW_PID" > "$DEPLOY_ROOT/app.pid"

echo "Application PID: $NEW_PID"


# ==========================================
# VERIFY APPLICATION HEALTH
# ==========================================

echo "Checking application health..."

sleep 2

if curl --fail \
    --silent \
    --show-error \
    --max-time 10 \
    http://127.0.0.1:3000/health \
    > /tmp/application-health.json; then

    echo "Application started successfully"

    echo "Health response:"

    cat /tmp/application-health.json

    echo

else

    echo "ERROR: Application health check failed"

    echo "Application logs:"

    cat "$DEPLOY_ROOT/app.log" || true

    echo "Running Node.js processes:"

    ps aux | grep '[n]ode' || true

    echo "Port 3000 status:"

    ss -ltnp | grep ':3000' || true

    exit 1

fi


# ==========================================
# CLEAN TEMPORARY FILES
# ==========================================

echo "Cleaning temporary files..."

rm -f "/tmp/$ARTIFACT"

rm -f /tmp/application-health.json


# ==========================================
# REMOVE OLD RELEASES
# KEEP LATEST 5
# ==========================================

echo "Removing old releases..."

find "$DEPLOY_ROOT/releases" \
    -mindepth 1 \
    -maxdepth 1 \
    -type d \
    -printf '%T@ %p\n' 2>/dev/null \
    | sort -nr \
    | tail -n +6 \
    | cut -d' ' -f2- \
    | xargs -r rm -rf


# ==========================================
# DEPLOYMENT COMPLETE
# ==========================================

echo "=========================================="
echo "Deployment completed successfully!"
echo "=========================================="

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

                        ssh -4 \
                            -o BatchMode=yes \
                            -o ConnectTimeout=10 \
                            -o ConnectionAttempts=3 \
                            -o StrictHostKeyChecking=accept-new \
                            "$EC2_USER@$EC2_HOST" \
                            "sleep 2 && curl --fail --silent --show-error http://127.0.0.1:3000/health"

                        echo

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

        always {
            echo 'Pipeline execution completed.'
        }
    }

}