#!/usr/bin/env groovy

def triggers = []
def errors = []
def nodes = 'Android && 25.0.1'

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
    nodes = nodes + " && TICS"
}

echo "Node : ${nodes}"

properties([
    [$class: 'ParametersDefinitionProperty', parameterDefinitions: [[$class: 'BooleanParameterDefinition', defaultValue: false, description: 'For feature branches mark the check box to build an APK file', name : 'createAPK']]],
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
      return "DevTst"
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
        return "Tst"
    }
    if (env.BRANCH_NAME =~ /release\/.*/) {
        return "Production"
    }
    if (env.BRANCH_NAME =~ /build\/.*/) {
        return "Production"
    }
    return "Dev"
}

def buildAPK(appRoot, version, buildFlavor, config) {
    sh """#!/bin/bash -le
        echo \"Building VERSION ${version} for FLAVOR ${buildFlavor} with CONFIG ${config}\"
        cd ${appRoot}
        rm -f appFramework/build/outputs/apk/*.apk
        ./gradlew assemble${buildFlavor}${config}
        mv appFramework/build/outputs/apk/*.apk HSS_${version}_${buildFlavor}_${config}.apk
    """
}

node(nodes) {
    timestamps {
        def APP_ROOT = "Source/AppFramework"
        def BUILD_FLAVOR = getBuildFlavor()
        def UNIT_TEST_FLAVOR = getUnitTestFlavor()
        def CONFIG = "Release"
        def VERSION = ""
        def ANDROID_RELEASE_CANDIDATE = ""
        def ANDROID_VERSION_CODE = ""
        def STARTED_BY_TIMER = isJobStartedByTimer()
        def startBuildAPK = true
        def stage_tics = "false"

        stage('Checkout') {
            step([$class: 'WsCleanup', deleteDirs: true, notFailBuild: true])
            checkout([$class: 'GitSCM', branches: [[name: '*/'+env.BRANCH_NAME]], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'CheckoutOption', timeout: 30], [$class: 'LocalBranch' , localBranch: "**"], [$class: 'WipeWorkspace'], [$class: 'PruneStaleBranch'], [$class: 'SubmoduleOption', disableSubmodules: false, parentCredentials: true, recursiveSubmodules: true, reference: '', trackingSubmodules: false]], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'bbd4d9e8-2a6c-4970-b856-4e4cf901e857', url: 'ssh://git@bitbucket.atlas.philips.com:7999/pow/pwr_sleep_app-framework_android.git']]])
            step([$class: 'StashNotifier'])
        }

        try {
            stage('Prep Env') {
                // Make scripts executable
                sh """#!/bin/bash -le
                    chmod +x git_version.sh
                    chmod +x set_version.sh
                    chmod +x ${APP_ROOT}/gradlew
                    cd ${APP_ROOT}/ && ./gradlew clean && cd -
                """

                // Determine version
                switch (BUILD_FLAVOR) {
                    case "DevTst":
                        VERSION = sh(returnStdout: true, script: './git_version.sh snapshot').trim()
                        break
                    case "EvalStagingProduction":
                        VERSION = sh(returnStdout: true, script: './git_version.sh rc').trim()
                        break
                    default:
                        VERSION = sh(returnStdout: true, script: './git_version.sh snapshot').trim()
                        break
                }

                currentBuild.description = VERSION
                ANDROID_RELEASE_CANDIDATE = sh(returnStdout: true, script: "echo ${VERSION} | cut -d. -f4").trim()
                ANDROID_RELEASE_CANDIDATE = ("0000" + ANDROID_RELEASE_CANDIDATE).substring(ANDROID_RELEASE_CANDIDATE.length())
                ANDROID_VERSION_CODE = sh(returnStdout: true, script: "echo ${VERSION} | cut -d- -f1 | sed 's/[^0-9]*//g'").trim()
                ANDROID_VERSION_CODE = (ANDROID_VERSION_CODE + ANDROID_RELEASE_CANDIDATE).toInteger()

                // Print environment
                sh """#!/bin/bash -le
                    echo "---------------------- Printing Environment --------------------------"
                    env | sort
                    echo "----------------------- End of Environment ---------------------------"

                    echo "Android Release Candidate: ${ANDROID_RELEASE_CANDIDATE}"
                    echo "Android Version Code: ${ANDROID_VERSION_CODE}"

                    ./set_version.sh ${VERSION} ${ANDROID_VERSION_CODE}
                """

                def CreateAPK = env.createAPK
                try{ echo "MyParam is:"+ createAPK } catch(ex) { CreateAPK = "false" }
                echo "CreateAPK = ${CreateAPK}"

                if (CreateAPK == "false") {
                    result = sh (script: "git log -1 --pretty=%B | grep '#APK'", returnStatus: true)
                    if (result == 0) {
                        echo "Overrule Create APK..."
                        CreateAPK = "true"
                    }
                }

                if (env.BRANCH_NAME =~ /bugfix\/.*|spike\/.*|feature\/.*/) {
                    echo env.BRANCH_NAME
                    if (CreateAPK == "false") {
                        echo "APK file not build."
                        startBuildAPK = false
                    }
                }
            }

            stage ('Unit Test') {
                sh """#!/bin/bash -le
                    cd ${APP_ROOT}
                    ./gradlew test${UNIT_TEST_FLAVOR}${CONFIG}
                """
                step([$class: 'JUnitResultArchiver', testResults: APP_ROOT + '/*/build/test-results/*/*.xml'])
            }

            stage ('Code Quality') {
                sh """#!/bin/bash -le
                    cd ${APP_ROOT}
                    ./gradlew codeQuality${UNIT_TEST_FLAVOR}
                """

                // Lint
                step([$class: 'LintPublisher', healthy: '0', unHealthy: '160', unstableTotalAll: '150'])

                // FindBugs
                publishHTML(target: [alwaysLinkToLastBuild: false, reportDir: APP_ROOT + '/appFramework/build/reports/findbugs', reportFiles:'findbugs*.html', reportName: 'Findbugs Analysis'])

                // PMD
                publishHTML(target: [reportDir: APP_ROOT + '/appFramework/build/reports/pmd', reportFiles:'pmd*.html', reportName: 'PMD Analysis'])

                // JDepend
                publishHTML(target: [reportDir: APP_ROOT + '/appFramework/build/reports/jdepend', reportFiles:'jdepend*.html', reportName: 'JDepend Analysis'])
            }

            stage ('Build') {
                if (startBuildAPK) {
                    // Execute build
                    switch (BUILD_FLAVOR) {
                        case "DevTst":
                            // Build Dev
                            BUILD_FLAVOR = "Dev"
                            buildAPK(APP_ROOT, VERSION, BUILD_FLAVOR, CONFIG)
                            // Build Test
                            BUILD_FLAVOR = "Tst"
                            buildAPK(APP_ROOT, VERSION, BUILD_FLAVOR, CONFIG)
                            break
                        case "Dev":
                            // Build Dev
                            buildAPK(APP_ROOT, VERSION, BUILD_FLAVOR, CONFIG)
                            break
                        case "EvalStagingProduction":
                            // Build Eval
                            BUILD_FLAVOR = "Eval"
                            buildAPK(APP_ROOT, VERSION, BUILD_FLAVOR, CONFIG)
                            // Build Staging
                            BUILD_FLAVOR = "Staging"
                            buildAPK(APP_ROOT, VERSION, BUILD_FLAVOR, CONFIG)
                            // Build Production
                            ANDROID_RELEASE_CANDIDATE = sh(returnStdout: true, script: "echo ${VERSION} | cut -d- -f1").trim()
                            BUILD_FLAVOR = "Production"
                            sh """#!/bin/bash -le
                                ./set_version.sh ${ANDROID_RELEASE_CANDIDATE} ${ANDROID_VERSION_CODE}
                            """
                            buildAPK(APP_ROOT, ANDROID_RELEASE_CANDIDATE, BUILD_FLAVOR, CONFIG)
                            break
                         default:
                            // do nothing
                            break
                    }

                    // Archive APK files
                    step([$class: 'ArtifactArchiver', artifacts:  APP_ROOT + '/*.apk', excludes: null, fingerprint: true, onlyIfSuccessful: true])

                    // Archive Mappings files
                    step([$class: 'ArtifactArchiver', artifacts:  APP_ROOT + '/appFramework/build/mapping/mapping-*.txt', excludes: null, fingerprint: true, onlyIfSuccessful: true])
                }
            }

            stage ('Reporting') {
                sh """#!/bin/bash -le
                    cd ${APP_ROOT}
                    ./gradlew jacoco${BUILD_FLAVOR}TestReport
                """

                step([$class: 'JacocoPublisher', execPattern: '**/*.exec', classPattern: '**/classes/' + UNIT_TEST_FLAVOR.toLowerCase() +'/' + CONFIG.toLowerCase() + '/com/philips', sourcePattern: '**/src/main/java', exclusionPattern: '**/R.class, **/R$*.class, */BuildConfig.class, **/Manifest*.*, **/*_Factory.class, **/*_*Factory.class , **/Dagger*.class, **/databinding/**/*.class, **/*Activity*.*, **/*Fragment*.*, **/*Service*.*, **/*ContentProvider*.*'])

                publishHTML(target: [keepAll: true, alwaysLinkToLastBuild: false, reportDir: APP_ROOT + '/appFramework/build/reports/jacoco/jacoco' + UNIT_TEST_FLAVOR + 'TestReport/html', reportFiles:'index.html', reportName: 'Overall code coverage'])

                if (startBuildAPK) {
                    // DexCount
                    publishHTML(target: [reportDir: APP_ROOT + '/appFramework/build/outputs/dexcount/' +  BUILD_FLAVOR.toLowerCase() + CONFIG + 'Chart', reportFiles:'index.html', reportName: 'Dexcount'])
                }

                sh """#!/bin/bash -le
                    cd ${APP_ROOT}
                    ./gradlew htmlDependencyReport
                """

                // Dependencies
                publishHTML(target: [reportDir: APP_ROOT + '/appFramework/build/reports/project/dependencies/', reportFiles:'index.html', reportName: 'Dependencies'])
            }

            stage('TICS') {
                if(STARTED_BY_TIMER) {
            		    echo "Running TICS..."
            		    stage_tics = "true"
                    sh """#!/bin/bash -le
            		        TICSMaintenance -project HealthySleep-Android -branchname develop -branchdir .
                		    TICSQServer -project HealthySleep-Android -tmpdir /mnt/tics/temp/HealthySleep-Android
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