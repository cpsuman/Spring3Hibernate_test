pipeline {
  agent {
    node {
      label "master"
      customWorkspace "workspace/${env.BRANCH_NAME}"
    }
  }
  // triggers {
  //       cron('30 16 * * *')
  // }
  parameters {
    string(name: 'PRODUCT_VERSION', defaultValue: '10.2.0.0.023.002', description: 'Version of SOM-B2B.')
    choice(name: 'USM_INSTALL_ACTIVITY', choices: ['Refresh', 'Full'], description: 'Pick installation activity type for USM')
    choice(name: 'SFO_INSTALLATION_ACTIVITY', choices: ['Refresh', 'Full'], description: 'Pick installation activity type for SFO')
  }
  options { skipDefaultCheckout() }
  stages {
    stage('Build') {
      when { anyOf { branch 'SOMB2B-DEV-STAGE'; branch 'SOMB2B-SIT'; branch 'SOMB2B-DEV-10.2Upgrade'; branch 'SOMB2B-DEV-TEMP'; branch 'SOMB2B-PROD'} }
       environment {
        NEXUS_DEPLOYMENT_CREDENTIAL = credentials('nexus_deployment_credential')
      }

      steps {
        echo 'Building ....'
        checkout scm
        dir(path: 'SFO') {
          sh """
            mvn \
            -Dmaven.skip.test=false \
            -Dnpm.resources=telstra \
            -Dmaven.javadoc.skip=true \
            -Dmaven.test.failure.ignore=true \
            -Drpm.product.release=${env.BUILD_ID} \
            -s ~/.m2_${env.BRANCH_NAME}/settings.xml \
            -gs ~/.m2_${env.BRANCH_NAME}/settings.xml \
            -Drpm.product.version=${params.PRODUCT_VERSION} \
            -Dtelstra.delivery.repository.address=${env.NEXUS_URL} \
            clean install
          """
        }
      }
    }
    stage('Code Quality') {
      when { expression { env.JENKINS_URL == 'https://jenkins.o2a.b2b.cloud.corp.telstra.com/' } }

      parallel {
        stage('SonarQube Scan'){
          environment {
            MAVEN_OPTS = "-XX:-UseSplitVerifier"
            SONARQUBE_AUTH_TOKEN = credentials('sonarqube_auth_token')
          }
          steps {
            dir(path: 'SFO') {
              sh """
                mvn \
                -Dsonar.host.url=${env.SONARQUBE_URL} \
                -Dsonar.login=${env.SONARQUBE_AUTH_TOKEN} \
                -Dsonar.branch=${env.SONARQUBE_BRANCH_PREFIX}_${env.BRANCH_NAME} \
                -Dtelstra.delivery.repository.address=${env.NEXUS_URL} \
                org.codehaus.mojo:sonar-maven-plugin:2.7:sonar
              """
            }
          }
        }
        stage('Coverity Scan'){
          when { expression { env.JENKINS_URL == 'https://jenkins.o2a.b2b.cloud.corp.telstra.com/' } }
          environment {
            COVERITY_SCAN_CREDENTIAL = credentials('coverity_scan_credential')
          }
          steps {
              dir(path: 'SFO'){
                sh """
                  /opt/coverity/bin/cov-build --dir ./idir \
                  --fs-capture-search . \
                  --fs-capture-search-exclude-regex "components/ui/sfo-war/target/" \
                  --fs-capture-search-exclude-regex "apps/sfo-ear/target/" \
                  --fs-capture-search-exclude-regex "apps/gt-war/target/gt-telstra" \
                  --fs-capture-search-exclude-regex "apps/gt-war/target/war" \
                  mvn \
                  -Dmaven.skip.test=true \
                  -Dmaven.test.failure.ignore=true \
                  -Dnpm.resources=telstra \
                  -Dmaven.javadoc.skip=true \
                  -Drpm.product.release=${BUILD_ID} \
                  -Drpm.product.version=${params.PRODUCT_VERSION} \
                  -Dtelstra.delivery.repository.address=${env.NEXUS_URL} \
                  clean install

                  /opt/coverity/bin/cov-analyze --dir ./idir --force --all --webapp-security --enable-constraint-fpp  -j auto --webapp-security-preview

                  /opt/coverity/bin/cov-commit-defects --dir ./idir --stream O2AStaticCodeAnalysis --user ${env.COVERITY_SCAN_CREDENTIAL_USR}  --password ${env.COVERITY_SCAN_CREDENTIAL_PSW} --host ${env.COVERITY_HOST} --https-port ${env.COVERITY_PORT} --on-new-cert trust
                """
              }
          }
        }
        stage('SourceClear Scan'){
          when { expression { env.JENKINS_URL == 'https://jenkins.o2a.b2b.cloud.corp.telstra.com/' } }
          environment {
            M2 = "/usr/local/bin/apache-maven-3.2.5/bin"
            M2_HOME = "/usr/local/bin/apache-maven-3.2.5"
            JAVA_HOME = "/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.212.b04-0.el7_6.x86_64/jre"

            SRCCLR_API_TOKEN = credentials('srcclr_api_token')
          }
          steps {
            dir(path: 'SFO') {
              sh """
                touch srcclr.yml
                echo "---" > srcclr.yml
                echo "custom_maven_command: clean install -Dmaven.test.skip=true -Dmaven.test.failure.ignore=true -Dnpm.resources=telstra -Dmaven.javadoc.skip=true -Drpm.product.release=${BUILD_ID} -Drpm.product.version=${params.PRODUCT_VERSION} -Dtelstra.delivery.repository.address=${env.NEXUS_URL}" >> srcclr.yml
                echo "..." >> srcclr.yml
                cat srcclr.yml
                export DEBUG=1
                curl -sSL https://download.sourceclear.com/ci.sh | sh
              """
            }
          }
        }
      }
    }
    stage('Artifact Upload') {
      when { anyOf { branch 'SOMB2B-DEV-STAGE'; branch 'SOMB2B-SIT'; branch 'SOMB2B-DEV-10.2Upgrade'; branch 'SOMB2B-DEV-TEMP'; branch 'SOMB2B-PROD'} }
      steps {
       sh """
        rm -rf ~/workspace/${env.BRANCH_NAME}/SFO/setup/acd/USM_build.number
        echo "PRODUCT.build.number=${env.BUILD_ID}" > ~/workspace/${env.BRANCH_NAME}/SFO/setup/acd/USM_build.number
        cd ~/workspace/${env.BRANCH_NAME}/SFO
        mkdir -p ~/workspace/Infra_Scripts_${env.BUILD_ID}
        wget -r -nH -nd -l1 --no-parent "http://indlin996:8081/nexus/content/repositories/Infra/Scripts/rpm_upload.ksh" -P ~/workspace/Infra_Scripts_${env.BUILD_ID}
        chmod +x  ~/workspace/Infra_Scripts_${env.BUILD_ID}/*
        ~/workspace/Infra_Scripts_${env.BUILD_ID}/rpm_upload.ksh ${env.NEXUS_URL}/nexus/content/repositories/rpm_release ${env.BRANCH_NAME} ${env.BUILD_ID}
        rm -rf ~/workspace/Infra_Scripts_${env.BUILD_ID}
       """
      }
    }
    stage('Backup'){
      when { anyOf { branch 'SOMB2B-DEV-STAGE'; branch 'SOMB2B-SIT'; branch 'SOMB2B-DEV-10.2Upgrade'; branch 'SOMB2B-DEV-TEMP'; branch 'SOMB2B-PROD'} }
      failFast true
      parallel {
        stage('Application') {
          when { expression { env.JENKINS_URL == 'https://jenkins.o2a.b2b.cloud.corp.telstra.com/' } }
          steps {
            echo "Backing up application AMIs ...."
          }
        }
        stage('Database') {
          when { expression { env.JENKINS_URL == 'https://jenkins.o2a.b2b.cloud.corp.telstra.com/' } }
          steps {
            echo "Backing up database AMIs ...."
          }
        }
      }
    }
    stage('USM Deployment') {
      agent {
        node {
          label "master"
          customWorkspace "pipeline/${env.BRANCH_NAME}"
        }
      }
      steps {
        script {
          if (env.JENKINS_URL == 'http://indlin5260.corp.amdocs.com:7070/') {
            p4sync(credential: 'PerforceCredentials', changelog: true, charset: 'utf-8', depotPath: '//TelstraXel', stream: '//TelstraXel/SOMB2B-Platform')
            sh """
              cp -r USM/Installation_Scripts $HOME/Installation_Scripts
              chmod -R 755 $HOME/Installation_Scripts
              $HOME/Installation_Scripts/bin/usm10_wrapper.sh ${params.USM_INSTALL_ACTIVITY} ${env.BUILD_ID} ${env.BRANCH_NAME} ${env.NEXUS_URL} LOCAL
            """
          }
          if (env.JENKINS_URL == 'https://jenkins.o2a.b2b.cloud.corp.telstra.com/' ) {
            git(url: 'https://git02.ae.sda.corp.telstra.com/scm/ed/somb2b-platform.git', branch: 'master', credentialsId: '19cfe01c-9e64-466b-8dde-80dd2734db2a')
            sh """
              cp -r USM/Installation_Scripts $HOME/Installation_Scripts
              chmod -R 755 $HOME/Installation_Scripts
              $HOME/Installation_Scripts/bin/usm10_wrapper.sh Refresh ${env.BUILD_ID} ${env.BRANCH_NAME} ${env.NEXUS_URL} SITE
            """
          }
        }
      }
    }
    stage('SFO Deployment') {
      agent {
        node {
          label "master"
          customWorkspace "pipeline/${env.BRANCH_NAME}"
        }
      }
      environment {
        PASSWORD = credentials('SOM_INSTALLATION_PASS')
      }
      steps{
        script {
          if (env.JENKINS_URL == 'http://indlin5260.corp.amdocs.com:7070/') {
            p4sync(credential: 'PerforceCredentials', changelog: true, charset: 'utf-8', depotPath: '//TelstraXel', stream: '//TelstraXel/SOMB2B-Platform')
            sh """
              cp -r SFO/Installation_Scripts_v10.2 $HOME/Installation_Scripts_v10.2
              chmod -R 755 $HOME/Installation_Scripts_v10.2
              ${HOME}/Installation_Scripts_v10.2/bin/install_odo_v102.sh ${env.BRANCH_NAME} ${env.BUILD_ID} ${env.SFO_INSTALLATION_ACTIVITY} Yes No LOCAL
            """
          }
          if (env.JENKINS_URL == 'https://jenkins.o2a.b2b.cloud.corp.telstra.com/' ) {
            git(url: 'https://git02.ae.sda.corp.telstra.com/scm/ed/somb2b-platform.git', branch: 'master', credentialsId: '19cfe01c-9e64-466b-8dde-80dd2734db2a')
            sh """
              cp -r SFO/Installation_Scripts_v10.2 $HOME/Installation_Scripts_v10.2
              chmod -R 755 $HOME/Installation_Scripts_v10.2
              ${HOME}/Installation_Scripts_v10.2/bin/install_odo_v102.sh ${env.BRANCH_NAME} ${env.BUILD_ID} ${env.SFO_INSTALLATION_ACTIVITY} Yes No STAGE2
            """
          }
        }
      }  
    }
  }
}
