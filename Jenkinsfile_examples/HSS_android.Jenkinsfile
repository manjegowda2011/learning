#!/usr/bin/env groovy

def triggers = []
def errors = []

@NonCPS
def isJobStartedByTimer() {
    def startedByTimer = false
    try {
        def buildCauses = currentBuild.rawBuild.getCauses()
        for(buildCause in buildCauses) {
            if(buildCause != null) {
                def causeDescription = buildCause.getShortDescription()
                if(causeDescription.contains("Started by timer")) {
                    startedByTimer = true
                }
            }
        }
    } catch(err) {
        echo "Error determining build cause"
    }
    return startedByTimer
}

if (env.BRANCH_NAME == "feature_powersleep_develop") {
    triggers << cron('H H(18-20) * * *')
}

properties([
    [$class: 'ParametersDefinitionProperty', parameterDefinitions: [[$class: 'BooleanParameterDefinition', defaultValue: false, description: 'For feature branches mark the check box to build an IPA file', name : 'createIPA']]],
    [$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', numToKeepStr: '5']],
    pipelineTriggers(triggers),
])

def getBuildFlavor() {
    if (env.BRANCH_NAME == "feature_powersleep_develop") {
        return "Dev"
    }
    if (env.BRANCH_NAME =~ /feature\/.*/) {
        return "Dev"
    }
    if (env.BRANCH_NAME =~ /codefreeze\/.*/) {
        return "DevTest"
    }
    if (env.BRANCH_NAME =~ /release\/.*/) {
        return "EvalStagingProduction"
    }
    if (env.BRANCH_NAME =~ /build\/.*/) {
        return "EvalStagingProduction"
    }
    return "Dev"
}

def getUnitTestFlavor() {
    if (env.BRANCH_NAME == "feature_powersleep_develop") {
        return "Dev"
    }
    if (env.BRANCH_NAME =~ /feature\/.*/) {
        return "Dev"
    }
    if (env.BRANCH_NAME =~ /codefreeze\/.*/) {
        return "Test"
    }
    if (env.BRANCH_NAME =~ /release\/.*/) {
        return "Production"
    }
    if (env.BRANCH_NAME =~ /build\/.*/) {
        return "Production"
    }
    return "Dev"
}


def buildIPA(appRoot, version, buildFlavor, config, releaseType) {
   sh """#!/bin/sh -le
       echo \"Building VERSION ${version} for FLAVOR ${buildFlavor} with CONFIG ${config}\"
       cd ${appRoot}
       export FASTLANE_HIDE_TIMESTAMP=true
       bundle exec fastlane ios release configuration:${buildFlavor} plist_path:AppFramework/SupportingFiles/Info.plist release_type:${releaseType} release_name:HSS_${version}_${buildFlavor}_${config}
       mv fastlane/build_output/*.ipa .
   """
}
node('iOS && HSS') {
    timestamps {
        def APP_ROOT = "Source/App/AppFramework"
        def BUILD_FLAVOR = getBuildFlavor()
        def UNIT_TEST_FLAVOR = getUnitTestFlavor()
        def CONFIG = "Release"
        def VERSION = ""
        def PROJECT_TEMP_ROOT = ""
        def APPLE_BUNDLE_VERSION = ""
        def APPLE_SHORT_VERSION = ""
        def BUILD_SETTINGS = ""
        def SETTINGS = ""
        def STARTED_BY_TIMER = isJobStartedByTimer()
        def startBuildIPA = true
        def stage_tics = "false"

        stage('Checkout') {
            checkout scm
            // checkout([$class: 'GitSCM', branches: [[name: '*/'+env.BRANCH_NAME]], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'CheckoutOption', timeout: 30], [$class: 'WipeWorkspace'], [$class: 'PruneStaleBranch'], [$class: 'SubmoduleOption', disableSubmodules: false, parentCredentials: false, recursiveSubmodules: true, reference: '', trackingSubmodules: false]], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'bbd4d9e8-2a6c-4970-b856-4e4cf901e857', url: 'ssh://git@bitbucket.atlas.philips.com:7999/pow/pwr_sleep_app-framework_ios.git']]])
            step([$class: 'StashNotifier'])
        }

        try {
            stage('Prep Env') {
                // Ensure Dependencies
                sh """#!/bin/sh -le
                    cd Source
                    bundle install
                    brew bundle
                    export BUNDLE_GEMFILE=`pwd`/Gemfile
                    cd App/AppFramework
                    ./install_pods.sh
                """

                // Determine version
                switch (BUILD_FLAVOR) {
                    case ["Dev", "DevTest"]:
                        VERSION = sh(returnStdout: true, script: './git_version.sh snapshot').trim()
                        APPLE_SNAPSHOT_VERSION = sh(returnStdout: true, script: "echo ${VERSION} | cut -d. -f4").trim()
                        APPLE_SNAPSHOT_VERSION = ("0000" + APPLE_SNAPSHOT_VERSION).substring(APPLE_SNAPSHOT_VERSION.length())
                        break
                    case "EvalStagingProduction":
                        VERSION = sh(returnStdout: true, script: './git_version.sh rc').trim()
                        APPLE_SNAPSHOT_VERSION = sh(returnStdout: true, script: "echo ${VERSION} | cut -d. -f4").trim()
                        APPLE_SNAPSHOT_VERSION = ("00" + APPLE_SNAPSHOT_VERSION).substring(APPLE_SNAPSHOT_VERSION.length())
                        break
                    default:
                        VERSION = sh(returnStdout: true, script: './git_version.sh snapshot').trim()
                        break
                }
                currentBuild.description = VERSION
                APPLE_BUNDLE_VERSION = sh(returnStdout: true, script: "echo ${VERSION} | cut -d- -f1 | sed 's/[^0-9]*//g'").trim()
                APPLE_BUNDLE_VERSION = (APPLE_BUNDLE_VERSION + APPLE_SNAPSHOT_VERSION).toInteger()
                APPLE_SHORT_VERSION = sh(returnStdout: true, script: "echo ${VERSION} | cut -d- -f1").trim()

                // Print environment
                sh """#!/bin/sh -le
                    echo "---------------------- Printing Environment --------------------------"
                    env | sort
                    echo "----------------------- End of Environment ---------------------------"

                    echo "CFBundleVersion (a.k.a. build number): ${APPLE_BUNDLE_VERSION}"
                    echo "CFBundleShortVersionString (a.k.a. version number): ${APPLE_SHORT_VERSION}"

                    /usr/libexec/Plistbuddy -c "Set CFBundleVersion ${APPLE_BUNDLE_VERSION}" "${APP_ROOT}/AppFramework/SupportingFiles/Info.plist"
                    /usr/libexec/Plistbuddy -c "Set CFBundleShortVersionString ${APPLE_SHORT_VERSION}" "${APP_ROOT}/AppFramework/SupportingFiles/Info.plist"
                    /usr/libexec/Plistbuddy -c "Set CFJenkinsVersion ${VERSION}" "${APP_ROOT}/AppFramework/SupportingFiles/Info.plist"
                """

                def CreateIPA = env.createIPA
                try{ echo "MyParam is:"+ createIPA } catch(ex) { CreateIPA = "false" }
                echo "CreateIPA = ${CreateIPA}"

                if (CreateIPA == "false") {
                    result = sh (script: "git log -1 --pretty=%B | grep '#IPA'", returnStatus: true)
                    if (result == 0) {
                        echo "Overrule Create IPA..."
                        CreateIPA = "true"
                    }
                }

                if (env.BRANCH_NAME =~ /bugfix\/.*|spike\/.*|feature\/.*/) {
                    echo env.BRANCH_NAME
                    if (CreateIPA == "false") {
                        echo "IPA file not build."
                        startBuildIPA = false
                    }
                }
            }

            wrap([$class: 'AnsiColorBuildWrapper']) {
                stage('Unit Test') {
                    sh """#!/bin/sh -le
                        cd ${APP_ROOT}
                        export FASTLANE_HIDE_TIMESTAMP=true
                        bundle exec fastlane ios continuous configuration:${UNIT_TEST_FLAVOR}
                        killall "Simulator" 2> /dev/null; xcrun simctl erase all
                    """
                    publishHTML(target: [reportDir: APP_ROOT + '/fastlane/build_output', reportFiles: 'report.html', reportName: 'Unit Tests'])
                    publishHTML(target: [reportDir: APP_ROOT + '/fastlane/build_output/coverage', reportFiles: 'index.html', reportName: 'Code Coverage'])
                }

                stage('Analyse') {
                    sh """#!/bin/sh -le
                        cd Source/App/AppFramework
                        export FASTLANE_HIDE_TIMESTAMP=true
                        bundle exec fastlane ios analyse_swift
                    """
                    publishHTML(target: [reportDir: APP_ROOT + '/fastlane/build_output', reportFiles: 'report.swiftlint', reportName: 'Swift Lint'])
                }

                stage('Build') {
                    if (startBuildIPA) {
                        // Execute build
                        switch (BUILD_FLAVOR) {
                            case "DevTest":
                                // Build Test
                                buildIPA(APP_ROOT, VERSION, "Test", CONFIG, "enterprise")
                                // Build Dev
                                buildIPA(APP_ROOT, VERSION, "Dev", CONFIG, "enterprise")
                                break
                            case "Dev":
                                // Build Dev
                                buildIPA(APP_ROOT, VERSION, "Dev", CONFIG, "enterprise")
                                break
                            case "EvalStagingProduction":
                                // Build Eval
                                buildIPA(APP_ROOT, VERSION, "Eval", CONFIG, "app-store")
                                // Build Staging
                                buildIPA(APP_ROOT, VERSION, "Staging", CONFIG, "enterprise")
                                // Build Production
                                sh """#!/bin/sh -le
                                    /usr/libexec/Plistbuddy -c "Set CFJenkinsVersion ${APPLE_SHORT_VERSION}" "${APP_ROOT}/AppFramework/SupportingFiles/Info.plist"
                                """
                                buildIPA(APP_ROOT, VERSION, "Production", CONFIG, "app-store")
                                break
                             default:
                                // do nothing
                                break
                        }

                        // Archive IPA files
                        step([$class: 'ArtifactArchiver', artifacts: APP_ROOT + '/*.ipa', excludes: null, fingerprint: true, onlyIfSuccessful: true])
                        step([$class: 'ArtifactArchiver', artifacts: APP_ROOT + '/Podfile.lock', excludes: null, fingerprint: true, onlyIfSuccessful: true])
                    }
                }
            }

            if(STARTED_BY_TIMER) {
                stage('TICS') {
                    echo "Running TICS..."
                    stage_tics = "true"
                    sh """#!/bin/sh -le
                        ./translate-llmv-cov.sh
                        TICSMaintenance -project HealthySleep-iOS -branchname develop -branchdir .
                        TICSQServer -project HealthySleep-iOS -nosanity -tmpdir ./HealthySleep_iOS
                       ./CopyTICSLog2TICSserver.sh
                    """
                }
            }

            currentBuild.result = 'SUCCESS'
        } catch(err) {
            errors << "errors found: ${err}"
        } finally {
            if (errors.size() > 0) {
                stage ('error reporting') {
                    if(stage_tics == "true") {
                        currentBuild.result = 'UNSTABLE'
                    } else {
                        currentBuild.result = 'FAILED'
                    }
                    for (int i = 0; i < errors.size(); i++) {
                        echo errors[i];
                    }
                }
            }

            stage('Send Notifications') {
                step([$class: 'StashNotifier'])
                step([$class: 'Mailer', notifyEveryUnstableBuild: true, recipients: emailextrecipients([[$class: 'CulpritsRecipientProvider'], [$class: 'RequesterRecipientProvider']])])
            }

            stage('Cleaning workspace slave') {
               step([$class: 'WsCleanup', deleteDirs: true, notFailBuild: true])
            }
        }
    }
}


node('master') {
    stage('Cleaning workspace master') {
        def wrk = pwd() + "@script/"
        dir("${wrk}") {
            deleteDir()
        }
    }
}