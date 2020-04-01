pipeline {

    agent {
        label "jenkins-slave-npm"
    }

    environment {
        // Global Vars
        NAMESPACE_PREFIX="summit-tekton"

        PIPELINES_NAMESPACE = "tekton-summit"
        APP_NAME = "todolist"

        JENKINS_TAG = "${JOB_NAME}.${BUILD_NUMBER}".replace("/", "-")
        JOB_NAME = "${JOB_NAME}".replace("/", "-")

        GIT_SSL_NO_VERIFY = true
    }

    // The options directive is for configuration that applies to the whole job.
    options {
        buildDiscarder(logRotator(numToKeepStr:'10'))
        timeout(time: 15, unit: 'MINUTES')
    }

    stages {
        stage("node-build") {
            steps {
                sh 'printenv'

                echo '### Install deps ###'
                sh 'npm install'

                echo '### Running tests ###'
                sh 'npm run test:all:ci'

                echo '### Running build ###'
                sh 'npm run build:ci'

//                echo '### Packaging App for Nexus ###'
//                sh 'npm run package'
//                sh 'npm run publish'
//                stash 'source'
            }
        }

        stage("node-bake") {
            steps {
                echo '### Create Linux Container Image from package ###'
                sh  '''
                        oc project ${PIPELINES_NAMESPACE} # probs not needed
                        oc patch bc ${APP_NAME} -p "{\\"spec\\":{\\"output\\":{\\"to\\":{\\"kind\\":\\"ImageStreamTag\\",\\"name\\":\\"${APP_NAME}:${JENKINS_TAG}\\"}}}}"
                        oc start-build ${APP_NAME} --from-dir=package-contents/ --follow
                    '''
            }
        }

        stage("node-deploy") {
            steps {
                echo '### tag image for namespace ###'
                sh  '''
                    oc project ${PROJECT_NAMESPACE}
                    oc tag ${PIPELINES_NAMESPACE}/${APP_NAME}:${JENKINS_TAG} ${PROJECT_NAMESPACE}/${APP_NAME}:${JENKINS_TAG}
                    '''
                echo '### set env vars and image for deployment ###'
                sh '''
                    oc set env dc ${APP_NAME} NODE_ENV=${NODE_ENV}
                    oc set image dc/${APP_NAME} ${APP_NAME}=docker-registry.default.svc:5000/${PROJECT_NAMESPACE}/${APP_NAME}:${JENKINS_TAG}
                    oc rollout latest dc/${APP_NAME}
                '''
                echo '### Verify OCP Deployment ###'
                openshiftVerifyDeployment depCfg: env.APP_NAME, 
                    namespace: env.PROJECT_NAMESPACE, 
                    replicaCount: '1', 
                    verbose: 'false', 
                    verifyReplicaCount: 'true', 
                    waitTime: '',
                    waitUnit: 'sec'
            }
        }
    }
}
