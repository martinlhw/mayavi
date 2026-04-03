pipeline {
    // Inline pod: sonar-scanner for analysis, gcloud for Dataproc/GCS operations.
    // Both containers share the same workspace volume so git checkout is visible to all.
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli:latest
    command: [cat]
    tty: true
  - name: gcloud
    image: google/cloud-sdk:slim
    command: [cat]
    tty: true
'''
            defaultContainer 'gcloud'
        }
    }

    environment {
        SONAR_PROJECT_KEY  = "mayavi-analysis"
        SONAR_PROJECT_NAME = "Mayavi Static Analysis"
        // Injected via JCasC globalNodeProperties — available on all agent pods
        SONARQUBE_URL    = "${env.SONARQUBE_URL}"
        DATAPROC_REGION  = "${env.DATAPROC_REGION}"
        DATAPROC_CLUSTER = "${env.DATAPROC_CLUSTER_NAME}"
        GCP_PROJECT      = "${env.GCP_PROJECT_ID}"
        HADOOP_BUCKET    = "${env.HADOOP_BUCKET}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Checked out: ${env.GIT_COMMIT}"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('sonar-scanner') {
                    withSonarQubeEnv('SonarQube') {
                        withCredentials([usernamePassword(
                            credentialsId: 'sonarqube-admin-password',
                            usernameVariable: 'SONAR_USER',
                            passwordVariable: 'SONAR_PASS'
                        )]) {
                            sh """
                                sonar-scanner \
                                  -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                  -Dsonar.projectName="${SONAR_PROJECT_NAME}" \
                                  -Dsonar.sources=. \
                                  -Dsonar.inclusions="**/*.py" \
                                  -Dsonar.python.version=3 \
                                  -Dsonar.login=${SONAR_USER} \
                                  -Dsonar.password=${SONAR_PASS}
                            """
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            echo "Quality Gate status: ${qg.status}"
                            echo "SonarQube analysis found blocker/critical issues. Hadoop job will NOT run."
                            currentBuild.result = 'UNSTABLE'
                            error("Stopping pipeline: SonarQube quality gate failed (${qg.status})")
                        } else {
                            echo "Quality Gate PASSED — no blocker issues found."
                        }
                    }
                }
            }
        }

        stage('Prepare Hadoop Input') {
            steps {
                echo "Uploading repository files to GCS for Hadoop input..."
                sh """
                    gsutil -m rsync -r -x '.*\\.git.*' . gs://${HADOOP_BUCKET}/input/

                    echo "Files uploaded to gs://${HADOOP_BUCKET}/input/"
                    gsutil ls gs://${HADOOP_BUCKET}/input/ | head -20
                """
            }
        }

        stage('Run Hadoop MapReduce Job') {
            steps {
                echo "Submitting Hadoop Streaming MapReduce job to Dataproc..."
                sh """
                    # Remove any previous output
                    gsutil -m rm -rf gs://${HADOOP_BUCKET}/output/ || true

                    gcloud dataproc jobs submit hadoop \
                        --cluster=${DATAPROC_CLUSTER} \
                        --region=${DATAPROC_REGION} \
                        --project=${GCP_PROJECT} \
                        --jar=file:///usr/lib/hadoop/hadoop-streaming.jar \
                        -- \
                        -D mapreduce.input.fileinputformat.input.dir.recursive=true \
                        -files gs://${HADOOP_BUCKET}/jobs/mapper.py,gs://${HADOOP_BUCKET}/jobs/reducer.py \
                        -mapper  "python3 mapper.py" \
                        -reducer "python3 reducer.py" \
                        -input   gs://${HADOOP_BUCKET}/input/ \
                        -output  gs://${HADOOP_BUCKET}/output/
                """
            }
        }

        stage('Display Results') {
            steps {
                echo "=== Hadoop MapReduce Results: Line Count Per File ==="
                sh """
                    gsutil cat gs://${HADOOP_BUCKET}/output/part-* | sort
                """
                echo "Results saved at gs://${HADOOP_BUCKET}/output/"
            }
        }
    }

    post {
        unstable {
            echo "Pipeline marked UNSTABLE: SonarQube quality gate blocked Hadoop execution."
        }
        failure {
            echo "Pipeline FAILED. Check SonarQube for analysis details."
        }
        success {
            echo "Pipeline SUCCEEDED: Hadoop job ran and results are stored in gs://${HADOOP_BUCKET}/output/"
        }
    }
}
