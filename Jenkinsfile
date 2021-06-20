//version 0.0.1
def gradleTasks = ''
def UUID_DIR = UUID.randomUUID().toString()
def buildAppend = """
allprojects {
    dependencyLocking {
        lockAllConfigurations()
    }
    task resolveAndLockAll {
        doFirst {
            assert gradle.startParameter.writeDependencyLocks
        }
        doLast {
            configurations.findAll {
                // Add any custom filtering on the configurations to be resolved
                it.canBeResolved
            }.each { it.resolve() }
        }
    }
}
allprojects {
    task copyLocks(type: Copy) {
        from 'gradle.lockfile'
        rename {
            "\${project.name}.lockfile"
        }
        into "\$rootDir/locks"
    }


}
task envReport() {
    doLast {
        def config = project.getBuildscript().getConfigurations().getByName("classpath")

        def configuration = config.getResolvedConfiguration()
        configuration.getResolvedArtifacts().each {
            file("\$rootDir/locks/plugins.lockfile") << (it.moduleVersion.toString() + '\\n')
        }
    }
}

task archiveLocks(type: Zip) {
    archiveFileName = "locks.zip"
    destinationDirectory = file("\$rootDir")
    from "\$rootDir/locks"
}"""


pipeline {
  agent {
    label {
      label 'local-jenkins'
      customWorkspace UUID_DIR
    }
  }
  parameters {
    string(name: 'gitUrl', description: 'test', defaultValue: 'master')
    string(name: 'branch', description: 'test', defaultValue: 'master')
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '15', artifactNumToKeepStr: '1'))
    timeout(time: 30, unit: 'MINUTES')
    skipStagesAfterUnstable()
    timestamps()
  }

  stages {
    stage('git repo') {
      steps {
        script {
          git credentialsId: 'git-credential', branch: params.branch, url: params.gitUrl
        }
      }
    }
    stage('Prepare Build') {
      steps {
        script {
          sh 'ls -lh'
          def readContent = readFile "build.gradle"
          writeFile file: "build.gradle", text: "$readContent\n$buildAppend"

          def readSettings = readFile "settings.gradle"
          writeFile file: "settings.gradle", text: "$readSettings\nenableFeaturePreview(\"ONE_LOCKFILE_PER_PROJECT\")"
          sh 'echo "build gradle to execute:"'
          sh 'cat build.gradle'
        }
      }
    }
    stage('Create Locks') {
      steps {
        script {
          sh './gradlew resolveAndLockAll  --write-locks'
          sh './gradlew copyLocks'
          sh 'echo "found locks"'
          sh 'ls -lh locks/'
          sh './gradlew envReport'
          sh './gradlew archiveLocks'
          sh 'echo "archived files"'
        }
      }
    }
    stage('Results') {
      steps {
        script {
          sh 'ls -lh locks'
          sh 'curl -X POST 192.168.1.37:8081/file -H "Content-Type: multipart/form-data" -F file=@locks.zip'
        }
      }
    }
  }
}
