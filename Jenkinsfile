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
          git branch: params.branch, url: params.gitUrl
        }
      }
    }
    stage('Prepare Build') {
      steps {
        scripts {
          sh 'ls -lh'
          readContent = readFile "build.gradle"
          writeFile file: "build.gradle", text: "$readContent\n$buildAppend"

          readSettings = readFile "settings.gradle"
          writeFile file: "settings.gradle", text: "$readSettings\nenableFeaturePreview(\"ONE_LOCKFILE_PER_PROJECT\")"
        }
      }
    }
    stage('Create Locks') {
      steps {
        scripts {
          sh './gradlew resolveAndLockAll  --write-locks'
          sh './gradlew copyLocks'
          sh './gradlew envReport'
          sh './gradlew archiveLocks'
        }
      }
    }
    stage('Results') {
      steps {
        scripts {
          sh 'ls -lh locks'
          sh 'curl -X POST 192.168.1.37:8099/data -H "Content-Type: application/zip" --data-binary @locks.zip'
        }
      }
    }
  }
}
