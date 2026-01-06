// Jenkinsfile for Black Duck SCA scans on Ubuntu/Linux agents using the Bridge CLI
// Requires: curl, unzip, git, Java, Maven on the Linux agent.

pipeline {
  agent any

  environment {
    // Credentials in Jenkins > Manage Credentials
    GITHUB_TOKEN              = credentials('hugo-github-token-2026')

    // Bridge/Black Duck configuration (Bridge picks these up from env)
    BRIDGE_BLACKDUCKSCA_URL   = 'https://partner-sca.demo.blackduck.com'
    BRIDGE_BLACKDUCKSCA_TOKEN = credentials('partner-sca-demo-instance')

    // Optional: forwarded to Detect by Bridge
    BRIDGE_DETECT_ARGS        = '--detect.project.name=hugo-goldendemo-webgoat-jenkins --detect.project.version.name=v1.0 --detect.excluded.detector.types=GIT'
    BRIDGE_DETECT_SEARCH_DEPTH = '5'

    // (Optional) where Bridge CLI lives if you want to reference it
    // Not used below, but you can switch the sh call to "$BRIDGE_HOME"
    BRIDGE_HOME               = '/home/ubuntu/blackduck/bridge-cli-bundle-linux64/bridge-cli'
  }

  stages {
    stage('Pre-Checkout') {
      steps {
        sh 'git config --global http.sslVerify false'
      }
    }

    stage('Checkout Source Code') {
      steps {
        echo 'Fetching source code from GitHub...'
        git branch: 'main',
            url: 'https://github.com/HUGO-AppSec/hugo-goldendemo-webgoat.git'
      }
    }

    stage('Build for Security Scan') {
      steps {
        sh 'mvn -B clean compile'
      }
    }

    stage('Black Duck Security Scan') {
      steps {
        script {
          // Ensure the CLI exists and is executable (optional but helpful)
          sh'''
            if [ ! -x "$BRIDGE_HOME" ]; then
              echo "ERROR: $BRIDGE_HOME not found or not executable."
              ls -l "$BRIDGE_HOME" || true
              exit 127
            fi
          '''

          // Run Bridge via bash -lc so PATH/env are set consistently
          def status = sh(
            returnStatus: true,
            label: 'BlackDuck Bridge CLI',
            script: '''
              bash -lc "$BRIDGE_HOME --stage blackducksca \
                blackducksca.scan.failure.severities=CRITICAL \
                blackducksca.scan.full=true"
            '''
          )

          echo "Black Duck Bridge exit code: ${status}"

          if (status == 8) {
            unstable('policy violation')
          } else if (status != 0) {
            error("bridge failure (exit ${status})")
          }
        }
      }
    }

    stage('Build Artifact') {
      steps {
        sh 'mvn clean package -DskipTests'
      }
    }    
  }


  post {
    success {
      echo 'Pipeline security scan completed successfully. Artifact built.'
      // ▶️ Added: Archive the JAR so you can download it from the build page
      archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }
    unstable {
      echo 'Pipeline marked UNSTABLE due to Black Duck SCA policy violation. Review results in the Black Duck Hub.'
    }
    failure {
      echo 'Pipeline failed. Check the Black Duck Hub for details.'
    }
  }
}