pipeline {
    agent any

    environment {
        PROJECT_ID = 'cmu-14848-471600'
        BUCKET_DATA = 'gs://cpo1-data'
        BUCKET_SCRIPTS = 'gs://cpo1-countlinescripts'
        REGION = 'northamerica-northeast2'
        ZONE = 'northamerica-northeast2-c'
        CLUSTER = 'cpo1-dataprocclusterlocal'
        VERSION = 'v0.0.1'

        SONAR_TOKEN = credentials('sonar-token-id') 
    }

    stages {

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') { 
                    sh '''
                        /opt/sonar-scanner/bin/sonar-scanner \
                          -Dsonar.projectKey=test-2 \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=http://34.69.251.47:9000 \
                          -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Wait for Quality Gate') {
            steps {
                script {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "SonarQube quality gate failed: ${qg.status}"
                    }
                }
            }
        }

        stage('Upload Repo Files to GCS') {
            steps {
                echo 'Uploading all repo files to GCS...'
                sh '''
                    gsutil -m cp -r ./* "$BUCKET_DATA"
                '''
            }
        }

        stage('Generate and Upload filelist.txt') {
            steps {
                echo 'Generating filelist.txt and uploading...'
                sh '''
                    gsutil ls "$BUCKET_DATA/**" \
                        | grep -v '/\\.[^/]*$' \
                        | grep -v '__' \
                        | grep -v 'temp_merge/' \
                        | grep -v 'merged_file.txt' \
                        | grep -v 'output_dir.txt' \
                        > filelist.txt

                    gsutil cp filelist.txt "$BUCKET_SCRIPTS/filenames/filelist.txt"
                '''
            }
        }

        stage('Run Dataproc Job') {
            steps {
                echo 'Running Hadoop streaming job on Dataproc...'
                sh '''
                    OUTPUT_DIR="$BUCKET_SCRIPTS/outputs/$(date +%s)"

                    gcloud dataproc jobs submit hadoop \
                        --cluster=$CLUSTER \
                        --region=$REGION \
                        --jar=file:///usr/lib/hadoop/hadoop-streaming.jar \
                        -- -files "$BUCKET_SCRIPTS/$VERSION/mapper.py","$BUCKET_SCRIPTS/$VERSION/reducer.py" \
                           -mapper mapper.py \
                           -reducer reducer.py \
                           $(awk '{print "-input \\""$1"\\""}' filelist.txt) \
                           -output $OUTPUT_DIR

                    echo $OUTPUT_DIR > output_dir.txt
                '''
            }
        }

        stage('Merge Outputs') {
            steps {
                echo 'Merging all output files into one...'
                sh '''
                    OUTPUT_DIR=$(cat output_dir.txt)
                    rm -rf temp_merge merged_file.txt || true
                    mkdir temp_merge
                    gsutil -m cp "$OUTPUT_DIR/**" temp_merge
                    cat temp_merge/* | sed 's#gs://cpo1-data/##' > merged_file.txt
                    echo "Merged file contents:"
                    cat merged_file.txt
                    rm -rf temp_merge
                    rm merged_file.txt
                    rm output_dir.txt
                '''
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed xxx'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
    }
}

