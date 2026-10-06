pipeline {
    agent any

    options {
        // Six Deployments are updated together; two overlapping runs would fight over them.
        disableConcurrentBuilds()
        // Heavy build (frappe_docker layered image, node + bench build for 3 apps)
        timeout(time: 90, unit: 'MINUTES')
    }

    environment {
        AWS_REGION        = 'ap-south-1'
        AWS_ACCOUNT_ID    = '379220350808'
        IMAGE_REGISTRY    = '379220350808.dkr.ecr.ap-south-1.amazonaws.com'   // matches image_registry in inventory/group_vars/all/main.yml
        IMAGE_NAME        = 'oan-frappe'   // matches frappe_image_repo; must exist as an ECR repository first
        RKE2_NODE         = '13.233.56.204'
        K8S_NAMESPACE     = 'develop'
        // All six Deployments run the SAME oan-frappe image with different commands:
        //   backend      -> gunicorn frappe.app:application
        //   frontend     -> nginx-entrypoint.sh
        //   scheduler    -> bench schedule
        //   websocket    -> node apps/frappe/socketio.js
        //   worker-long  -> bench worker --queue long
        //   worker-short -> bench worker --queue short,default
        DEPLOYMENTS       = 'backend frontend scheduler websocket worker-long worker-short'
        // frappe_docker build definition. Pin to a tag or commit here for reproducible builds.
        FRAPPE_DOCKER_REF = 'main'
        VERIFY_URL        = 'https://grievance-backend-dev.oanstaging.com/api/method/ping'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    // Immutable, traceable tag: develop-<build number>-<short sha of THIS repo>.
                    // (The image also carries oan_auth_service HEAD, which this sha does not cover.)
                    def sha = sh(returnStdout: true, script: 'git rev-parse --short=7 HEAD').trim()
                    env.IMAGE_TAG_BUILD = "develop-${env.BUILD_NUMBER}-${sha}"
                    env.FULL_IMAGE      = "${env.IMAGE_REGISTRY}/${env.IMAGE_NAME}:${env.IMAGE_TAG_BUILD}"
                }
            }
        }

        stage('Prepare apps.json') {
            when { branch 'develop' }
            steps {
                script {
                    // Only the EXTRA apps go here: frappe itself is installed from FRAPPE_PATH /
                    // FRAPPE_BRANCH in the build step. Both OAN apps are pinned to the branch being
                    // built, so the image carries current develop HEAD of BOTH apps.
                    // Written as plain text: groovy.json.JsonOutput is blocked by the Jenkins
                    // script sandbox ("Scripts not permitted to use staticMethod ...").
                    def branch = env.BRANCH_NAME
                    def appsJson = """[
  {"url": "https://github.com/Centre-for-Open-Societal-Systems/oan_auth_service", "branch": "${branch}"},
  {"url": "https://github.com/Centre-for-Open-Societal-Systems/oan_grievance_service", "branch": "${branch}"}
]
"""
                    writeFile file: 'apps.json', text: appsJson
                }
            }
        }

        stage('Build image') {
            when { branch 'develop' }
            steps {
                // frappe_docker's own layered Containerfile is the build definition; this repo
                // only supplies the apps.json branch pins above.
                // Current frappe_docker reads apps.json from a BuildKit secret with id "apps_json"
                // (the old APPS_JSON_BASE64 build arg is silently ignored and gives plain frappe).
                // BuildKit does not include secret contents in its cache key, so CACHE_BUST (the
                // Containerfile's own cache-busting arg) forces `bench init` to re-fetch the apps
                // on every build; without it a cached no-apps layer would be reused.
                sh '''
                    [ -s "$WORKSPACE/apps.json" ] || { echo "apps.json is missing or empty"; exit 1; }
                    cat "$WORKSPACE/apps.json"

                    rm -rf frappe_docker
                    git clone --depth 1 --branch "$FRAPPE_DOCKER_REF" https://github.com/frappe/frappe_docker.git
                    cd frappe_docker
                    DOCKER_BUILDKIT=1 docker build \
                        --secret id=apps_json,src="$WORKSPACE/apps.json" \
                        --build-arg CACHE_BUST="$BUILD_NUMBER" \
                        --build-arg FRAPPE_PATH=https://github.com/frappe/frappe \
                        --build-arg FRAPPE_BRANCH=version-16 \
                        --file images/layered/Containerfile \
                        -t "$FULL_IMAGE" \
                        -t "$IMAGE_REGISTRY/$IMAGE_NAME:develop" \
                        .

                    # Never push or deploy an image that is missing the OAN apps
                    docker run --rm "$FULL_IMAGE" bash -c "ls apps && test -d apps/oan_auth_service && test -d apps/oan_grievance_service"
                '''
            }
        }

        stage('Push image') {
            when { branch 'develop' }
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-ecr-creds'
                ]]) {
                    sh '''
                        aws ecr get-login-password --region "$AWS_REGION" \
                            | docker login --username AWS --password-stdin "$IMAGE_REGISTRY"
                        docker push "$FULL_IMAGE"
                        docker push "$IMAGE_REGISTRY/$IMAGE_NAME:develop"
                    '''
                }
            }
        }

        stage('Deploy to develop (kubectl)') {
            when { branch 'develop' }
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'grievance-dev-ssh-key',   // grievance.pem
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )]) {
                    // Sets all six Deployments to the immutable build tag (no dependence on
                    // imagePullPolicy or a floating tag). If anything fails, only the Deployments
                    // already updated by this run are rolled back.
                    sh '''
ssh -o StrictHostKeyChecking=no -i "$SSH_KEY" "$SSH_USER@$RKE2_NODE" \
    bash -s -- "$(echo "$DEPLOYMENTS" | tr ' ' ',')" "$FULL_IMAGE" "$K8S_NAMESPACE" <<'ENDSSH'
set -euo pipefail
# ssh joins its arguments into one string and the remote shell re-splits it, so the list
# of Deployments travels comma-separated and is split here.
DEPLOYMENTS="$(echo "$1" | tr ',' ' ')"; IMAGE="$2"; NS="$3"
# Non-interactive SSH sessions do not load the login PATH; RKE2 keeps kubectl in its own bin dir
export PATH="$PATH:/var/lib/rancher/rke2/bin:/usr/local/bin:/snap/bin"
export KUBECONFIG="$HOME/.kube/config"

# Fail fast if any Deployment name does not exist
for d in $DEPLOYMENTS; do
    kubectl get deployment "$d" -n "$NS" > /dev/null
done

UPDATED=""
rollback() {
    echo "Rolling back: $UPDATED"
    for d in $UPDATED; do
        kubectl rollout undo "deployment/$d" -n "$NS" || true
    done
}

echo "Deploying $IMAGE to: $DEPLOYMENTS ($NS)"
for d in $DEPLOYMENTS; do
    kubectl set image "deployment/$d" "*=$IMAGE" -n "$NS" || { rollback; exit 1; }
    UPDATED="$UPDATED $d"
done

for d in $DEPLOYMENTS; do
    echo "waiting on $d..."
    kubectl rollout status "deployment/$d" -n "$NS" --timeout=300s || { rollback; exit 1; }
done
ENDSSH
                    '''
                }
            }
        }

        stage('Verify') {
            when { branch 'develop' }
            steps {
                sh '''
                    for i in $(seq 1 20); do
                        code=$(curl -sk -o /dev/null -w "%{http_code}" "$VERIFY_URL" || true)
                        echo "attempt $i: $VERIFY_URL -> $code"
                        [ "$code" = "200" ] && exit 0
                        sleep 10
                    done
                    echo "frappe backend did not return 200 at $VERIFY_URL"
                    exit 1
                '''
            }
        }
    }

    post {
        failure {
            echo 'oan-frappe develop deploy failed. Check the stage logs above, and `kubectl get deployment -n develop -o wide` for any Deployment still on a different image.'
        }
    }
}
