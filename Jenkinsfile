// Jenkinsfile - luci-app-adguardhome 构建

properties([
    buildDiscarder(logRotator(numToKeepStr: '10')),
    disableConcurrentBuilds()
])

def PROJECT_NAME = 'luci-app-adguardhome'
def GIT_URL = 'https://github.com/xiaoxiao29/luci-app-adguardhome.git'
def GIT_CREDENTIALS = 'github-ssh'
def OPENWRT_VERSION = '22.03.5'
def TARGET_ARCH = 'x86_64'
def SDK_URL = "https://downloads.openwrt.org/releases/${OPENWRT_VERSION}/targets/x86/64/openwrt-sdk-${OPENWRT_VERSION}-${TARGET_ARCH}_gcc-11.2.0_musl.Linux-x86_64.tar.xz"
def SDK_DIR = "openwrt-sdk-${OPENWRT_VERSION}-${TARGET_ARCH}"
def ARTIFACTS_DIR = "artifacts"

pipeline {
    agent any

    options {
        timeout(time: 2, unit: 'HOURS')
    }

    stages {
        stage('初始化') {
            steps {
                script {
                    def gitCommit = sh(script: 'git rev-parse HEAD', returnStdout: true).trim().substring(0, 8)
                    echo "========================================"
                    echo "Project: ${PROJECT_NAME}"
                    echo "Version: ${OPENWRT_VERSION}"
                    echo "Arch: ${TARGET_ARCH}"
                    echo "Build: ${BUILD_NUMBER}"
                    echo "Git: ${gitCommit}"
                    echo "========================================"
                }

                cleanWs(
                    cleanWhenAborted: true,
                    cleanWhenFailure: true,
                    cleanWhenNotBuilt: true,
                    cleanWhenUnstable: true,
                    deleteDirs: true
                )

                dir('source') {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: '*/master']],
                        userRemoteConfigs: [[
                            url: GIT_URL,
                            credentialsId: GIT_CREDENTIALS
                        ]]
                    ])
                }
            }
        }

        stage('安装依赖') {
            steps {
                sh '''#!/bin/bash
set -e
echo "Install dependencies..."
sudo apt-get update -qq
sudo apt-get install -y -qq build-essential cmake g++ git wget tar xz-utils subversion unzip zip python3 time
echo "Done"'''
            }
        }

        stage('准备 SDK') {
            steps {
                sh '''#!/bin/bash
set -e
echo "SDK URL: ${SDK_URL}"
if [ ! -d "${SDK_DIR}" ]; then
    echo "Downloading SDK..."
    wget -q "${SDK_URL}" -O sdk.tar.xz
    tar -xf sdk.tar.xz
    rm -f sdk.tar.xz
    echo "SDK ready"
else
    echo "Using cached SDK"
fi
ls -la "${SDK_DIR}/"'''
            }
        }

        stage('配置 SDK') {
            steps {
                sh '''#!/bin/bash
set -e
cd "${SDK_DIR}"
echo "Update feeds..."
./scripts/feeds update -a || true
./scripts/feeds install -a || true
echo "Copy project..."
cp -r ../source/luci-app-adguardhome package/
rm -rf package/luci-app-adguardhome/.git
echo "Configure..."
echo "CONFIG_PACKAGE_luci-app-adguardhome=y" > .config'''
            }
        }

        stage('编译') {
            steps {
                script {
                    timeout(time: 90, unit: 'MINUTES') {
                        sh '''#!/bin/bash
set -e
cd "${SDK_DIR}"
echo "Start build..."
START_TIME=$(date +%s)
make defconfig 2>&1 | tail -10
make package/luci-app-adguardhome/compile -j$(nproc) V=s 2>&1 | tee ../build.log
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))
echo "Build completed in ${DURATION} seconds"
find bin/packages -name "luci-app-adguardhome*.ipk" -exec ls -lh {} \\;'''
                    }
                }
            }
        }

        stage('后处理') {
            steps {
                sh '''#!/bin/bash
set -e
cd "${SDK_DIR}"
echo "Collect artifacts..."
mkdir -p "${WORKSPACE}/${ARTIFACTS_DIR}/${OPENWRT_VERSION}/${TARGET_ARCH}"
find bin/packages -name "luci-app-adguardhome*.ipk" -exec cp -v {} "${WORKSPACE}/${ARTIFACTS_DIR}/${OPENWRT_VERSION}/${TARGET_ARCH}/" \\;
cp ../build.log "${WORKSPACE}/${ARTIFACTS_DIR}/${OPENWRT_VERSION}/${TARGET_ARCH}/" 2>/dev/null || true
cd "${WORKSPACE}/${ARTIFACTS_DIR}/${OPENWRT_VERSION}/${TARGET_ARCH}"
ls -lh
sha256sum * > SHA256SUMS.txt'''
            }
        }

        stage('归档') {
            steps {
                archiveArtifacts(
                    artifacts: "${ARTIFACTS_DIR}/**",
                    fingerprint: true,
                    allowEmptyArchive: true
                )
            }
        }
    }

    post {
        success {
            echo "========================================"
            echo "BUILD SUCCESS!"
            echo "========================================"
        }

        failure {
            echo "========================================"
            echo "BUILD FAILED!"
            echo "========================================"
        }

        always {
            sh 'echo "Log: ${WORKSPACE}/build.log"'
        }
    }
}
