pipeline {
    agent any
    environment {
        AWS_REGION = "us-east-1"
        S3_BUCKET_APKS = "saitama-apks-balti"
        S3_BUCKET_SITE = "saitama-site-balti"
        CLOUDFRONT_ID = ""
        //SLACK_WEBHOOK = credentials('slack-webhook') instead, using Bot as per Slack Notification plugin doc
        AWS_CREDS_ID = "aws-credential
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'add-jenkinsfile', url: 'git@github.com:radicalZer0/Saitama.git', credentialsId: 'saitama-ssh-key'
                //scm checkout #after adding github webhook
            }
        }

        stage('Set Bump Type') {
            steps {
                script {
                    def commitMsg = sh(script: "git log -1 --pretty=%s", returnStdout: true).trim()
                    echo "Latest commit message: $commitMsg"

                    if (commitMsg =~ /(?i)\bmajor\b/) {
                        env.BUMP_TYPE = "major"
                    } else if (commitMsg =~ /(?i)\bfeat\b/) {
                            env.BUMP_TYPE = "minor"

                    } else if (commitMsg =~ /(?i)\bfix\b/) {
                        env.BUMP_TYPE = "patch"
                    } else {
                        env.BUMP_TYPE = "none"
                    }
                    echo "${BUMP_TYPE}"
                }
            }
        }

        stage('Versioning') {
            when { expression { env.BUMP_TYPE != "none" } }
            steps {
                script {
                    def currentVersion = readFile('version.txt').trim()
                    echo "Current version: v${currentVersion}"

                    def parts = currentVersion.tokenize('.')
                    int major = parts[0].toInteger()
                    int minor = parts[1].toInteger()
                    int patch = parts[2].toInteger()

                    if (env.BUMP_TYPE == "major") {
                        major++; minor = 0; patch = 0
                    } else if (env.BUMP_TYPE == "minor") {
                        minor++; patch = 0
                    } else {
                        patch++
                    }

                    env.APP_VERSION = "${major}.${minor}.${patch}"
                    writeFile file: 'version.txt', text: env.APP_VERSION
                    echo "New version: v${env.APP_VERSION}"
                }
            }
        }

        stage('Build APK') {
            when { expression { env.BUMP_TYPE != "none" } }
            steps {
                sh './gradlew assembleRelease --no-daemon'
            }
        }

        stage('Upload to S3') {
            when { expression { env.BUMP_TYPE != "none" } }
            steps {
                withCredentials([[ $class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'my_aws_credential']]){
                    script {
                        def apkPath = "app/build/output/apk/release/app-release.apk"
                        def versionFileName = "saitama-v${APP_VERSION}.apk"

                        sh """
                            set -euo pipefail
                            aws s3 cp ${apkPath} s3://${S3_BUCKET_APKS}/releases/${versionFileName} --region ${AWS_REGION}
                            aws s3 cp ${apkPath} s3://${S3_BUCKET_APKS}/latest/saitama-latest.apk --region ${AWS_REGION}
                            #create versions.json

                            aws s3api list-objects \
                                --bucket "${S3_BUCKET_APKS}" \
                                --prefix "releases/" \
                                --query "Contents[].Key" \
                                --region "$AWS_REGION" \
                            | jq -r '.[]' \
                            | sed 's|releases/||; s|.apk\$||' \
                            | sort -V \
                            | tail -n 5 \
                            | jq -R -s -c '
                                split(\"\\n\")[:-1] |
                                map({
                                    version: .,
                                    url: \"https://${S3_BUCKET_APKS}.s3.${AWS_REGION}.amazonaws.com/releases/saitama-\" + .
                                }) > versions.json

                            aws s3 cp versions.json s3://${S3_BUCKET_SITE}/versions.json
                        """
                    }
                }
            }
        }

        //stage('Commit and Push back')

        stage('Archive APK') {
            when { expression { env.BUMP_TYPE != "none" } }
            steps {
                archiveArtifacts artifacts: 'app/build/outputs/apk/release/*.apk', fingerprint: true
            }
        }

        stage('Notify Success') {
            when { expression { env.BUMP_TYPE != "none" } }
            steps {
                script {
                    def attachments = [
                      [
                        text: 'New APK Released!\\nVersion: v{APP_VERSION}',
                        fallback: 'Hey, Vader seems to be mad at you.',
                        color: '#ff0000'
                      ]
                    ]

                    slackSend(channel: "#saitama", attachments: attachments)
                }
            }
        }

        stage('Notify Skipped Build') {
            when { expression { env.BUMP_TYPE != "none" } }
            steps {
                script {
                    def attachments = [
                      [
                        text: 'No version bump, No APK built for commit: ${commitMsg}',
                        fallback: 'Hey, Vader seems to be mad at you.',
                        color: '#ff0000'
                      ]
                    ]

                    slackSend(channel: "#saitama", attachments: attachments)
                }
            }
        }
    }

    post {
        failure {
            script {
                def attachments = [
                    [
                        text: 'Build Failed',
                        fallback: 'Hey, Vader seems to be mad at you.',
                        color: '#ff0000'
                      ]
                    ]

                    slackSend(channel: "#saitama", attachments: attachments)
            }
        }
        always {
            echo "Cleaning Workspace"
            cleanWs()
        }
    }
}
