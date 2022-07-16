#!groovy

retry (10) {
    // load pipeline configuration into the environment
    httpRequest("${FEDORA_CI_PIPELINES_CONFIG_URL}/environment").content.split('\n').each { l ->
        l = l.trim(); if (l && !l.startsWith('#')) { env["${l.split('=')[0].trim()}"] = "${l.split('=')[1].trim()}" }
    }
}

def msg
def artifactId
def additionalArtifactIds
def taskId
def allTaskIds = [] as Set

pipeline {

    agent {
        label 'rpmdeplint-trigger'
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '45', artifactNumToKeepStr: '100'))
        skipDefaultCheckout()
    }

    triggers {
       ciBuildTrigger(
           noSquash: true,
           providerList: [
               rabbitMQSubscriber(
                   name: env.FEDORA_CI_MESSAGE_PROVIDER,
                   overrides: [
                       topic: 'org.fedoraproject.prod.bodhi.update.status.testing.koji-build-group.build.complete',
                       queue: 'osci-pipelines-queue-15'
                   ],
                   checks: [
                       [field: '$.artifact.release', expectedValue: env.FEDORA_CI_RAWHIDE_RELEASE_ID]
                   ]
               )
           ]
       )
    }

    parameters {
        string(name: 'CI_MESSAGE', defaultValue: '{}', description: 'CI Message')
    }

    stages {
        stage('Trigger Testing') {
            steps {
                script {
                    msg = readJSON text: CI_MESSAGE

                    if (msg) {

                        if (msg['artifact']['builds'].size() > 40) {
                            echo "There are way too many (${msg['artifact']['builds'].size()} > 40) builds in the update. Skipping..."
                            return
                        }

                        msg['artifact']['builds'].each { build ->
                            allTaskIds.add(build['task_id'])
                        }

                        if (allTaskIds) {
                            taskId = allTaskIds[0]
                            artifactId = "koji-build:${taskId}"
                            additionalArtifactIds = allTaskIds.findAll{ it != taskId }.collect{ "koji-build:${it}" }.join(',')

                            build job: 'fedora-ci/rpmdeplint-pipeline/master', wait: false, parameters: [ string(name: 'ARTIFACT_ID', value: artifactId), string(name: 'ADDITIONAL_ARTIFACT_IDS', value: additionalArtifactIds) ]
                        }
                    }
                }
            }
        }
    }
}
