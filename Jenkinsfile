#!/bin/env groovy


def triggers = []

if("$BRANCH_NAME" == 'develop') {
    triggers << cron('H H(0-2) * * *')
}

@Library('cliqz-shared-library@vagrant') _

properties([
    disableConcurrentBuilds(),
    [$class: 'JobRestrictionProperty'],
    pipelineTriggers(triggers),
])

def vagrantfile = '''
require 'uri'

node_id = URI::encode(ENV['NODE_ID'] || '')
name = "mojave-#{ENV['BRANCH_NAME'] || ''}"

Vagrant.configure("2") do |config|
    config.vm.box = "mojave"

    config.vm.define "mojave" do |mojave|
        mojave.vm.hostname ="mojave"
        mojave.ssh.forward_agent = true

        config.vm.provider "parallels" do |prl|
            prl.name = name
            prl.memory = ENV["NODE_MEMORY"] || 8000
            prl.cpus = ENV["NODE_CPU_COUNT"] || 2
        end

        mojave.vm.provision "shell", privileged: false, run: "always", inline: <<-SHELL#!/bin/bash -l
            set -e
            set -x

            sudo mkdir -p /jenkins
            sudo chown vagrant /jenkins

            brew -v

            brew tap adoptopenjdk/openjdk
            # `which java` does not work as MacOS try to be smart and request jave installation
            java -version &>/dev/null || brew cask install adoptopenjdk8
            java -version

            curl -LO #{ENV['JENKINS_URL']}/jnlpJars/agent.jar
            nohup java -jar agent.jar -jnlpUrl #{ENV['JENKINS_URL']}/computer/#{node_id}/slave-agent.jnlp -secret #{ENV["NODE_SECRET"]} > agent.log 2> agent.log &
        SHELL
    end
end
'''

node('gideon') {
    stage('Start VM')

    writeFile file: 'Vagrantfile', text: vagrantfile

    vagrant.inside(
        'Vagrantfile',
        '/jenkins',
        4, // CPU
        8192, // MEMORY
        12000, // VNC port
        false, // rebuild image
    ) { nodeId ->
        node(nodeId) {
            stage('Checkout') {
                checkout scm
            }

            stage('Bootstrap') {
                timeout(40) {
                    ansiColor('xterm') {
                        sh '''#!/bin/bash -l
                            set -e
                            set -x

                            # For Cocoapods and Fastlane
                            export LC_ALL=en_US.UTF-8
                            export LANG=en_US.UTF-8

                            which node &>/dev/null || brew install node
                            node -v
                            npm -v

                            which carthage &>/dev/null || brew install carthage
                            carthage version

                            # for some reason we cannot install watchman in CI, it works when installed manually
                            which watchman &>/dev/null || brew install watchman
                            watchman --version

                            sudo xcode-select --switch /Applications/Xcode.app/
                            xcodebuild -version

                            pkgutil --pkg-info=com.apple.pkg.CLTools_Executables
                            sudo xcodebuild -license accept

                            sudo gem install which bundler || gem install bundler

                            ./bootstrap.sh
                        '''
                    }
                }
            }

            stage('Build') {
                withCredentials([
                    [
                        $class          : 'UsernamePasswordMultiBinding',
                        credentialsId   : '85859bba-4927-4b14-bfdf-aca726009962',
                        passwordVariable: 'GITHUB_PASSWORD',
                        usernameVariable: 'GITHUB_USERNAME',
                    ],
                    string(credentialsId: '8b4f7459-c446-4058-be61-3c3d98fe72e2', variable: 'ITUNES_USER'),
                    string(credentialsId: 'c21d2e60-e4b9-4f75-bad7-6736398a1a05', variable: 'SentryDSN'),
                    string(credentialsId: '05be12cd-5177-4adf-9812-809f01451fa0', variable: 'FASTLANE_PASSWORD'),
                    string(credentialsId: 'ea8c47ad-1de8-4300-ae93-ec9ff4b68f39', variable: 'MATCH_PASSWORD'),
                    string(credentialsId: 'ab91f92a-4588-4034-8d7f-c1a741fa31ab', variable: 'FASTLANE_ITC_TEAM_ID'),
                ]) {
                    timeout(20) {
                        ansiColor('xterm') {
                            sh '''#!/bin/bash -l
                                set -x
                                set -e

                                # For Cocoapods and Fastlane
                                export LC_ALL=en_US.UTF-8
                                export LANG=en_US.UTF-8

                                rm -rf /Users/vagrant/Library/Keychains/ios-build.keychain*

                                export MATCH_KEYCHAIN_NAME=ios-build.keychain

                                bundle exec fastlane CliqzNightly
                            '''
                        }
                    }
                }
            }

            stage('Upload') {
                def allBuilds = getAllBuilds(currentBuild)
                def newChangelog = getChangeString(allBuilds)

                def changelog = readFile 'CHANGELOG.md'
                writeFile file: 'CHANGELOG.md', text: """${changelog}

## [Unreleased]
${newChangelog}"""

                withCredentials([
                    string(credentialsId: '8b4f7459-c446-4058-be61-3c3d98fe72e2', variable: 'ITUNES_USER'),
                    string(credentialsId: 'f206e880-e09a-4369-a3f6-f86ee94481f2', variable: 'SENTRY_AUTH_TOKEN'),
                    string(credentialsId: 'ab91f92a-4588-4034-8d7f-c1a741fa31ab', variable: 'FASTLANE_ITC_TEAM_ID'),
                ]) {
                    timeout(60) {
                        ansiColor('xterm') {
                            sh """#!/bin/bash -l
                                set -x
                                set -e

                                # For Cocoapods and Fastlane
                                export LC_ALL=en_US.UTF-8
                                export LANG=en_US.UTF-8

                                bundle exec fastlane testfight
                            """
                        }
                    }
                }
            }
        }
    }
}

def getAllBuilds(build) {
    def results = [build]
    build = build.getPreviousBuild()
    while (build != null && build.result != 'SUCCESS') {
        results.add(build)
        build = build.getPreviousBuild()
    }
    return results
}

def getChangeString(builds) {
    def changeString = ""
    for (int x = builds.size() - 1; x >= 0; x--) {
        def currentBuild = builds[x];
        def buildNumber = currentBuild.number
        def changeLogSets = currentBuild.rawBuild.changeSets
        for (int i = 0; i < changeLogSets.size(); i++) {
            def entries = changeLogSets[i].items
            for (int j = 0; j < entries.length; j++) {
                def entry = entries[j]
                changeString += "- ${entry.msg} by ${entry.author} \n"
            }
        }
    }
    return changeString;
}