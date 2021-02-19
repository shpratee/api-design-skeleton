def m2settings = 'm2settings-uk-digital'
//def m2settings = 'm2settings-uk-gad-datastreaming'
def nexusreleaserepourl = "https://nexus-emea.1dc.com/repository/uk-digital-ipp-apidesignspecs/"
def REPO_ROOT = ''

def branch =''

def isHead = true

def nexusUrl = 'nexus-emea.1dc.com'
def nexusVersion = 'nexus3'
def protocol = 'https'
def credentialsId = 'USER_jenkins-nexus-emea'
def repository = 'uk-digital-ipp-apidesignspecs'
def groupId = 'com.fiserv.ipp.api.dp'
def yamlFile = 'ipp-documents-management-iapi.yaml'
def artifactId = 'ipp-documents-management-iapi-specs'
def artifactType = 'zip'
def ziparchive = "${artifactId}"
def spectralArtifact = "specs-rules"
def spectralArtifactVer = "1.0-1"
def nexusSpectralArtifactPath = "${nexusreleaserepourl}/com/fiserv/emea/ipp/${spectralArtifact}/${spectralArtifactVer}/${spectralArtifact}-${spectralArtifactVer}.zip"

pipeline {
	agent {
		label 'IPP'
	}
	tools {
		nodejs "node-8.9.4_npm-5.6.0"
		jdk "jdk1.8.0_201"
		maven "maven-3.0.5"
	}
	triggers {
      		pollSCM 'H/2 * * * *'
      	}
	options {
    		buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10'))
    		timestamps()
    		retry(1)
    		timeout time:30, unit:'MINUTES'
    		disableConcurrentBuilds()
    	}
	environment {
		BUILD_NUMBER = "${env.BUILD_NUMBER}"
		VERSIONFILE = 'version.properties'
		REPO_ROOT = ""
		nexus_Url = "${nexusUrl}"
		nexus_Version = "${nexusVersion}"
		protocol_ = "${protocol}"
		repository_ = "${repository}"
		credentials_Id = "${credentialsId}"
		group_Id = "${groupId}"
		artifact_Id = "${artifactId}"
		artifact_Type = "${artifactType}"
		BUILD_WORKDIR=pwd()
		currenthead=''
		expectedhead=''
	}
	stages {
		stage('Prepare git') {
			steps {
				script {
					sh "git log --oneline | head -2"
					// check for successful checkout of the needed commit
					currenthead = sh ( script: 'git rev-parse HEAD', returnStdout: true).trim()
					expectedhead = sh ( script: 'git rev-parse FETCH_HEAD', returnStdout: true).trim()

				}
			}
		}
		stage('Prepare Env') {
			steps {
				//  see global env on  https://jenkins-emea.1dc.com/pipeline-syntax/globals#env
				echo "start prepare env"
				echo "build number: ${env.BUILD_NUMBER}"			// from generell env
				echo "WORKSPACE: ${env.WORKSPACE}"
				echo "NODE_NAME: ${env.NODE_NAME}"
				echo "PATH: ${env.PATH}"
			}
		}
		stage('Check if latest HEAD') {
			steps {
				echo "check if commit is latest head"
				dir (REPO_ROOT){
					script {
					branch = getCurrentBranch();
						remoteurl = sh ( script: "git remote -v 2>/dev/null | head -1 | cut -f2 | cut -f1 -d ' '", returnStdout: true).trim()
						env.REMOTEURL = "${remoteurl}"
						remotehead = sh ( script: 'git ls-remote $REMOTEURL refs/heads/${branch} 2>/dev/null | cut -f1', returnStdout: true).trim()
						if (remotehead != currenthead){
							//isHead = false         // set marker
							echo "current commit ${currenthead} is not equal to remote HEAD  ${remotehead} of branch ${branch}, will skip the build"
						}
					}
				}
			}
		}
		stage('Install dependencies') {
			steps{
				sh 'npm config set metrics-registry "https://nexus-emea.1dc.com/repository/registry-yarnpkg-com"'
				sh 'npm config set registry  "https://nexus-emea.1dc.com/repository/registry-yarnpkg-com"'
				sh 'npm config set strict-ssl  false -g'
				sh 'npm config set always-auth true'
				sh 'npm install'
				sh 'npm install @stoplight/spectral@5.3.0'
				sh './node_modules/.bin/spectral --version'
			}
		}
		stage('Download Spectral rules From Repository'){
		    steps {
		    echo "Checkout the Spectral rules"
		    script {
            String urlAdminProject = "ssh://jenkins@gerrit-emea.1dc.com:29418/UK/Digital/IPP/APIDesignSpecs/Admin/spectral-rules.git"
					echo "Spectral Repos path :${urlAdminProject}"
					checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: urlAdminProject]]])
					sh 'ls -l'
				}
				}
		}
		stage('Validate Swagger Spec & Run Custom Rules') {
			steps {
				sh 'echo "Swagger spec validation ::" '
				script{
                    sh 'pwd'
					sh 'ls -l'
				   sh './node_modules/.bin/spectral lint -r specs-rules/spectral.yml '+yamlFile
			  }
			}
		}
		stage('Set unique version') {
			when {
				beforeAgent true
				expression { isHead ; isReleaseBranch() }
			}
			steps {
				dir (REPO_ROOT){
					script {

						echo "VERSIONFILE:: $WORKSPACE/$VERSIONFILE"
						configFileProvider(
								[configFile(fileId: 'setVersionProperties.sh', variable: 'NEXT_BUILD_VERSION')]) {

							env.VERSION = sh ( script: """sh $NEXT_BUILD_VERSION ${VERSIONFILE}""", returnStdout: true).trim()
							echo "next version:: ${env.VERSION}"
						}
					}
					script {
						echo "NEXT VERSION : ${env.VERSION}"
					}
				}
			}
		}
		stage('Create & Upload ApiDesignSpec Bundle to Nexus') {

			when {
				beforeAgent true
				allOf {
					expression { isHead ; isReleaseBranch()}
				}
			}
			steps {
				//
				echo "Upload ${ziparchive} package with version ${env.VERSION} "

				script {

					sh 'rm -rf node_modules/'
					sh 'echo branch-$branch'

					sh '''
        							cd ${WORKSPACE}
        							ls -ltr
        				'''
        			sh 'zip -r '+ziparchive+'.zip . -x *.git* *.idea* "*tmp*" "*specs-rules*" *.log *.properties *.json Jenkinsfile .'

					fileToUpload = "${WORKSPACE}/${ziparchive}.zip"
					echo "Workspace is compressed successfully::"+fileToUpload

					nexusArtifactUploader artifacts: [[artifactId: ziparchive, file: "${WORKSPACE}/${ziparchive}.zip", type: artifact_Type]],
							credentialsId: credentials_Id,
							groupId: group_Id, nexusUrl: nexus_Url, nexusVersion: nexus_Version, protocol: protocol_, repository: repository_, version: "${env.VERSION}"

					echo "uploaded to nexus"

					sh '''
                        echo "Removed the created the zip file ${WORKSPACE}/${ziparchive}.zip"
                        rm -rf ${WORKSPACE}/${ziparchive}.zip
                    '''
				}

			}
		}
		stage('set QG1 tag') {
			when {
				beforeAgent true
				allOf {
					expression { isHead; isReleaseBranch() }
				}
			}
			steps {
				//
				echo "set QG tag for ${env.VERSION} "
				script {
					// set QG with script
					configFileProvider(
							[configFile(fileId: 'setQG.sh', variable: 'SET_QG')]) {
						sh ( script: 'sh $SET_QG ${VERSION} ${WORKSPACE}/${REPO_ROOT}')
					}
					// show version
					currentBuild.displayName = "#" + currentBuild.getNumber() + "_" + "${env.VERSION}"
				}
				echo "pushed tag to gerrit "
			}
		}

		stage('annotate build') {
			steps {
				script {
					if (isHead){
						if (currentBuild.result == null || currentBuild.result == 'SUCCESS' || currentBuild.result == 'UNSTABLE'){

                             if(isReleaseBranch()){
                              currentBuild.displayName = "#" + currentBuild.getNumber()+ "_" + "${VERSION}"
                             }
                             else{
                             currentBuild.displayName = "#" + currentBuild.getNumber()
                             }

							addShortText (text: "${branch}", background: 'yellow')
						}
						else {
							addShortText (text: "${branch}", background: 'red')
						}
					}
					else {
						currentBuild.displayName = "#" + currentBuild.getNumber() + "_${currenthead}".substring(0,7) + "_Not_HEAD_no_Build"
						addShortText (text: "${branch}", background: 'orange')
					}
				}

			}
		}
	}
}

def getCurrentBranch () {
		return (env.GIT_BRANCH !=null )?env.GIT_BRANCH.minus("origin/"):''
	}

def isReleaseBranch(){
	boolean isReleaseBranch = false;
	currentBranch = getCurrentBranch()
	if (currentBranch.startsWith("release/")) {
		isReleaseBranch = true;
	}
	return isReleaseBranch;
}