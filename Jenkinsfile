pipeline {
    agent none
    options {
        ansiColor("xterm")
        timeout(time: 6, unit: "HOURS")
        timestamps()
    }
    parameters {
        choice(name: "BUILD_TYPE", choices: ["Release", "Debug"], description: "")
        choice(name: "CHANNEL", choices: ["nightly", "dev", "beta", "release", "development"], description: "")
        string(name: "SLACK_BUILDS_CHANNEL", defaultValue: "#build-downloads-bot", description: "The Slack channel to send the list of artifact download links to. Leave blank to skip sending the message.")
        booleanParam(name: "SKIP_SIGNING", defaultValue: true, description: "")
        booleanParam(name: "WIPE_WORKSPACE", defaultValue: false, description: "")
        booleanParam(name: "SKIP_INIT", defaultValue: false, description: "")
        booleanParam(name: "DISABLE_SCCACHE", defaultValue: false, description: "")
        booleanParam(name: "DCHECK_ALWAYS_ON", defaultValue: true, description: "")
        booleanParam(name: "DEBUG", defaultValue: false, description: "")
    }
    environment {
        GITHUB_CREDENTIAL_ID = "brave-builds-github-token-for-pr-builder"
        REFERRAL_API_KEY = credentials("REFERRAL_API_KEY")
        BRAVE_SERVICES_KEY = credentials("brave-services-key")
        BRAVE_INFURA_PROJECT_ID = credentials("brave-infura-project-id")
        BRAVE_GOOGLE_API_KEY = credentials("npm_config_brave_google_api_key")
        BRAVE_ARTIFACTS_S3_BUCKET = credentials("brave-jenkins-artifacts-s3-bucket")
        SLACK_USERNAME_MAP = credentials("github-to-slack-username-map")
        SIGN_WIDEVINE_PASSPHRASE = credentials("447b2fa7-c989-43af-9047-8ae158fad0a3")
        BINANCE_CLIENT_ID = credentials("binance-client-id")
    }
    stages {
        stage("env") {
            steps {
                withCredentials([usernamePassword(credentialsId: "${GITHUB_CREDENTIAL_ID}", usernameVariable: "PR_BUILDER_USER", passwordVariable: "PR_BUILDER_TOKEN")]) {
                    script {
                        setEnv()
                    }
                }
            }
        }
        stage("abort") {
            steps {
                script {
                    checkAndAbortBuild()
                }
            }
        }
        stage("s3-init") {
            agent { label "master" }
            steps {
                sh "echo ${BUILD_URL} > build.txt"
                s3Upload(bucket: BRAVE_ARTIFACTS_S3_BUCKET, path: BUILD_TAG_SLASHED, includePathPattern: "build.txt")
            }
        }
        stage("build-all") {
            when {
                beforeAgent true
                expression { !SKIP }
            }
            parallel {
                stage("android") {
                    when {
                        beforeAgent true
                        expression { !SKIP_ANDROID }
                    }
                    agent { label "android-ci" }
                    environment {
                        GIT_CACHE_PATH = "${HOME}/cache"
                        QA_CODE = credentials("android-browser-qa-code")
                    }
                    stages {
                        stage("checkout") {
                            steps {
                                checkout([$class: "GitSCM", branches: [[name: BRANCH]], extensions: [[$class: WIPE_WORKSPACE]], userRemoteConfigs: [[url: "https://github.com/brave/brave-browser.git"]]])
                            }
                        }
                        stage("pin") {
                            when {
                                expression { BRANCH_EXISTS_IN_BC }
                            }
                            steps {
                                pin()
                            }
                        }
                        stage("install") {
                            steps {
                                script {
                                    install()
                                }
                            }
                        }
                        stage("test-scripts") {
                            steps {
                                sh "npm run test:scripts -- --verbose"
                            }
                        }
                        stage("init") {
                            when {
                                expression { return !fileExists("src/brave/package.json") || !SKIP_INIT }
                            }
                            steps {
                                sh """
                                    rm -rf src/brave
                                    npm run init -- --target_os=android
                                """
                            }
                        }
                        stage("lint") {
                            steps {
                                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                    script {
                                        lint()
                                    }
                                }
                            }
                        }
                        stage("sccache") {
                            when {
                                expression { !DISABLE_SCCACHE }
                            }
                            steps {
                                script {
                                    sccache()
                                }
                            }
                        }
                        stage("build") {
                            steps {
                                script {
                                    config()
                                }
                                sh """
                                    npm config --userconfig=.npmrc set brave_android_developer_options_code ${QA_CODE}
                                    npm run build -- ${BUILD_TYPE} --channel=${CHANNEL} --target_os=android --target_arch=arm
                                """
                            }
                        }
                        stage("dist") {
                            steps {
                                sh "npm run create_dist -- ${BUILD_TYPE} --channel=${CHANNEL} --target_os=android --target_arch=arm"
                            }
                        }
                        stage("s3-upload") {
                            steps {
                                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                    sh """
                                        aws s3 cp --no-progress src/out/android_${BUILD_TYPE}_arm/apks/Bravearm.apk s3://${BRAVE_ARTIFACTS_S3_BUCKET}/${BUILD_TAG_SLASHED}/
                                        aws s3 cp --no-progress src/out/android_${BUILD_TYPE}_arm/dist/Defaultarmclassic s3://${BRAVE_ARTIFACTS_S3_BUCKET}/${BUILD_TAG_SLASHED}/
                                    """
                                }
                            }
                        }
                    }
                }
                stage("ios") {
                    when {
                        beforeAgent true
                        expression { !SKIP_IOS }
                    }
                    agent { label "mac-ci" }
                    environment {
                        GIT_CACHE_PATH = "${HOME}/cache"
                    }
                    stages {
                        stage("checkout") {
                            steps {
                                checkout([$class: "GitSCM", branches: [[name: BRANCH]], extensions: [[$class: WIPE_WORKSPACE]], userRemoteConfigs: [[url: "https://github.com/brave/brave-browser.git"]]])
                            }
                        }
                        stage("pin") {
                            when {
                                expression { BRANCH_EXISTS_IN_BC }
                            }
                            steps {
                                pin()
                            }
                        }
                        stage("install") {
                            steps {
                                script {
                                    install()
                                }
                            }
                        }
                        stage("test-scripts") {
                            steps {
                                sh "npm run test:scripts -- --verbose"
                            }
                        }
                        stage("init") {
                            when {
                                expression { return !fileExists("src/brave/package.json") || !SKIP_INIT }
                            }
                            steps {
                                sh """
                                    rm -rf src/brave
                                    npm run init -- --target_os=ios
                                """
                            }
                        }
                        stage("lint") {
                            steps {
                                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                    script {
                                        lint()
                                    }
                                }
                            }
                        }
                        stage("build") {
                            steps {
                                script {
                                    config()
                                    sh """
                                        npm run build -- ${BUILD_TYPE} --channel=${CHANNEL} --target_os=ios
                                        npm run build -- ${BUILD_TYPE} --channel=${CHANNEL} --target_os=ios --target_arch=arm64
                                    """
                                }
                            }
                        }
                        stage("test-unit") {
                            steps {
                                timeout(time: 2, unit: "MINUTES") {
                                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                        sh "npm run test -- brave_rewards_ios_tests ${BUILD_TYPE} --target_os=ios"
                                    }
                                }
                            }
                        }
                        stage("dist") {
                            steps {
                                sh """
                                    cd src/out
                                    cp -R ios_${BUILD_TYPE}_arm64/BraveRewards.framework .
                                    lipo -create -output BraveRewards.framework/BraveRewards ios_${BUILD_TYPE}/BraveRewards.framework/BraveRewards ios_${BUILD_TYPE}_arm64/BraveRewards.framework/BraveRewards
                                    zip -r BraveRewards.framework.zip BraveRewards.framework
                                """
                            }
                        }
                        stage("s3-upload") {
                            steps {
                                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                    withAWS(credentials: "mac-build-s3-upload-artifacts", region: "us-west-2") {
                                        sh "aws s3 cp --no-progress src/out/BraveRewards.framework.zip s3://${BRAVE_ARTIFACTS_S3_BUCKET}/${BUILD_TAG_SLASHED}/"
                                    }
                                }
                            }
                        }
                    }
                }
                stage("linux") {
                    when {
                        beforeAgent true
                        expression { !SKIP_LINUX }
                    }
                    agent { label "linux-ci" }
                    environment {
                        GIT_CACHE_PATH = "${HOME}/cache"
                    }
                    stages {
                        stage("checkout") {
                            steps {
                                checkout([$class: "GitSCM", branches: [[name: BRANCH]], extensions: [[$class: WIPE_WORKSPACE]], userRemoteConfigs: [[url: "https://github.com/brave/brave-browser.git"]]])
                            }
                        }
                        stage("pin") {
                            when {
                                expression { BRANCH_EXISTS_IN_BC }
                            }
                            steps {
                                pin()
                            }
                        }
                        stage("install") {
                            steps {
                                script {
                                    install()
                                }
                            }
                        }
                        stage("test-scripts") {
                            steps {
                                sh "npm run test:scripts -- --verbose"
                            }
                        }
                        stage("init") {
                            when {
                                expression { return !fileExists("src/brave/package.json") || !SKIP_INIT }
                            }
                            steps {
                                sh """
                                    rm -rf src/brave
                                    npm run init
                                """
                            }
                        }
                        stage("lint") {
                            steps {
                                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                    script {
                                        lint()
                                    }
                                }
                            }
                        }
                        stage("sccache") {
                            when {
                                expression { !DISABLE_SCCACHE }
                            }
                            steps {
                                script {
                                    sccache()
                                }
                            }
                        }
                        stage("build") {
                            steps {
                                script {
                                    config()
                                }
                                sh "npm run build -- ${BUILD_TYPE} --channel=${CHANNEL}"
                            }
                        }
                        stage("audit-network") {
                            when {
                                expression { RUN_NETWORK_AUDIT }
                            }
                            steps {
                                timeout(time: 4, unit: "MINUTES") {
                                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                        sh "npm run network-audit -- --output_path=\"${OUT_DIR}/brave\""
                                    }
                                }
                            }
                        }
                        stage("test-unit") {
                            steps {
                                timeout(time: 60, unit: "MINUTES") {
                                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                        script {
                                            sh "npm run test -- brave_unit_tests ${BUILD_TYPE} --output brave_unit_tests.xml"
                                            xunit([GoogleTest(pattern: "src/brave_unit_tests.xml", deleteOutputFiles: false, failIfNotNew: true, skipNoTestFiles: false, stopProcessingIfError: false)])
                                            xunit([GoogleTest(pattern: "src/brave_installer_unittests.xml", deleteOutputFiles: false, failIfNotNew: true, skipNoTestFiles: false, stopProcessingIfError: false)])
                                        }
                                    }
                                }
                            }
                        }
                        stage("test-browser") {
                            steps {
                                timeout(time: 20, unit: "MINUTES") {
                                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                        script {
                                            sh "npm run test -- brave_browser_tests ${BUILD_TYPE} --output brave_browser_tests.xml"
                                            xunit([GoogleTest(pattern: "src/brave_browser_tests.xml", deleteOutputFiles: false, failIfNotNew: true, skipNoTestFiles: false, stopProcessingIfError: false)])
                                        }
                                    }
                                }
                            }
                        }
                        stage("dist") {
                            steps {
                                sh "npm run create_dist -- ${BUILD_TYPE} --channel=${CHANNEL}"
                            }
                        }
                        stage("s3-upload") {
                            steps {
                                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                    sh """
                                        cd ${OUT_DIR}
                                        aws s3 cp --no-progress . s3://${BRAVE_ARTIFACTS_S3_BUCKET}/${BUILD_TAG_SLASHED} --recursive --exclude="*" --include "brave-*.deb"
                                        aws s3 cp --no-progress . s3://${BRAVE_ARTIFACTS_S3_BUCKET}/${BUILD_TAG_SLASHED} --recursive --exclude="*" --include "brave-*.rpm"
                                        aws s3 cp --no-progress dist s3://${BRAVE_ARTIFACTS_S3_BUCKET}/${BUILD_TAG_SLASHED} --recursive --exclude="*" --include "brave-*.zip"
                                    """
                                }
                            }
                        }
                    }
                }
                stage("macos") {
                    when {
                        beforeAgent true
                        expression { !SKIP_MACOS }
                    }
                    agent { label "mac-ci" }
                    environment {
                        GIT_CACHE_PATH = "${HOME}/cache"
                        KEYCHAIN = "signing-ci"
                        KEYCHAIN_PATH = "/Users/jenkins/Library/Keychains/${KEYCHAIN}.keychain-db"
                        KEYCHAIN_PASS = credentials("mac-ci-signing-keychain-password")
                        MAC_APPLICATION_SIGNING_IDENTIFIER = credentials("mac-ci-signing-application-id")
                        MAC_INSTALLER_SIGNING_IDENTIFIER = credentials("mac-ci-signing-installer-id")
                    }
                    stages {
                        stage("checkout") {
                            steps {
                                checkout([$class: "GitSCM", branches: [[name: BRANCH]], extensions: [[$class: WIPE_WORKSPACE]], userRemoteConfigs: [[url: "https://github.com/brave/brave-browser.git"]]])
                            }
                        }
                        stage("pin") {
                            when {
                                expression { BRANCH_EXISTS_IN_BC }
                            }
                            steps {
                                pin()
                            }
                        }
                        stage("install") {
                            steps {
                                script {
                                    install()
                                    sh """
                                        mkdir -p src/third_party/widevine/scripts
                                        cp ${HOME}/signature_generator.py src/third_party/widevine/scripts
                                    """
                                }
                            }
                        }
                        stage("test-scripts") {
                            steps {
                                sh "npm run test:scripts -- --verbose"
                            }
                        }
                        stage("init") {
                            when {
                                expression { return !fileExists("src/brave/package.json") || !SKIP_INIT }
                            }
                            steps {
                                sh """
                                    rm -rf src/brave
                                    npm run init
                                """
                            }
                        }
                        stage("lint") {
                            steps {
                                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                    script {
                                        lint()
                                    }
                                }
                            }
                        }
                        stage("sccache") {
                            when {
                                expression { !DISABLE_SCCACHE }
                            }
                            steps {
                                script {
                                    sccache()
                                }
                            }
                        }
                        stage("build") {
                            environment {
                                SIGN_WIDEVINE_CERT = credentials("widevine_brave_prod_cert.der")
                                SIGN_WIDEVINE_KEY = credentials("widevine_brave_prod_key.pem")
                            }
                            steps {
                                script {
                                    config()
                                }
                                sh "npm run build -- ${BUILD_TYPE} --channel=${CHANNEL} ${SKIP_SIGNING}"
                            }
                        }
                        stage("audit-network") {
                            when {
                                expression { RUN_NETWORK_AUDIT }
                            }
                            steps {
                                timeout(time: 4, unit: "MINUTES") {
                                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                        sh "npm run network-audit -- --output_path=\"${OUT_DIR}/Brave\\ Browser${CHANNEL_CAPITALIZED_BACKSLASHED_SPACED}.app/Contents/MacOS/Brave\\ Browser${CHANNEL_CAPITALIZED_BACKSLASHED_SPACED}\""
                                    }
                                }
                            }
                        }
                        stage("test-unit") {
                            steps {
                                timeout(time: 60, unit: "MINUTES") {
                                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                        script {
                                            sh "npm run test -- brave_unit_tests ${BUILD_TYPE} --output brave_unit_tests.xml"
                                            xunit([GoogleTest(pattern: "src/brave_unit_tests.xml", deleteOutputFiles: false, failIfNotNew: true, skipNoTestFiles: false, stopProcessingIfError: false)])
                                            xunit([GoogleTest(pattern: "src/brave_installer_unittests.xml", deleteOutputFiles: false, failIfNotNew: true, skipNoTestFiles: false, stopProcessingIfError: false)])
                                        }
                                    }
                                }
                            }
                        }
                        stage("test-browser") {
                            steps {
                                timeout(time: 20, unit: "MINUTES") {
                                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                        script {
                                            sh "npm run test -- brave_browser_tests ${BUILD_TYPE} --output brave_browser_tests.xml"
                                            xunit([GoogleTest(pattern: "src/brave_browser_tests.xml", deleteOutputFiles: false, failIfNotNew: true, skipNoTestFiles: false, stopProcessingIfError: false)])
                                        }
                                    }
                                }
                            }
                        }
                        stage("dist") {
                            environment {
                                SIGN_WIDEVINE_CERT = credentials("widevine_brave_prod_cert.der")
                                SIGN_WIDEVINE_KEY = credentials("widevine_brave_prod_key.pem")
                            }
                            steps {
                                sh """
                                    security unlock-keychain -p "${KEYCHAIN_PASS}" "${KEYCHAIN_PATH}"
                                    npm run create_dist -- ${BUILD_TYPE} --channel=${CHANNEL} ${SKIP_SIGNING} --mac_signing_keychain=${KEYCHAIN} --mac_signing_identifier=${MAC_APPLICATION_SIGNING_IDENTIFIER} --mac_installer_signing_identifier=${MAC_INSTALLER_SIGNING_IDENTIFIER}
                                    security lock-keychain -a
                                """
                            }
                        }
                        stage("test-install") {
                            steps {
                                timeout(time: 5, unit: "MINUTES") {
                                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                        testInstallMac()
                                    }
                                }
                            }
                        }
                        stage("s3-upload") {
                            steps {
                                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                    withAWS(credentials: "mac-build-s3-upload-artifacts", region: "us-west-2") {
                                        sh """
                                            cd ${OUT_DIR}
                                            aws s3 cp --no-progress . s3://${BRAVE_ARTIFACTS_S3_BUCKET}/${BUILD_TAG_SLASHED} --recursive --exclude="*" --include "unsigned_dmg/Brave*.dmg"
                                            aws s3 cp --no-progress . s3://${BRAVE_ARTIFACTS_S3_BUCKET}/${BUILD_TAG_SLASHED} --recursive --exclude="*" --include "Brave*.dmg"
                                            aws s3 cp --no-progress . s3://${BRAVE_ARTIFACTS_S3_BUCKET}/${BUILD_TAG_SLASHED} --recursive --exclude="*" --include "Brave*.pkg"
                                            aws s3 cp --no-progress dist s3://${BRAVE_ARTIFACTS_S3_BUCKET}/${BUILD_TAG_SLASHED} --recursive --exclude="*" --include "brave-*.zip"
                                        """
                                    }
                                }
                            }
                        }
                    }
                }
                stage("windows-x64") {
                    when {
                        beforeAgent true
                        expression { !SKIP_WINDOWS }
                    }
                    agent {
                        node {
                            label "windows-ci"
                            customWorkspace "C:\\" + BRANCH[-4..-1]
                        }
                    }
                    environment {
                        GIT_CACHE_PATH = "C:\\Users\\Administrator\\cache"
                        PATH = "C:\\Program Files (x86)\\Windows Kits\\10\\bin\\10.0.18362.0\\x64\\;C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Community\\Common7\\IDE\\Remote Debugger\\x64;${PATH}"
                        SIGNTOOL_ARGS = "sign /t http://timestamp.digicert.com /fd sha256 /sm"
                        CERT = "Brave"
                        KEY_CER_PATH = "C:\\jenkins\\digicert.cer"
                        KEY_PFX_PATH = "C:\\jenkins\\digicert.pfx"
                        AUTHENTICODE_PASSWORD = credentials("digicert-brave-browser-ci-certificate-ps-escaped")
                        AUTHENTICODE_PASSWORD_UNESCAPED = credentials("digicert-brave-browser-ci-certificate")
                    }
                    stages {
                        stage("checkout") {
                            steps {
                                checkout([$class: "GitSCM", branches: [[name: BRANCH]], extensions: [[$class: WIPE_WORKSPACE]], userRemoteConfigs: [[url: "https://github.com/brave/brave-browser.git"]]])
                            }
                        }
                        stage("pin") {
                            when {
                                expression { BRANCH_EXISTS_IN_BC }
                            }
                            steps {
                                echo "Pinning brave-core locally to use branch ${BRANCH}"
                                powershell """
                                    \$ErrorActionPreference = "Stop"
                                    \$PSDefaultParameterValues['Out-File:Encoding'] = "utf8"
                                    jq "del(.config.projects[\\`"brave-core\\`"].branch) | .config.projects[\\`"brave-core\\`"].branch=\\`"${BRANCH}\\`"" package.json > package.json.new
                                    Move-Item -Force package.json.new package.json
                                """
                            }
                        }
                        stage("install") {
                            environment {
                                SOURCE_KEY_CER_PATH = credentials("digicert-brave-browser-ci-certificate-cer")
                                SOURCE_KEY_PFX_PATH = credentials("digicert-brave-browser-ci-certificate-pfx")
                                SIGN_WIDEVINE_CERT = credentials("widevine_brave_prod_cert.der")
                            }
                            steps {
                                script {
                                    installWindows()
                                }
                            }
                        }
                        stage("test-scripts") {
                            steps {
                                powershell """
                                    \$ErrorActionPreference = "Stop"
                                    npm run test:scripts -- --verbose
                                """
                            }
                        }
                        stage("init") {
                            when {
                                expression { return !fileExists("src/brave/package.json") || !SKIP_INIT }
                            }
                            steps {
                                powershell """
                                    Remove-Item -Recurse -Force src/brave
                                    git gc
                                    git -C vendor/depot_tools clean -fxd
                                    \$ErrorActionPreference = "Stop"
                                    npm run init
                                """
                            }
                        }
                        stage("lint") {
                            steps {
                                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                    script {
                                        lintWindows()
                                    }
                                }
                            }
                        }
                        stage("build") {
                            environment {
                                SIGN_WIDEVINE_CERT = credentials("widevine_brave_prod_cert.der")
                                SIGN_WIDEVINE_KEY = credentials("widevine_brave_prod_key.pem")
                            }
                            steps {
                                script {
                                    configWindows()
                                }
                                powershell """
                                    \$ErrorActionPreference = "Stop"
                                    npm run build -- ${BUILD_TYPE} --channel=${CHANNEL} ${SKIP_SIGNING}
                                """
                            }
                        }
                        stage("audit-network") {
                            when {
                                expression { RUN_NETWORK_AUDIT }
                            }
                            steps {
                                timeout(time: 4, unit: "MINUTES") {
                                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                        powershell """
                                            \$ErrorActionPreference = "Stop"
                                            npm run network-audit -- --output_path="${OUT_DIR}/brave.exe"
                                        """
                                    }
                                }
                            }
                        }
                        stage("test-unit") {
                            steps {
                                timeout(time: 60, unit: "MINUTES") {
                                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                        powershell """
                                            \$ErrorActionPreference = "Stop"
                                            npm run test -- brave_unit_tests ${BUILD_TYPE} --output brave_unit_tests.xml
                                        """
                                        xunit([GoogleTest(pattern: "src/brave_unit_tests.xml", deleteOutputFiles: false, failIfNotNew: true, skipNoTestFiles: false, stopProcessingIfError: false)])
                                        xunit([GoogleTest(pattern: "src/brave_installer_unittests.xml", deleteOutputFiles: false, failIfNotNew: true, skipNoTestFiles: false, stopProcessingIfError: false)])
                                    }
                                }
                            }
                        }
                        stage("dist") {
                            environment {
                                SIGN_WIDEVINE_CERT = credentials("widevine_brave_prod_cert.der")
                                SIGN_WIDEVINE_KEY = credentials("widevine_brave_prod_key.pem")
                            }
                            steps {
                                powershell """
                                    \$ErrorActionPreference = "Stop"
                                    (Get-Content src/brave/vendor/omaha/omaha/hammer-brave.bat) | % { \$_ -replace "10.0.15063.0\\\\", "" } | Set-Content src/brave/vendor/omaha/omaha/hammer-brave.bat
                                    npm run create_dist -- ${BUILD_TYPE} --channel=${CHANNEL} ${SKIP_SIGNING} --build_omaha --tag_ap=x64-${CHANNEL}
                                """
                            }
                        }
                        stage("test-install") {
                            steps {
                                timeout(time: 5, unit: "MINUTES") {
                                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                        script {
                                            testInstallWindows()
                                        }
                                    }
                                }
                            }
                        }
                        stage("s3-upload") {
                            steps {
                                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                    powershell """
                                        \$ErrorActionPreference = "Stop"
                                        Set-Location -Path ${OUT_DIR}
                                        aws s3 cp --no-progress . s3://${BRAVE_ARTIFACTS_S3_BUCKET}/${BUILD_TAG_SLASHED} --recursive --exclude="*" --include ""brave_installer*.exe""
                                        aws s3 cp --no-progress . s3://${BRAVE_ARTIFACTS_S3_BUCKET}/${BUILD_TAG_SLASHED} --recursive --exclude="*" --include "BraveBrowser*Setup*.exe"
                                        aws s3 cp --no-progress dist s3://${BRAVE_ARTIFACTS_S3_BUCKET}/${BUILD_TAG_SLASHED} --recursive --exclude="*" --include "*.zip"
                                    """
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                if (env.SLACK_USERNAME) {
                    def slackColorMap = ["SUCCESS": "good", "FAILURE": "danger", "UNSTABLE": "warning", "ABORTED": null]
                    slackSend(color: slackColorMap[currentBuild.currentResult], channel: env.SLACK_USERNAME, message: "[${BUILD_TAG_SLASHED} `${BRANCH}`] " + currentBuild.currentResult + " (<${BUILD_URL}/flowGraphTable/?auto_refresh=true|Open>)")
                }
            }
            node("master") {
                script {
                    if (SLACK_BUILDS_CHANNEL) {
                        sendSlackDownloadsNotification()
                    }
                }
            }
        }
    }
}

def setEnv() {
    BUILD_TYPE = params.BUILD_TYPE
    CHANNEL = params.CHANNEL
    SLACK_BUILDS_CHANNEL = params.SLACK_BUILDS_CHANNEL
    DCHECK_ALWAYS_ON = params.DCHECK_ALWAYS_ON
    CHANNEL_CAPITALIZED = CHANNEL.equals("release") ? "" : CHANNEL.capitalize()
    CHANNEL_CAPITALIZED_BACKSLASHED_SPACED = CHANNEL.equals("release") ? "" : "\\ " + CHANNEL.capitalize()
    SKIP_SIGNING = params.SKIP_SIGNING ? "--skip_signing" : ""
    WIPE_WORKSPACE = params.WIPE_WORKSPACE ? "WipeWorkspace" : "RelativeTargetDirectory"
    SKIP_INIT = params.SKIP_INIT
    DISABLE_SCCACHE = params.DISABLE_SCCACHE
    DEBUG = params.DEBUG
    OUT_DIR = "src/out/" + BUILD_TYPE
    BUILD_TAG_SLASHED = env.JOB_NAME + "/" + env.BUILD_NUMBER
    LINT_BRANCH = "TEMP_LINT_BRANCH_" + env.BUILD_NUMBER
    BRAVE_GITHUB_TOKEN = "brave-browser-releases-github"
    GITHUB_API = "https://api.github.com/repos/brave"
    SKIP = false
    SKIP_ANDROID = false
    SKIP_IOS = false
    SKIP_LINUX = false
    SKIP_MACOS = false
    SKIP_WINDOWS = false
    RUN_NETWORK_AUDIT = false
    BRANCH = env.BRANCH_NAME
    BASE_BRANCH = "master"
    if (env.CHANGE_BRANCH) {
        BRANCH = env.CHANGE_BRANCH
        BASE_BRANCH = env.CHANGE_TARGET
        def bbPrNumber = readJSON(text: httpRequest(url: GITHUB_API + "/brave-browser/pulls?head=brave:" + BRANCH, customHeaders: [[name: "Authorization", value: "token ${PR_BUILDER_TOKEN}"]], quiet: !DEBUG).content)[0].number
        def bbPrDetails = readJSON(text: httpRequest(url: GITHUB_API + "/brave-browser/pulls/" + bbPrNumber, customHeaders: [[name: "Authorization", value: "token ${PR_BUILDER_TOKEN}"]], quiet: !DEBUG).content)
        SKIP = bbPrDetails.mergeable_state.equals("draft") || bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip") }.equals(1)
        SKIP_ANDROID = bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-android") }.equals(1)
        SKIP_IOS = bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-ios") }.equals(1)
        SKIP_LINUX = bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-linux") }.equals(1)
        SKIP_MACOS = bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-macos") }.equals(1)
        SKIP_WINDOWS = bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-windows") }.equals(1)
        RUN_NETWORK_AUDIT = bbPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/run-network-audit") }.equals(1)
        env.SLACK_USERNAME = readJSON(text: SLACK_USERNAME_MAP)[bbPrDetails.user.login]
        env.BRANCH_PRODUCTIVITY_HOMEPAGE = "https://github.com/brave/brave-browser/pull/${bbPrNumber}"
        env.BRANCH_PRODUCTIVITY_NAME = "Brave Browser PR #${bbPrNumber}"
        env.BRANCH_PRODUCTIVITY_DESCRIPTION = bbPrDetails.title
        env.BRANCH_PRODUCTIVITY_USER = env.SLACK_USERNAME ?: bbPrDetails.user.login
    }
    BRANCH_EXISTS_IN_BC = httpRequest(url: GITHUB_API + "/brave-core/branches/" + BRANCH, validResponseCodes: "100:499", customHeaders: [[name: "Authorization", value: "token ${PR_BUILDER_TOKEN}"]], quiet: !DEBUG).status.equals(200)
    if (BRANCH_EXISTS_IN_BC) {
        def bcPrDetails = readJSON(text: httpRequest(url: GITHUB_API + "/brave-core/pulls?head=brave:" + BRANCH, customHeaders: [[name: "Authorization", value: "token ${PR_BUILDER_TOKEN}"]], quiet: !DEBUG).content)[0]
        if (bcPrDetails) {
            env.BC_PR_NUMBER = bcPrDetails.number
            bcPrDetails = readJSON(text: httpRequest(url: GITHUB_API + "/brave-core/pulls/" +  env.BC_PR_NUMBER, customHeaders: [[name: "Authorization", value: "token ${PR_BUILDER_TOKEN}"]], quiet: !DEBUG).content)
            BASE_BRANCH = bcPrDetails.base.ref
            SKIP = bcPrDetails.mergeable_state.equals("draft") || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip") }.equals(1)
            SKIP_ANDROID = SKIP_ANDROID || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-android") }.equals(1)
            SKIP_IOS = SKIP_IOS || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-ios") }.equals(1)
            SKIP_LINUX = SKIP_LINUX || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-linux") }.equals(1)
            SKIP_MACOS = SKIP_MACOS || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-macos") }.equals(1)
            SKIP_WINDOWS = SKIP_WINDOWS || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/skip-windows") }.equals(1)
            RUN_NETWORK_AUDIT = RUN_NETWORK_AUDIT || bcPrDetails.labels.count { label -> label.name.equalsIgnoreCase("CI/run-network-audit") }.equals(1)
            env.SLACK_USERNAME = readJSON(text: SLACK_USERNAME_MAP)[bcPrDetails.user.login]
            env.BRANCH_PRODUCTIVITY_HOMEPAGE = "https://github.com/brave/brave-core/pull/${bcPrDetails.number}"
            env.BRANCH_PRODUCTIVITY_NAME = "Brave Core PR #${bcPrDetails.number}"
            env.BRANCH_PRODUCTIVITY_DESCRIPTION = bcPrDetails.title
            env.BRANCH_PRODUCTIVITY_USER = bcPrDetails.user.login
        }
    }
    if (env.SLACK_USERNAME) {
        slackSend(color: null, channel: env.SLACK_USERNAME, message: "[${BUILD_TAG_SLASHED} `${BRANCH}`] STARTED (<${BUILD_URL}/flowGraphTable/?auto_refresh=true|Open>)")
    }
}

def checkAndAbortBuild() {
    if (SKIP) {
        echo "Aborting build as PR is in draft or has \"CI/skip\" label"
        stopCurrentBuild()
    }
    else if (BRANCH_EXISTS_IN_BC) {
        if (isStartedManually()) {
            if (env.BC_PR_NUMBER) {
                echo "Aborting build as PR exists in brave-core and build has not been started from there"
                echo "Use " + env.JENKINS_URL + "view/ci/job/brave-core-build-pr/view/change-requests/job/PR-" + env.BC_PR_NUMBER + " to trigger"
            }
            else {
                echo "Aborting build as there's a matching branch in brave-core, please create a PR there first"
                echo "Use https://github.com/brave/brave-core/compare/" + BASE_BRANCH + "..." + BRANCH + " to create PR"
            }
            SKIP = true
            stopCurrentBuild()
        }
    }
    if (!SKIP) {
        for (build in getBuilds()) {
            if (build.isBuilding() && build.getNumber() < env.BUILD_NUMBER.toInteger()) {
                echo "Aborting older running build " + build
                build.doStop()
                // build.finish(hudson.model.Result.ABORTED, new java.io.IOException("Aborting build"))
            }
        }
        sleep(time: 1, unit: "MINUTES")
    }
}

def sendSlackDownloadsNotification() {
    // Notify links to all the build files
    echo "Reading all uploaded files for slack notification..."
    def attachments = getSlackFileAttachments()
    if (!attachments.isEmpty()) {
        def messageText = getSlackMessageText()
        echo "Sending builds message: '${messageText}'."
        slackSend(
            channel: SLACK_BUILDS_CHANNEL,
            message: messageText,
            attachments: attachments
        )
    } else {
        echo "Not sending message, no files found."
    }
}

def getSlackFileAttachments () {
    def files = s3FindFiles(bucket: BRAVE_ARTIFACTS_S3_BUCKET, path: BUILD_TAG_SLASHED, glob: "**")
    def attachments = [ ]
    files.each { file ->
        echo "Found file: ${file.name}"
        if (!file.directory && file.name != "build.txt") {
            attachments.add([
                title: file.name,
                title_link: "https://" + BRAVE_ARTIFACTS_S3_BUCKET + ".s3.amazonaws.com/" + BUILD_TAG_SLASHED + "/" + file.path,
                footer: byteLengthToString(file.length)
            ])
        }
    }
    return attachments
}

def getSlackMessageText () {
    def messageText = "Downloads are available for branch `${BRANCH}`"
    if (env.BRANCH_PRODUCTIVITY_NAME) {
        messageText += "\n(<${env.BRANCH_PRODUCTIVITY_HOMEPAGE}|${env.BRANCH_PRODUCTIVITY_NAME}>)"
    }
    if (env.SLACK_USERNAME) {
        messageText += " by <${env.SLACK_USERNAME}>"
    } else if (env.BRANCH_PRODUCTIVITY_USER) {
        messageText += " by ${env.BRANCH_PRODUCTIVITY_USER}"
    }
    if (env.BRANCH_PRODUCTIVITY_DESCRIPTION) {
        messageText += "\n_${env.BRANCH_PRODUCTIVITY_DESCRIPTION}_"
    }
    return messageText
}

def byteLengthToString (long byteLength) {
    def base = 1048576L
    def mbLength = (byteLength / base) as Double
    return mbLength.round() + " Mb"
}

@NonCPS
def stopCurrentBuild() {
    Jenkins.instance.getItemByFullName(env.JOB_NAME).getLastBuild().doStop()
}

@NonCPS
def isStartedManually() {
    return Jenkins.instance.getItemByFullName(env.JOB_NAME).getLastBuild().getCause(hudson.model.Cause$UpstreamCause) == null
}

@NonCPS
def getBuilds() {
    return Jenkins.instance.getItemByFullName(env.JOB_NAME).builds
}

def pin() {
    echo "Pinning brave-core locally to use branch ${BRANCH}"
    sh """
        jq 'del(.config.projects["brave-core"].branch) | .config.projects["brave-core"].branch="${BRANCH}"' package.json > package.json.new
        mv package.json.new package.json
    """
}

def install() {
    sh """
        rm -rf ${GIT_CACHE_PATH}/*.lock
        npm install --no-optional
    """
}

def lint() {
    sh """
        git -C src/brave config user.name jenkins
        git -C src/brave config user.email no@reply.com
        git -C src/brave checkout -b ${LINT_BRANCH}
        npm run lint -- --base=origin/${BASE_BRANCH}
        git -C src/brave checkout -q -
        git -C src/brave branch -D ${LINT_BRANCH}
    """
}

def lintWindows() {
    powershell """
        \$ErrorActionPreference = "Stop"
        git -C src/brave config user.name jenkins
        git -C src/brave config user.email no@reply.com
        git -C src/brave checkout -b ${LINT_BRANCH}
        npm run lint -- --base=origin/${BASE_BRANCH}
        git -C src/brave checkout -q -
        git -C src/brave branch -D ${LINT_BRANCH}
    """
}

def sccache() {
    echo "Enabling sccache"
    sh "npm config --userconfig=.npmrc set sccache sccache"
}

def config() {
    sh """
        npm config --userconfig=.npmrc set brave_referrals_api_key ${REFERRAL_API_KEY}
        npm config --userconfig=.npmrc set brave_services_key ${BRAVE_SERVICES_KEY}
        npm config --userconfig=.npmrc set brave_infura_project_id ${BRAVE_INFURA_PROJECT_ID}
        npm config --userconfig=.npmrc set brave_google_api_endpoint https://location.brave.com/v1/geolocate?key=
        npm config --userconfig=.npmrc set brave_google_api_key ${BRAVE_GOOGLE_API_KEY}
        npm config --userconfig=.npmrc set google_api_endpoint safebrowsing.brave.com
        npm config --userconfig=.npmrc set google_api_key dummytoken
        npm config --userconfig=.npmrc set dcheck_always_on ${DCHECK_ALWAYS_ON}
        npm config --userconfig=.npmrc set binance_client_id ${BINANCE_CLIENT_ID}
    """
}

def configWindows() {
    powershell """
        \$ErrorActionPreference = "Stop"
        npm config --userconfig=.npmrc set brave_referrals_api_key ${REFERRAL_API_KEY}
        npm config --userconfig=.npmrc set brave_services_key ${BRAVE_SERVICES_KEY}
        npm config --userconfig=.npmrc set brave_infura_project_id ${BRAVE_INFURA_PROJECT_ID}
        npm config --userconfig=.npmrc set brave_google_api_endpoint https://location.brave.com/v1/geolocate?key=
        npm config --userconfig=.npmrc set brave_google_api_key ${BRAVE_GOOGLE_API_KEY}
        npm config --userconfig=.npmrc set google_api_endpoint safebrowsing.brave.com
        npm config --userconfig=.npmrc set google_api_key dummytoken
        npm config --userconfig=.npmrc set dcheck_always_on ${DCHECK_ALWAYS_ON}
        npm config --userconfig=.npmrc set binance_client_id ${BINANCE_CLIENT_ID}
    """
}

def installWindows() {
    powershell """
        Remove-Item -Recurse -Force ${GIT_CACHE_PATH}/*.lock
        Get-ChildItem "Cert:\\LocalMachine\\My" | Remove-Item
        \$ErrorActionPreference = "Stop"
        npm install --no-optional
        Copy-Item "${SOURCE_KEY_CER_PATH}" -Destination "${KEY_CER_PATH}"
        Copy-Item "${SOURCE_KEY_PFX_PATH}" -Destination "${KEY_PFX_PATH}"
        Import-Certificate -FilePath "${SIGN_WIDEVINE_CERT}" -CertStoreLocation "Cert:\\LocalMachine\\My"
        Import-PfxCertificate -FilePath "${KEY_PFX_PATH}" -CertStoreLocation "Cert:\\LocalMachine\\My" -Password (ConvertTo-SecureString -String "${AUTHENTICODE_PASSWORD_UNESCAPED}" -AsPlaintext -Force)
        New-Item -Force -ItemType directory -Path "src\\third_party\\widevine\\scripts"
        Copy-Item "C:\\jenkins\\signature_generator.py" -Destination "src\\third_party\\widevine\\scripts\\"
    """
}

def testInstallWindows() {
    powershell """
        \$ErrorActionPreference = "Stop"
        Stop-Process -Name "Brave*" -Force
        Start-Process "${OUT_DIR}/brave_installer.exe"
        # sleep 60s
        Start-Sleep -Second 60
        # open Brave Browser
        Start-Process "C:\\Users\\Administrator\\AppData\\Local\\BraveSoftware\\Brave-Browser\\Application\\brave.exe"
        # make sure there are brave process
        \$Service = Get-Process | Where {\$_.ProcessName -eq "brave"}
        If (\$Service) { "Brave Browser was installed and running"  }
        Else {
            "Brave Browser was not running"
            exit 1
        }
        # Stop brave
        Stop-Process -Name "Brave*" -Force
        Remove-Item -Recurse -Force "C:\\Users\\Administrator\\Desktop\\Brave*"
    """
}

def testInstallMac() {
    sh '''
        # eject all existing volume first for brave
        VOLUME=$(diskutil list | grep "Brave Browser" | awk -F'MB   ' '{ print $2 }')
        declare -a arr=($VOLUME)
        # loop through the above array to eject all volumes
        for i in "${arr[@]}"
        do
            diskutil unmountDisk force $i
            diskutil eject $i
        done
        if [ -z ${CHANNEL} ]; then
            BROWSER="Brave Browser Nightly"
            BUILD_TYPE="Release"
            SKIP_SIGNING=true
        else
            if [ ${CHANNEL} = "release" ]; then CHANNEL_CAPITALIZED_SPACED=""; else CHANNEL_CAPITALIZED="$(tr '[:lower:]' '[:upper:]' <<< ${CHANNEL:0:1})${CHANNEL:1}"; fi
            if [ ${CHANNEL} = "release" ]; then BROWSER="Brave Browser"; else BROWSER="Brave Browser ${CHANNEL_CAPITALIZED}"; fi
        fi
        OUT_DIR="${WORKSPACE}/src/out/${BUILD_TYPE}"
        if [ ${SKIP_SIGNING} = true ] ; then
            hdiutil attach -nobrowse "${OUT_DIR}/unsigned_dmg/${BROWSER}.dmg"
        else
            hdiutil attach -nobrowse "${OUT_DIR}/${BROWSER}.dmg"
        fi
        sleep 10
        open "/Volumes/${BROWSER}/${BROWSER}.app"
        sleep 10
        pkill Brave
        VOLUME=$(diskutil list | grep "Brave Browser" | awk -F'MB   ' '{ print $2 }')
        declare -a arr=($VOLUME)
        # loop through the above array to eject all volumes
        for i in "${arr[@]}"
        do
            diskutil unmountDisk force $i
            diskutil eject $i
        done
    '''
}
