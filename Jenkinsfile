// Copyright 2019 Nikola Ružić
pipeline {
    agent {
        node {
            label 'linux'
        }
    }
    options {
        buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5', daysToKeepStr: '90',
            artifactDaysToKeepStr: '90'))
        disableConcurrentBuilds()
    }
    parameters {
        booleanParam(name: 'IS_RELEASE_BUILD', defaultValue: false, description: 'Perform release build')
        booleanParam(name: 'DRY_RUN', defaultValue: false,
            description: 'During release do not checkin or tag anything in the scm repository, or modify the checkout')
        string(name: 'RELEASE_VERSION', defaultValue: 'N/A',
            description: 'Version to release (leave default for non release builds)')
        string(name: 'NEXT_DEVELOPMENT_VERSION', defaultValue: 'N/A',
            description: 'Next development version (leave default for non release builds)')
    }
    stages {
        stage('Prepare Build') {
            steps {
                // Copy settings files and keys
                configFileProvider([
                    configFile(fileId: '4dd159c7-33ac-4261-9c4c-3e8449945dc4', targetLocation: 'settings.xml'),
                    configFile(fileId: 'df225450-f81f-4698-b24d-ccdda7653b78',
                        targetLocation: 'settings-security.xml'),
                    configFile(fileId: 'foo.pub.asc', targetLocation: 'foo.pub.asc'),
                    configFile(fileId: 'bar.sub.asc', targetLocation: 'bar.sub.asc'),
                    configFile(fileId: 'c498ec5e-37fd-4376-a00f-94f1bbcc3c2a',
                        targetLocation: 'baz-id-rsa')]) {
                        // some block
                }
                // Import keys to system
                sh 'gpg --import foo.sub.asc || true'
                sh 'gpg --import bar.pub.asc || true'
                // Resolve project release versions
                script {
                    def releaseVersion = 'N/A'
                    def nextDevelopmentVersion = 'N/A'
                    // Use maven plugin to resolve versions
                    withMaven(jdk: 'jdk1.8.0_181', maven: 'apache-maven-3.6.0',
                        mavenSettingsConfig: '4dd159c7-33ac-4261-9c4c-3e8449945dc4',
                        mavenLocalRepo: '$WORKSPACE/.repository') {
                        // Use $MAVEN_HOME/bin/mvn to avoid additional output on sysout produced by withMaven()
                        // But we need withMaven() to ensure maven is installed on target node
                        def maven = '$MAVEN_HOME/bin/mvn'
                        def settings = '$WORKSPACE/settings.xml'
                        def plugin = 'org.codehaus.mojo:build-helper-maven-plugin:3.0.0:parse-version'
                        def goal = 'help:evaluate'
                        def std = '-DforceStdout'
                        def cmd = "$maven -q -s $settings $plugin $goal $std"
                        def majorVersion = sh returnStdout: true,
                            script: "$cmd -Dexpression=parsedVersion.majorVersion"
                        def minorVersion = sh returnStdout: true,
                            script: "$cmd -Dexpression=parsedVersion.minorVersion"
                        def incrementalVersion = sh returnStdout: true,
                            script: "$cmd -Dexpression=parsedVersion.incrementalVersion"
                        def qualifierVersion = sh returnStdout: true,
                            script: "$cmd -Dexpression=parsedVersion.qualifierVersion"
                        def buildNumberVersion = sh returnStdout: true,
                            script: "$cmd -Dexpression=parsedVersion.buildNumberVersion"
                        def nextMajorVersion = sh returnStdout: true,
                            script: "$cmd -Dexpression=parsedVersion.nextMajorVersion"
                        def nextMinorVersion = sh returnStdout: true,
                            script: "$cmd -Dexpression=parsedVersion.nextMinorVersion"
                        def nextIncrementalVersion = sh returnStdout: true,
                            script: "$cmd -Dexpression=parsedVersion.nextIncrementalVersion"
                        def nextBuildNumberVersion = sh returnStdout: true,
                            script: "$cmd -Dexpression=parsedVersion.nextBuildNumberVersion"
                        // Set versions
                        releaseVersion = "$majorVersion.$minorVersion.$incrementalVersion"
                        nextDevelopmentVersion = "$majorVersion.$minorVersion.$nextIncrementalVersion-SNAPSHOT"
                    }
                    // Redefine properties
                    properties([
                        parameters([
                            booleanParam(name: 'IS_RELEASE_BUILD', defaultValue: false,
                                description: 'Perform release build'),
                            booleanParam(name: 'DRY_RUN', defaultValue: false,
                                description: 'During release do not checkin or tag anything in the scm repository'
                                     + ', or modify the checkout'),
                            string(name: 'RELEASE_VERSION', defaultValue: releaseVersion,
                                description: 'Version to release (leave default for non release builds)'),
                            string(name: 'NEXT_DEVELOPMENT_VERSION', defaultValue: nextDevelopmentVersion,
                                description: 'Next development version (leave default for non release builds)')
                        ])
                    ])
                }
            }
        }
        stage('Build') {
            steps {
                withMaven(jdk: 'jdk1.8.0_181', maven: 'apache-maven-3.6.0',
                    mavenSettingsConfig: '4dd159c7-33ac-4261-9c4c-3e8449945dc4',
                    mavenLocalRepo: '$WORKSPACE/.repository') {
                    sh ("mvn clean deploy"
                        + " -DreleaseTasks"
                        + " -Dbranch=$BRANCH_NAME"
                        + " -Dsettings.security=$WORKSPACE/settings-security.xml"
                        + " -Dmaven.test.failure.ignore=true"
                        + " -Dcheckstyle.failOnViolation=false"
                    )
                }
            }
            post {
                success {
                    junit testResults: 'target/surefire-reports/**/TEST-*.xml,target/failsafe-reports/**/*.xml',
                        allowEmptyResults: true
                }
            }
        }
        stage('Site') {
           steps {
                withMaven(jdk: 'jdk1.8.0_181', maven: 'apache-maven-3.6.0',
                    mavenSettingsConfig: '4dd159c7-33ac-4261-9c4c-3e8449945dc4',
                    mavenLocalRepo: '$WORKSPACE/.repository') {
                    sh ("mvn site-deploy"
                        + " -DreleaseTasks"
                        + " -Dbranch=$BRANCH_NAME"
                        + " -Dsettings.security=$WORKSPACE/settings-security.xml"
                        + " -Dmaven.test.failure.ignore=true"
                        + " -Dcheckstyle.failOnViolation=false"
                    )
                }
            }
        }
        stage('Prepare Release') {
           when {
               expression {
                  return (params.IS_RELEASE_BUILD && params.RELEASE_VERSION != 'N/A'
                      && params.NEXT_DEVELOPMENT_VERSION != 'N/A')
               }
           }
           steps {
                withMaven(jdk: 'jdk1.8.0_181', maven: 'apache-maven-3.6.0',
                    mavenSettingsConfig: '4dd159c7-33ac-4261-9c4c-3e8449945dc4',
                    mavenLocalRepo: '$WORKSPACE/.repository') {
                    sh ("mvn release:prepare"
                        + " -Dbranch=$BRANCH_NAME"
                        + " -Dsettings-security=$WORKSPACE/settings-security.xml"
                        + " -DreleaseVersion=${params.RELEASE_VERSION}"
                        + " -Dtag='\${project.artifactId}-\${releaseVersion}'"
                        + " -DdevelopmentVersion=${params.NEXT_DEVELOPMENT_VERSION}"
                        + " -DdryRun=${params.DRY_RUN}"
                    )
                }
            }
        }
        stage('Perform Release') {
           when {
               expression {
                  return (params.IS_RELEASE_BUILD && params.RELEASE_VERSION != 'N/A'
                      && params.NEXT_DEVELOPMENT_VERSION != 'N/A')
               }
           }
           steps {
                withMaven(jdk: 'jdk1.8.0_181', maven: 'apache-maven-3.6.0',
                    mavenSettingsConfig: '4dd159c7-33ac-4261-9c4c-3e8449945dc4',
                    mavenLocalRepo: '$WORKSPACE/.repository') {
                    sh ("mvn release:perform"
                        + " -Dbranch=$BRANCH_NAME"
                        + " -Dsettings-security=$WORKSPACE/settings-security.xml"
                        + " -DreleaseVersion=${params.RELEASE_VERSION}"
                        + " -Dtag='\${project.artifactId}-\${releaseVersion}'"
                        + " -DdevelopmentVersion=${params.NEXT_DEVELOPMENT_VERSION}"
                        + " -DdryRun=${params.DRY_RUN}"
                    )
                }
            }
        }
    }
    post {
        always {
            recordIssues tool: mavenConsole(id: 'maven-console', name: 'Maven Console'), enabledForFailure: true
            recordIssues tool: tagList(pattern: '**/taglist.xml', reportEncoding: 'UTF-8'), enabledForFailure: false
            recordIssues tool: java(), enabledForFailure: false
            recordIssues tool: javaDoc(), enabledForFailure: false
            recordIssues tool: checkStyle(pattern: '**/checkstyle-result.xml', reportEncoding: 'UTF-8'),
                unstableTotalHigh: 1, unstableTotalNormal: 500, unstableTotalLow: 500,
                failedTotalHigh: 100, failedTotalNormal: 1000, failedTotalLow: 1000, enabledForFailure: false
        }
        changed {
            echo '--- Build Changed! ---'
            mail to: 'Team_foo@foo.com', subject: "[Build Changed] ${env.JOB_NAME}", mimeType: 'text/plain',
                body: """Buid number ${env.BUILD_ID} for job ${env.JOB_NAME} status changed from previous build!

Fore more details please take a look at ${env.BUILD_URL}"""
        }
        success {
            echo '--- Build Stable! ---'
            mail to: 'Team_foo@foo.com', subject: "[Build Stable] ${env.JOB_NAME}", mimeType: 'text/plain',
                body: """Buid number ${env.BUILD_ID} for job ${env.JOB_NAME} stable!

Fore more details please take a look at ${env.BUILD_URL}"""
        }
        unstable {
            echo '--- Build Unstable! ---'
            mail to: 'Team_foo@foo.com', subject: "[Build Unstable] ${env.JOB_NAME}", mimeType: 'text/plain',
                body: """Buid number ${env.BUILD_ID} for job ${env.JOB_NAME} unstable!

Fore more details please take a look at ${env.BUILD_URL}"""
        }
        failure {
            echo '--- Build Failure! ---'
            mail to: 'Team_foo@foo.com', subject: "[Build Failure] ${env.JOB_NAME}", mimeType: 'text/plain',
                body: """Buid number ${env.BUILD_ID} for job ${env.JOB_NAME} failed!

Fore more details please take a look at ${env.BUILD_URL}"""
        }
    }
}
