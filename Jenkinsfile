library 'api_shared_library@autobahn-refactor'
library 'reference-pipeline'
library 'AppServiceAccount'

pipeline {
	agent any
	
	environment {
		APP_NAME = 'FLMAddress'
		MAVEN_SETTINGS = 'MFXR-MavenSettings'
		GIT_URL = 'https://gitlab.prod.fedex.com/APP3535896/mfxr-profiletoken.git'
		API_TOKEN_ID = 'MFXR-ProfileToken_API_Token'
		GITLAB_PROJECT_ID = '13437'
		PAM_ID = '117001'
		nexusCred_ID = 'MFXR_Nexus'
		RECIPIENTS = ''		
	}
	
	parameters {
		choice choices: ['default','development', 'integration', 'release', 'stress', 'staging'], description: '', name: 'LEVEL'
		choice choices: ['BOTH','EDCW','WTC'], description: '', name: 'DATA_CENTER'		
		booleanParam(name: 'FORTIFY_SCAN', description: 'Run Fortify Scan')
		booleanParam(name: 'NEXUSIQ_SCAN', description: 'Run NexusIQ Scan')
	}
	
	options {
		buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '10', daysToKeepStr: '', numToKeepStr: '10')
		//connection: gitLabConnection("GitLab-3537409-FLMAddress_ProfileToken")
		//gitlabBuilds(builds: ["Build", "SonarQube", "NexusIQ Analysis", "Fortify Analysis"])
	}

	tools{
        jdk 'JAVA_8'
    }

	stages{
		stage('Parameter Initialization'){
			steps{
				script{
					def paramInit = load "${WORKSPACE}/build/paramInit.groovy"
					paramInit(api_endPoint_dev_integration_release_stress: 'https://api.sys.wtcdev1.paas.fedex.com',
					api_endpoint_staging_production_edcw: 'https://api.sys.edccf2.paas.fedex.com',
					api_endpoint_staging_production_wtc: 'https://api.sys.wtccf2.paas.fedex.com')
					
					VersionRetrival(pom_path: './pom.xml',
						artifact_ext_type: 'jar')
				}
			}
		}
		
		stage('New Branch Validation'){
			options{
				skipDefaultCheckout true
			}
			steps {
				script {
					NewBranchValidation()
				}
			}
		}

		stage('CI/CD - DevDeploy/Release Level') {
			when{
				expression { params.LEVEL.equals('default') || params.LEVEL.equals('development') || params.LEVEL.equals('release') }
			}
			stages{
				stage("Build"){
					steps{
						//gitlabCommitStatus("Build"){
							script{
								MavenBuild(project_directory: '.',
								pom_path: './pom.xml',
								mavenSettingsConfig: env.MAVEN_SETTINGS,
								mavenTargets: 'clean install')
							}
						//}
					}
				}

				stage("Quality Checks"){
					failFast true
					parallel{
						stage("SonarQube Analysis"){
							steps{
								//gitlabCommitStatus("SonarQube"){
									script{
										sonarqube(projectKey: 'MFXRC-Profile-Token',
										projectName: 'MFXRC-Profile-Token',
										projectVersion: "${env.VERSION}-${env.BRANCH_NAME}",
										profile: 'CXS',
										jacocoPath: 'target/test-results/coverage/jacoco/jacoco.exec')
									}
								//}
							}
						}

						stage('NexusIQ Analysis'){
							when{
								expression { params.NEXUSIQ_SCAN==true || env.BRANCH_NAME.startsWith('RB-') }
							}
							steps{
								//gitlabCommitStatus("NexusIQ Analysis"){
									script{
										nexusIQAnalysis(iqAppName: 'MFXRC-Profile-Token-3535896')
									}
								//}
							}
						}

						stage('Fortify Analysis') {
							when{
								expression { params.FORTIFY_SCAN==true || env.BRANCH_NAME.startsWith('RB-') }
							}
							steps{
								//gitlabCommitStatus("Fortify Analysis"){
									script{
										Fortify(fortify_id: 'FLMAddress-3537409-Token')
									}
								//}
							}
						}
					}
				}					

				stage('Nexus Upload') {
					when{
						expression { BRANCH_NAME.startsWith('RB-') }
					}
					steps{
						script{
							NexusUpload(mavenSettingsConfig: env.MAVEN_SETTINGS)
						}
					}
				}
				
				stage("CD - Develop/Release Branch"){
					when{
						expression { (BRANCH_NAME.startsWith('RB-') || BRANCH_NAME.equals('develop')) && (params.LEVEL.equals('default') || params.LEVEL.equals('development') || params.LEVEL.equals('release')) }
					}
					stages{
						stage('PCF Deployment - EDCW') {
							when{
								expression { env.DEPLOY_TO_EDCW.equals('true') }
							}
							steps{
								script{
									PCFDeploy(pamID: env.PAM_ID,
										api_endpoint: env.EDCW_API_ENDPOINT,
										space: env.SPACE,
										Redux_id: env.REDUX_ID,
										ups_configfile_id: env.UPS_CONFIGFILE_ID,
										AppD_plan: env.APPD_PLAN,
										data_tag: env.EDCW_DATA_TAG,
										manifestFile: env.MANIFEST_FILE_EDCW)
								}
							}
						}

						stage('PCF Deployment - WTC') {
							when{
								expression { env.DEPLOY_TO_WTC.equals('true') }
							}
							steps{
								script{
									PCFDeploy(pamID: env.PAM_ID,
										api_endpoint: env.WTC_API_ENDPOINT,
										space: env.SPACE,
										Redux_id: env.REDUX_ID,
										ups_configfile_id: env.UPS_CONFIGFILE_ID,
										AppD_plan: env.APPD_PLAN,
										data_tag: env.WTC_DATA_TAG,
										manifestFile: env.MANIFEST_FILE_WTC)
								}
							}
						}
					}
				}
			}
		}
		
		stage('CD - QADeploy & other levels') {
			when{
				expression { BRANCH_NAME.startsWith('RB-') && !(params.LEVEL.equals('default') || params.LEVEL.equals('development') || params.LEVEL.equals('release'))}
			}
			stages{
				stage('Nexus Download'){
					steps{
						script{
							VersionRetrival(pom_path: './pom.xml',
								artifact_ext_type: 'jar')
						
							NexusDownload()
						}
					}
				}

				stage('PCF Deployment - EDCW') {
					when{
						expression { env.DEPLOY_TO_EDCW.equals('true') }
					}
					steps{
						script{
							PCFDeploy(pamID: env.PAM_ID,
								api_endpoint: env.EDCW_API_ENDPOINT,
								space: env.SPACE,
								Redux_id: env.REDUX_ID,
								ups_configfile_id: env.UPS_CONFIGFILE_ID,
								AppD_plan: env.APPD_PLAN,
								data_tag: env.EDCW_DATA_TAG,
								manifestFile: env.MANIFEST_FILE_EDCW)
						}
					}
				}

				stage('PCF Deployment - WTC') {
					when{
						expression { env.DEPLOY_TO_WTC.equals('true') }
					}
					steps{
						script{
							PCFDeploy(pamID: env.PAM_ID,
								api_endpoint: env.WTC_API_ENDPOINT,
								space: env.SPACE,
								Redux_id: env.REDUX_ID,
								ups_configfile_id: env.UPS_CONFIGFILE_ID,
								AppD_plan: env.APPD_PLAN,
								data_tag: env.WTC_DATA_TAG,
								manifestFile: env.MANIFEST_FILE_WTC)
						}
					}
				}
			}
		}
	}
	post{
        always{            
            cleanWs()
        }
    }
}
