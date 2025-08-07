def buildAll = false
def affectedModules = [] as Set
def goal = "package"

pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }

    tools {
        maven 'MAVEN_3'
        jdk 'OPENJDK_11'
    }

    stages {
        stage("Analyze Changes") {
            steps {
                script {
                    List<String> changes = []

                    if (env.CHANGE_ID) {
                        echo "Pull Request Trigger"
                        def diffCmd = isUnix() 
                            ? "git --no-pager diff origin/${CHANGE_TARGET} --name-only" 
                            : "git --no-pager diff origin/${CHANGE_TARGET} --name-only"
                        def output = isUnix() 
                            ? sh(script: diffCmd, returnStdout: true) 
                            : bat(script: diffCmd, returnStdout: true)
                        changes = output.trim().split()
                        goal = "compile"
                    } else if (currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')) {
                        echo "Manual Trigger"
                        buildAll = true
                    } else {
                        echo "Push Trigger"
                        currentBuild.changeSets.each { cs ->
                            cs.items.each { entry ->
                                entry.affectedFiles.each { file ->
                                    changes << file.path
                                }
                            }
                        }
                    }

                    for (c in changes) {
                        if (c == "pom.xml" || c.contains("common")) {
                            buildAll = true
                            affectedModules.clear()
                            break
                        } else if (c.contains("/")) {
                            affectedModules << c.tokenize("/")[0]
                        }
                    }
                }
            }
        }

        stage("Build All Modules") {
            when {
                expression { buildAll }
            }
            steps {
                script {
                    def cmd = "mvn clean ${goal} -B -DskipTests -Pbuild -T 5"
                    isUnix() ? sh(cmd) : bat(cmd)
                }
            }
        }

        stage("Build Affected Modules") {
            when {
                expression { affectedModules }
            }
            steps {
                script {
                    def modules = affectedModules.join(",")
                    def cmd = "mvn clean ${goal} -B -pl ${modules} -am -DskipTests -Pbuild -T 5"
                    isUnix() ? sh(cmd) : bat(cmd)
                }
            }
        }
    }
}