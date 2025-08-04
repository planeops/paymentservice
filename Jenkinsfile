@Library('jenkins-shared-lib@main') _

pipeline {
	agent {
		kubernetes {
			label 'nodejs'
            defaultContainer 'node'
            yaml libraryResource('pod-templates/nodejs.yaml')
        }
    }

	parameters {
	string(name: 'ORG_NAME', defaultValue: 'DevSecOps-homelab', description: 'GitHub organization or user ')
	string(name: 'SERVICE_NAME', defaultValue: 'paymentservice', description: 'Name of the service')
	string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git branch to build')
	string(name: 'IMAGE_TAG', defaultValue: '', description: 'Docker image tag (optional, default = commit SHA)')
	}

	environment {
		REGISTRY = 'docker.io/volkamana'
		GIT_REPO_URL = "https://github.com/${params.ORG_NAME}/${params.SERVICE_NAME}"
		IMAGE_NAME = "${params.SERVICE_NAME}"
		TAG = "${params.IMAGE_TAG ?: env.GIT_COMMIT.take(7)}"
		FULL_IMAGE = "${env.REGISTRY}/${env.IMAGE_NAME}:${env.TAG}"
		GIT_CREDENTIALS_ID = "github-access-token"
	}

    stages {
		stage('Git Chechout') {
			steps {
				checkout([
					$class: 'GitSCM',
					branches: [[name: "*/${params.GIT_BRANCH}"]],
					userRemoteConfigs: [[
						url: env.GIT_REPO_URL,
						credentialsId: env.GIT_CREDENTIALS_ID
					]]
				])
            }
        }
		stage('Compilation') {
			steps {
				dir('client') {
					 sh 'find . -name "*.js" -exec node --check {} +'
				 }
			 }
		}
		stage('Gitleaks Scan') {
			 steps {
				 container('gitleaks') {
				   sh 'gitleaks detect --source . --exit-code 1'
				 }
			 }
		}
		stage('SonarQube Scan') {
			steps {
				container('sonar') {
					withSonarQubeEnv('SonarQube') {
						sh """
						sonar-scanner \
						-Dsonar.projectName=${params.SERVICE_NAME} \
						-Dsonar.projectKey=${params.SERVICE_NAME}
						"""
					}
				}
			}
		}
		stage('SonarQube Quality Gate') {
			steps {
				timeout(time: 1, unit: 'HOURS') {
					waitForQualityGate abortPipeline: true
				}
			}
		}
		stage('Trivy FS Scan') {
			steps {
				container('trivy') {
					sh 'trivy fs --format table -o fs-report.txt .'
				}
				archiveArtifacts artifacts: 'fs-report.txt', allowEmptyArchive: true
			}
		}
		stage('Build API image') {
			steps {
				container('kaniko') {
					sh """
						/kaniko/executor \
						--context `pwd` \
						--dockerfile `pwd`/Dockerfile \
						--destination=${env.FULL_IMAGE} \
						--cleanup \
						--verbosity=info
					"""
				}
			}
		}
		stage('Scan pushed image') {
			steps {
				container('trivy') {
					sh "trivy image --format table -o image-report.txt ${env.FULL_IMAGE}"
				}
				archiveArtifacts artifacts: 'image-report.txt', allowEmptyArchive: true
			}
		}
    }
}