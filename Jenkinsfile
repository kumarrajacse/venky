#!groovy

//Removed use of Docker Plugin due to Service Account not being member of Docker Group
//eg: druntime = docker.build(BASE_IMAGE,"-f Dockerfile.agent .")
//removed squash as well,until new nodes online that support this arg
//Need slack integration...cmjenkins2 doesn't have access, but edp does I think

String idartcred = '82858c6b-89f8-4f74-9ddf-bf3baf90bb4f'
String idgitcred = 'edpBB'
nodeName='Docker'
projName = 'oms'
activeEar = 'smcfs'
DevActiveEars = 'smcfs'
allEnv=0
boolean keepChecking=false
boolean BUILD_CONTAINER=true
ORIG_BLD_NUM=BUILD_NUMBER

cfMaintRepo = "ssh://git@bitbucket.service.edp.t-mobile.com:7999/omni/cf_maint.git"
runBashRepo = "ssh://git@bitbucket.service.edp.t-mobile.com:7999/~cwilker15/run_bash.git"

properties([parameters([
   booleanParam(defaultValue: false, description: 'Check only if using existing build for pipeline', name: 'RESTART_BUILD'), \
   booleanParam(defaultValue: false, description: 'Not evaluated unless RESTART_BUILD is checked.', name: 'START_WITH_AGENTS'), \
   string(defaultValue: '0', description: 'Not evaluated unless RESTART_BUILD is checked.', name: 'RESTART_BUILD_NUMBER', trim: true), \
   string(defaultValue: '0', description: 'Not evaluated unless RESTART_BUILD is checked.', name: 'STARTING_ENV', trim: false), \
  ]), pipelineTriggers([[$class: 'PeriodicFolderTrigger', interval: '1d']])])
RESTART_BUILD=params.RESTART_BUILD
START_WITH_AGENTS=params.START_WITH_AGENTS
if (START_WITH_AGENTS) { BUILD_CONTAINER=false }
RESTART_BUILD_NUMBER=params.RESTART_BUILD_NUMBER
STARTING_ENV=params.STARTING_ENV
echo "RESTART_BUILD=${RESTART_BUILD}, START_WITH_AGENTS=${START_WITH_AGENTS}, RESTART_BUILD_NUMBER=${RESTART_BUILD_NUMBER}, STARTING_ENV=${STARTING_ENV}"
   
if (RESTART_BUILD) {
  if ( params.RESTART_BUILD_NUMBER.equals("0") || params.STARTING_ENV.equals("0") ) { 
    echo 'RESTART_BUILD_NUMBER and/or STARTING_ENV values cannot equal 0 when restarting build.'
    currentBuild.result = 'FAILURE'
    return
  }
  ORIG_BLD_NUM=RESTART_BUILD_NUMBER
  keepChecking=true  
  node(nodeName) { 
    JWORKSPACE=WORKSPACE
    try {
      sh """
        curl -s ${JOB_URL}${RESTART_BUILD_NUMBER}/changes > ${JWORKSPACE}/GHASH0.txt
        if [[ \"\$(grep '<h2>Summary</h2>' GHASH0.txt|wc -l)\" -gt 1 ]] ; then
          echo \$(cat GHASH0.txt)|grep -oP '(?<=<h2>Summary</h2>).*(?=<h2>Summary</h2>)' > GHASH1.txt
          mv GHASH1.txt GHASH0.txt
        fi
        echo COMMIT_ID=\$(sed 's/.*<b> Commit //' GHASH0.txt |cut -d' ' -f1|tail -1)  > GHASH.txt
        if [ \"\$(cat GHASH.txt)\" == \"COMMIT_ID=\" ] ; then
          echo COMMIT_ID=\$(grep -A 1 '^[ ]*Commit[ ]*\$' GHASH0.txt | sed 's|[ ]*||g' | tail -1) > GHASH.txt
        fi
      """
    } catch(err) {
      nochange()
      currentBuild.result = "FAILURE"
      throw err
    }
    GHASH = readProperties file: "${JWORKSPACE}/GHASH.txt"
    if (GHASH.COMMIT_ID.equals("")) {
      nochange()
      currentBuild.result = 'FAILURE'
      sh "exit 1"  
    }    
  }
} 

projRepo = "ssh://git@bitbucket.service.edp.t-mobile.com:7999/omni/${projName}.git"
keyRepo = 'ssh://git@bitbucket.service.edp.t-mobile.com:7999/omnicp/npe.git'
entityRepo = 'ssh://git@bitbucket.service.edp.t-mobile.com:7999/omni/oms_entity_extensions.git'
stage ("Checkout") {
  timestamps {
    node(nodeName) {
      if (RESTART_BUILD) {
        checkout([$class: 'GitSCM', branches: [[name: GHASH.COMMIT_ID ]], doGenerateSubmoduleConfigurations: false, extensions:  [[$class: 'CleanBeforeCheckout']], gitTool: 'jgit', submoduleCfg: [], userRemoteConfigs: [[credentialsId: idgitcred, url: projRepo ]]])
        settings = readYaml file: "Jenkinsfile.yaml"
      } else {   
        checkout([$class: 'GitSCM', branches: scm.branches, doGenerateSubmoduleConfigurations: false, extensions:  [[$class: 'CleanBeforeCheckout']], gitTool: 'jgit', submoduleCfg: [], userRemoteConfigs: [[credentialsId: idgitcred, url: projRepo ]]])
        settings = readYaml file: "Jenkinsfile.yaml"
        try {
          checkout([$class: 'GitSCM', branches: scm.branches, doGenerateSubmoduleConfigurations: false, extensions:  [[$class: 'CleanBeforeCheckout'], [$class: 'RelativeTargetDirectory', relativeTargetDir: 'entities']], gitTool: 'jgit', submoduleCfg: [], userRemoteConfigs: [[credentialsId: settings.global.idgitcred, url: entityRepo ]]])
        } catch(err) {
          echo "Unable to find ${BRANCH_NAME} in OMS_ENTITY_EXTENSIONS repo. Entity checkout failure."
          currentBuild.result = "FAILURE"
          throw err
        }
      }
      checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions:  [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'key']], gitTool: 'jgit', submoduleCfg: [], userRemoteConfigs: [[credentialsId: idgitcred, url: keyRepo ]]])

      sh """
        mkdir python
      """

      checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions:  [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'python/cf_maint']], gitTool: 'jgit', submoduleCfg: [], userRemoteConfigs: [[credentialsId: idgitcred, url: cfMaintRepo ]]])
      checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions:  [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'python/run_bash']], gitTool: 'jgit', submoduleCfg: [], userRemoteConfigs: [[credentialsId: idgitcred, url: runBashRepo ]]])

      JWORKSPACE=WORKSPACE
      withEnv(["JWORKSPACE=${WORKSPACE}"]) {
        sh script: """
          sed -i \"s|SOURCE_IMAGE|${settings.global.omsBaseUrl}/${settings.global.omsAgentBase}|g\" ${JWORKSPACE}/Dockerfile.agent
          sed -i \"s|SOURCE_IMAGE|${settings.global.omsBaseUrl}/${settings.global.omsBaseImage}|g\" ${JWORKSPACE}/Dockerfile.base
          sed -i \"s|SOURCE_IMAGE|${settings.global.omsBaseUrl}/wasnd:${settings.global.wasND}|g\" ${JWORKSPACE}/Dockerfile.ear
          chmod a+x $JWORKSPACE/containers/*.sh ${JWORKSPACE}/Dockerfile.*
          $JWORKSPACE/containers/exposePorts.sh ${settings.global.agentRmiPortL} ${settings.global.agentRmiPortH} ${settings.global.omsRmiPortL} ${settings.global.omsRmiPortH}
          rm -fr ${JWORKSPACE}/sharedFiles ${JWORKSPACE}/*.skip ${JWORKSPACE}/agents.deploy $JWORKSPACE/usedPorts.txt && mkdir -p ${JWORKSPACE}/sharedFiles
        """
      }
      stash projName
    }
  }
}

echo "Active environments:"
for (xxEnv = 0; xxEnv < settings.deploy.size();xxEnv++) {
  echo "  ${xxEnv} :: ${settings.deploy[xxEnv].env}"
}
echo "Starting Environment: ${STARTING_ENV}"


ROOT_IMAGE="${settings.global.omsBaseUrl}/${settings.global.omsBuildImage}-${BRANCH_NAME}"
BASE_IMAGE="${ROOT_IMAGE}:${settings.global.omsBuildVersion}.${ORIG_BLD_NUM}-base"
AGENT_IMAGE="${ROOT_IMAGE}:${settings.global.omsBuildVersion}.${ORIG_BLD_NUM}-agent"

SKIP_CDT=settings.deploy[allEnv].skipCDT
CDT_ENTITY_BUILD=settings.deploy[allEnv].useEntityCDTfromBuild
if (SKIP_CDT.equals("true")) {
    CDT_ENTITY_BUILD="false"
	echo "CDT_ENTITY_BUILD = ${CDT_ENTITY_BUILD}"
	} 
else {
    if (CDT_ENTITY_BUILD.equals(null) || CDT_ENTITY_BUILD.equals("")) { CDT_ENTITY_BUILD="true" }
    echo "CDT_ENTITY_BUILD = ${CDT_ENTITY_BUILD}"
    currEnv=settings.deploy[allEnv].env
    }

if (STARTING_ENV.equals("0") && BUILD_CONTAINER) {
  stage("${currEnv}") {
    node(nodeName) {
      timestamps {
        JWORKSPACE=WORKSPACE
        withCredentials([usernamePassword(credentialsId: settings.global.idartcred, passwordVariable: 'APASS', usernameVariable: 'AUSER')]) {
          sh script: """
            rm -fr /tmp/.cf_* ~/.cf
            sudo docker login ${settings.global.omsBaseUrl} -u ${AUSER} -p '${APASS}'
            xxx=\$(sudo docker ps -aq) && if [ "\${xxx}z" != "z" ] ; then sudo docker stop \$(sudo docker ps -aq) ; fi
            xxx=\$(sudo docker ps -aq) && if [ "\${xxx}z" != "z" ] ; then sudo docker rm \$(sudo docker ps -aq) ; fi
            if (( \$(echo \$(sudo docker images $ROOT_IMAGE:*|wc -l)) > 1 )) ; then
              sudo docker rmi -f \$(sudo docker image list -q $ROOT_IMAGE:*) && true
            fi
            if [ \"\$(sudo docker images --filter 'dangling=true' -q --no-trunc)x\" != \"x\" ] ; then sudo docker rmi \$(sudo docker images --filter \"dangling=true\" -q --no-trunc) ; fi
            rm -fr ${JWORKSPACE}/sharedFiles && mkdir -p ${JWORKSPACE}/sharedFiles
          """
        }
        unstash projName
        stage ("base") {
          lock("${projName}.Entity-${currEnv}") {
            try {
              sh script: """
                rm -fr ${JWORKSPACE}/sharedFiles && mkdir -p ${JWORKSPACE}/sharedFiles
                sudo docker pull ${settings.global.omsBaseUrl}/${settings.global.omsBaseImage}
                sudo docker build -t ${BASE_IMAGE} -f Dockerfile.base --build-arg OMS_BASE=${settings.global.omsBase} \
                  --build-arg OMS_DEPENDS=${settings.global.omsDepends} --build-arg CDT_EXPORT=/tmp/jenkins/containers/CDT_EXPORT \
                  --build-arg SRC_DIR=/tmp/jenkins --build-arg CP_REPO=${settings.deploy[allEnv].cpRepo} \
                  --build-arg ACTIVE_EAR=${activeEar} --build-arg DB_UPDATE=${settings.deploy[allEnv].updateEntityCDT} \
                  --build-arg CP_BASE=${settings.deploy[allEnv].cpBase} --build-arg ENV_REPO=${settings.deploy[allEnv].envRepo} \
                  --build-arg BUILD_URL=${RUN_DISPLAY_URL} --build-arg ENV=${settings.deploy[allEnv].env} \
                  --build-arg JUNIT=${settings.global.junit} --build-arg SKIP_CDT=${settings.deploy[allEnv].skipCDT} \
		              --build-arg CDT_BASE_URL=${settings.global.cdtBaseUrl} --build-arg CDT_ENTITY_BUILD=${CDT_ENTITY_BUILD} \
                  --build-arg SOURCE_BUILD_STREAM=${BRANCH_NAME} --build-arg SOURCE_BUILD_NUMBER=${ORIG_BLD_NUM} .
                sudo docker run -i --rm -v ${JWORKSPACE}/sharedFiles:/tmp/sharedFiles -e CDT_EXPORT=/tmp/jenkins/containers/CDT_EXPORT \
                  -e JUNIT=${settings.global.junit} --entrypoint /tmp/jenkins/containers/get_artifacts.sh ${BASE_IMAGE}
                sudo docker push ${BASE_IMAGE}
              """
              if (settings.global.junit) {
                junit 'sharedFiles/TEST-*.xml'
              }
            } catch(err) {
              sh script: """
                if [ "\$(sudo docker ps -a | grep Exit)x\" != \"x\" ] ; then sudo docker ps -a | grep Exit | cut -d' ' -f1|xargs sudo docker rm ; fi
                if [ \"\$(sudo docker images --filter 'dangling=true' -q --no-trunc)x\" != \"x\" ] ; then sudo docker rmi \$(sudo docker images --filter \"dangling=true\" -q --no-trunc) ; fi
              """
              currentBuild.result = "FAILURE"
              throw err
            }
          }
        }
        stash includes: "sharedFiles/", name: "sharedFiles"
        if (CDT_ENTITY_BUILD.equals("true")) {
          echo "Uploading cdt.zip/entities.jar to artifactory"
          cmArtifactUpload("${JWORKSPACE}/sharedFiles/cdt.zip","tmo-pipeline-releases-dev/com/tmobile/omni/oms/${BRANCH_NAME}/${settings.global.omsBuildVersion}.${ORIG_BLD_NUM}/")
          cmArtifactUpload("${JWORKSPACE}/sharedFiles/entities.jar","tmo-pipeline-releases-dev/com/tmobile/omni/oms/${BRANCH_NAME}/${settings.global.omsBuildVersion}.${ORIG_BLD_NUM}/")
          cmArtifactUpload("${JWORKSPACE}/sharedFiles/resources.jar","tmo-pipeline-releases-dev/com/tmobile/omni/oms/${BRANCH_NAME}/${settings.global.omsBuildVersion}.${ORIG_BLD_NUM}/")
        } else {
          echo "CDT_ENTITY_BUILD=${CDT_ENTITY_BUILD} thus CDT extract was not done...skipping artifactory upload of cdt.zip"        
        }
        sh script: """
          rm -fr ${JWORKSPACE}/sharedFiles && mkdir -p ${JWORKSPACE}/sharedFiles
        """
        String sq = steps.tool 'SonarQube Scanner 2.8'
        checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions:  [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'sharedFiles']], gitTool: 'jgit', submoduleCfg: [], userRemoteConfigs: [[credentialsId: idgitcred, url: settings.deploy[allEnv].envRepo ]]])
        unstash 'sharedFiles'
        // iSteps = DevActiveEars + ",SonarQube,agents"
	// removed sonarqube as we are facing errors connecting
        iSteps = DevActiveEars + ",agents"

        iList = iSteps.split(',')
        def iLoops = [:]
        for (int i = 0; i < iList.size() ; i++) {
          def iActive = "${iList[i]}"
          iLoops["${iActive}"] = {
            stage (iActive) {
              try {
                sh script: """
                  if [ "${iActive}" == "SonarQube" ] ; then
                    sudo docker run -i --rm --entrypoint /tmp/jenkins/containers/sonarQube.sh \
                      -e projName=${projName} -e sonarHost=${settings.global.sonarHost} \
                      -e branch=${BRANCH_NAME} -e buildNumber=${ORIG_BLD_NUM} -e buildBase=${settings.global.omsBuildVersion} \
                      -v ${sq}/sonar-scanner-2.8:/tmp/sonarqube -v /apps/jdk:/tmp/jdk  ${BASE_IMAGE}
                    touch ${JWORKSPACE}/${iActive}.skip
                  else if [ "$iActive" == "agents" ] ; then
                          sudo docker pull ${settings.global.omsBaseUrl}/${settings.global.omsAgentBase}
                          sudo docker build --rm -t ${AGENT_IMAGE} -f Dockerfile.agent \
                            --build-arg BUILD_URL=${RUN_DISPLAY_URL} --build-arg OMS_DEPENDS=${settings.global.omsDepends} .
                          sudo docker push ${AGENT_IMAGE} && sudo docker rmi ${AGENT_IMAGE}
                          #Let all other parallel loops start then create file used in next section
                          sleep 10 && touch ${JWORKSPACE}/${iActive}.deploy
                       fi
                  fi
                """
              } catch(err) {
                sh script: """
                  echo "Failed in $iActive section..."
                  if [ "\$(sudo docker ps -a | grep Exit)x\" != \"x\" ] ; then sudo docker ps -a | grep Exit | cut -d' ' -f1|xargs sudo docker rm ; fi
                  if [ \"\$(sudo docker images --filter 'dangling=true' -q --no-trunc)x\" != \"x\" ] ; then sudo docker rmi \$(sudo docker images --filter \"dangling=true\" -q --no-trunc) ; fi
                  sudo docker rmi ${BASE_IMAGE} ${AGENT_IMAGE} && true
                """
                currentBuild.result = 'FAILURE'
              }
              try {
                if (fileExists("${JWORKSPACE}/${iActive}.deploy")) {
                  agentInstall(allEnv)
                } else {
                  if (!fileExists("${JWORKSPACE}/${iActive}.skip")) {
                    EAR_IMAGE="${settings.global.omsBaseUrl}/${settings.global.omsBuildImage}-${BRANCH_NAME}:${settings.global.omsBuildVersion}.${ORIG_BLD_NUM}-${iActive}"
                    sh script: """
                      sudo docker pull ${settings.global.omsBaseUrl}/wasnd:${settings.global.wasND}
                      sudo docker build -t ${EAR_IMAGE} -f Dockerfile.ear \
                        --build-arg CURR_EAR=${iActive} --build-arg BUILD_URL=${RUN_DISPLAY_URL} \
                        --build-arg SRC_DIR=/tmp/jenkins --build-arg CP_REPO=${settings.deploy[allEnv].cpRepo} --build-arg VM_MODELS=${settings.global.models} \
                        --build-arg CP_BASE=${settings.deploy[allEnv].cpBase} --build-arg ENV_REPO=${settings.deploy[allEnv].envRepo} \
                        --build-arg ENV_CF_ORG=${settings.deploy[allEnv].cfOrg} --build-arg ENV_CF_SPACE=${settings.deploy[allEnv].cfSpace} \
                        --build-arg OMS_DEPENDS=${settings.global.omsDepends} --build-arg VM_DEPENDS=${settings.global.vmDepends} \
                        --build-arg OMS_BASE=${settings.global.omsBase} .
                      sudo docker push ${EAR_IMAGE} && sudo docker rmi ${EAR_IMAGE}
                    """
                    pcfPush(projName,allEnv,iActive,ORIG_BLD_NUM)
                  }
                }
              } catch(err) {
                sh script: """
                  sudo docker rmi ${BASE_IMAGE} ${EAR_IMAGE} && true
                  if [ "\$(sudo docker ps -a | grep Exit)x\" != \"x\" ] ; then sudo docker ps -a | grep Exit | cut -d' ' -f1|xargs sudo docker rm ; fi
                  if [ \"\$(sudo docker images --filter 'dangling=true' -q --no-trunc)x\" != \"x\" ] ; then sudo docker rmi \$(sudo docker images --filter \"dangling=true\" -q --no-trunc) ; fi
                """
                currentBuild.result = "FAILURE"
                throw err
              }
            }
          }
        }
        parallel iLoops
        sh ("sudo docker rmi ${BASE_IMAGE}")
        fix_containers()
      }
    }
  }
} else {
  echo "Skipping docker image builds..."
} // end Skip

if (START_WITH_AGENTS) {
  if (STARTING_ENV.equals(settings.deploy[allEnv].env)) {
    node(nodeName) {
      sh("rm -fr ${JWORKSPACE}/sharedFiles && mkdir -p ${JWORKSPACE}/sharedFiles")
      agentInstall(allEnv)
    }
  }
}

// Subsequent Envs
for (allEnv = 1; allEnv < settings.deploy.size();allEnv++) {
  currEnv=settings.deploy[allEnv].env
  if (keepChecking) {
    echo "STARTING_ENV: ${STARTING_ENV}, currEnv: ${currEnv}, keepChecking: ${keepChecking}"
    if (!currEnv.equals(STARTING_ENV)) {
      echo "Skipping ${currEnv}"
      continue
    }
  } else {
    echo "Processing ${currEnv}"
    keepChecking=false
  }
  if (START_WITH_AGENTS) {
    if (settings.deploy[allEnv].approval) {
      timeout(20) {
        input message: "Restart at deploy Agents for ${currEnv}", submitter: settings.deploy[allEnv].approvalList, submitterParameter: 'approver'
      }
    }
    node(nodeName) {
     sh("rm -fr ${JWORKSPACE}/sharedFiles && mkdir -p ${JWORKSPACE}/sharedFiles")
     agentInstall(allEnv)
    }
  } else {
    timestamps {
      if (settings.deploy[allEnv].approval) {
        timeout(20) {
          input message: "Deploy to ${currEnv}", submitter: settings.deploy[allEnv].approvalList, submitterParameter: 'approver'
        }
      }
    }
    stage (currEnv) {
      node(nodeName) {
        timestamps {
          JWORKSPACE=WORKSPACE
          unstash projName
          withCredentials([usernamePassword(credentialsId: settings.global.idartcred, passwordVariable: 'APASS', usernameVariable: 'AUSER')]) {
            lock("${projName}.EntityCDT-${currEnv}") {
              try {
                sh script: """
                  rm -fr ${JWORKSPACE}/sharedFiles && mkdir -p ${JWORKSPACE}/sharedFiles
                  sudo docker login ${settings.global.omsBaseUrl} -u ${AUSER} -p '${APASS}'
                  sudo docker run -i --rm -e CP_BASE=${settings.deploy[allEnv].cpBase} -e SRC_DIR=/tmp/jenkins \
                    -e CP_REPO=${settings.deploy[allEnv].cpRepo} -e ENV_REPO=${settings.deploy[allEnv].envRepo} \
                    -e CDT_EXPORT=/tmp/jenkins/containers/CDT_EXPORT \
                    -e DB_UPDATE=${settings.deploy[allEnv].updateEntityCDT} \
                    -e CDT_ENTITY_BUILD=${CDT_ENTITY_BUILD} \
                    --entrypoint /tmp/jenkins/containers/entity_cdt.sh ${BASE_IMAGE}
                """
              } catch(err) {
                sh script: """
                  echo 'Failed in Ear Image section...'
                  if [ "\$(sudo docker ps -a | grep Exit)x\" != \"x\" ] ; then sudo docker ps -a | grep Exit | cut -d' ' -f1|xargs sudo docker rm ; fi
                  if [ \"\$(sudo docker images --filter 'dangling=true' -q --no-trunc)x\" != \"x\" ] ; then sudo docker rmi \$(sudo docker images --filter \"dangling=true\" -q --no-trunc) ; fi
                  sudo docker rmi ${BASE_IMAGE} && true
                """
                currentBuild.result = "FAILURE"
                throw err
              }
              sh("xx=\$(echo \$(sudo docker images ${BASE_IMAGE}|wc -l)) && if (( \$xx > 1 )) ; then sudo docker rmi ${BASE_IMAGE} ; fi ")
            }
          }
        }
      }
      node(nodeName) {
        timestamps {
          JWORKSPACE=WORKSPACE
          unstash projName
          sh("rm -fr /tmp/.cf_* ~/.cf && touch ${JWORKSPACE}/agents.deploy")
          checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions:  [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'sharedFiles']], gitTool: 'jgit', submoduleCfg: [], userRemoteConfigs: [[credentialsId: idgitcred, url: settings.deploy[allEnv].envRepo ]]])
          try {
            iSteps = "${activeEar},${settings.global.buildEars},agents"
            iList = iSteps.split(',')
            def iLoops = [:]
            for (int i = 0; i < iList.size() ; i++) {
              def iActive = "${iList[i]}"
              iLoops["${iActive}"] = {
                stage (iActive) {
                  if (fileExists("${JWORKSPACE}/${iActive}.deploy")) {
                    sh("rm -f ${JWORKSPACE}/${iActive}.deploy")
                    agentInstall(allEnv)
                  } else {
                    pcfPush(projName,allEnv,iActive,ORIG_BLD_NUM)
                  }
                }
              }
            }
            parallel iLoops
          } catch(err) {
            currentBuild.result = "FAILURE"
            throw err
          }
          fix_containers()
        }
      }
    }
  }
}

def pcfPush(appType,cEnv,cfApp,bnum) {
  JWORKSPACE=WORKSPACE
  cfHome = steps.tool 'cf'
  String idpcf = 'pcf-order-order-management-system-NPE'
  lock("${appType}.${cfApp}-${settings.deploy[cEnv].env}.PCF") {
    cfAppFile="${JWORKSPACE}/sharedFiles/${settings.deploy[cEnv].cpBase}/${appType}-${cfApp}"
    if (appType.equals("agent")) {
      cfAppName="${settings.global.omsBaseUrl}/${settings.global.omsBuildImage}-${BRANCH_NAME}:${settings.global.omsBuildVersion}.${bnum}-${appType}"
      RPL=settings.global.agentRmiPortL
      RPH=settings.global.agentRmiPortH
    } else {
      cfAppName="${settings.global.omsBaseUrl}/${settings.global.omsBuildImage}-${BRANCH_NAME}:${settings.global.omsBuildVersion}.${bnum}-${cfApp}"
      RPL=settings.global.omsRmiPortL
      RPH=settings.global.omsRmiPortH
    }
    envName=settings.deploy[cEnv].env
    ENV_REPO=settings.deploy[cEnv].envRepo
    CP_BASE=settings.deploy[cEnv].cpBase
    CF_DOMAIN=settings.deploy[cEnv].cfDomain
    CF_SPACE=settings.deploy[cEnv].cfSpace
    CF_URL=settings.deploy[cEnv].cfUrl
    CF_ORG=settings.deploy[cEnv].cfOrg
    REMOTE_DIR=settings.global.remoteDir
    DOMAIN_URL=settings.deploy[cEnv].cfDomain
    REMOTE_LOG=settings.global.remoteLog
    OMS_DEPENDS=settings.global.omsDepends
    SYSLOG=settings.deploy[cEnv].sysLog
    DEPLOY_EARS=settings.deploy[cEnv].deployEars
    CF_DEBUG=settings.deploy[cEnv].cfDebug
    try {
      withCredentials([usernamePassword(credentialsId: idpcf, usernameVariable: 'PUSER', passwordVariable: 'PPASS')]) {
        withEnv(["envName=${envName}","cfAppFile=${cfAppFile}","cfAppName=${cfAppName}","cfHome=${cfHome}",
                 "CP_REPO=${settings.deploy[cEnv].cpRepo}","CP_BASE=${CP_BASE}", "appType=${appType}", "CF_DEBUG=${CF_DEBUG}", 
                 "cfApp=${cfApp}", "RPL=${RPL}", "RPH=${RPH}", "ENV_REPO=${ENV_REPO}", "SYSLOG=${SYSLOG}",
                 "CF_SPACE=${CF_SPACE}", "CF_DOMAIN=${CF_DOMAIN}","CF_URL=${CF_URL}", "CF_ORG=${CF_ORG}",
                 "REMOTE_LOG=${REMOTE_LOG}", "REMOTE_DIR=${REMOTE_DIR}", "JWORKSPACE=${JWORKSPACE}",
                 "cEnv=${cEnv}", "DOMAIN_URL=${DOMAIN_URL}", "OMS_DEPENDS=${OMS_DEPENDS}", "DEPLOY_EARS=${DEPLOY_EARS}" ]) {
          sh('containers/pcfPush.sh')
        }
      }
    } catch(err) {
      currentBuild.result = "FAILURE"
      throw err
    }
  }
}

def agentInstall(allEnv) {
  JWORKSPACE=WORKSPACE
  cfHome = steps.tool 'cf'
  unstash projName
  //sh("rm -fr ${JWORKSPACE}/sharedFiles && mkdir -p ${JWORKSPACE}/sharedFiles")
  checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions:  [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'sharedFiles']], gitTool: 'jgit', submoduleCfg: [], userRemoteConfigs: [[credentialsId: settings.global.idgitcred, url: settings.deploy[allEnv].envRepo ]]])
  activeAgents = readYaml file: "sharedFiles/${settings.deploy[allEnv].cpBase}/agent.yml"
  AgentSteps = activeAgents.active[0].agent
  for (i = 0; i < activeAgents.active.size();i++) {
    AgentSteps = AgentSteps + "," + activeAgents.active[i].agent
  }
  AgentList = AgentSteps.split(',')
  def AgentLoops = [:]
  for (int allAgents = 0; allAgents < AgentList.size() ; allAgents++) {
    def AgentActive = "${AgentList[allAgents]}"
    AgentLoops["${AgentActive}"] = {
      stage (AgentActive) {
        pcfPush('agent',allEnv,AgentActive,ORIG_BLD_NUM)
      }
    }
  }
  parallel AgentLoops
}

def nochange() {
  echo "Unable to find changes for this ${BRANCH_NAME} in build number: ${RESTART_BUILD_NUMBER}"
  echo "*** Suggestion ***"
  echo "Click here:"
  echo "  ${JOB_URL}/${RESTART_BUILD_NUMBER}/changes"
  echo "     and select 'Previous Build' until you find one that works"
  echo " or ${JOB_URL}/changes to find next candidate build."
}

def fix_containers() {
  if (settings.deploy[allEnv].fixContainers != null && settings.deploy[allEnv].fixContainers) {
    String idpcf = 'pcf-order-order-management-system-NPE'
    withCredentials([usernamePassword(credentialsId: idpcf, usernameVariable: 'PUSER', passwordVariable: 'PPASS')]) {
      try {
        cfHome = steps.tool 'cf'
        sh script: """
          export PATH="${cfHome}:\$PATH"
          cf login --skip-ssl-validation -a ${CF_URL} -u \${PUSER} -p \${PPASS} -o ${CF_ORG} -s ${CF_SPACE}

          export PYTHONPATH="${JWORKSPACE}/python:\$PYTHONPATH"
          python "${JWORKSPACE}/python/cf_maint/src/maintain_cf.py" "${currEnv}"
        """
      }
      catch(err) {
        currentBuild.result = "FAILURE"
        throw err
      }
    }
  }
}