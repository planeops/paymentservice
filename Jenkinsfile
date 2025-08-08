@Library('jenkins-shared-lib@main') _

pipeline {
	agent {
		kubernetes {
			label 'nodejs'
            defaultContainer 'node'
            yaml libraryResource('pod-templates/nodejs-buildkit.yaml')
        }
    }

	parameters {
		string(name: 'ORG_NAME', defaultValue: 'DevSecOps-homelab', description: 'GitHub organization or user')
		string(name: 'SERVICE_NAME', defaultValue: 'paymentservice', description: 'Name of the service')
		string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git branch to build')
		string(name: 'IMAGE_TAG', defaultValue: '', description: 'Docker image tag (optional, default = commit SHA)')
	}

	environment {
		GIT_REPO_URL = "https://github.com/${params.ORG_NAME}/${params.SERVICE_NAME}"
		IMAGE_NAME = "${params.SERVICE_NAME}"
		TAG = "${params.IMAGE_TAG ?: env.GIT_COMMIT.take(7)}"
		FULL_IMAGE = "${env.REGISTRY}/${env.IMAGE_NAME}:${env.TAG}"
		GIT_CREDENTIALS_ID = "github-access-token"
		BUILDKIT_ADDR = 'tcp://buildkit-service.buildkit.svc.cluster.local:1234'
		S3_ENDPOINT = 'http://minio.minio.svc.cluster.local:9000'
		REGISTRY = 'harbor-harbor-core.harbor.svc.cluster.local/online-boutique'
	}

    stages {
		stage('Git Checkout') {
			steps {
				script {
					git.checkout(
						params.GIT_BRANCH,
						env.GIT_REPO_URL,
						env.GIT_CREDENTIALS_ID
					)
				}
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
					script {
						gitleaks.scan()
					}
				}
			}
		}
		stage('SonarQube Scan') {
			steps {
				container('sonar') {
					script {
						sonarqubeNode.scan(params.SERVICE_NAME)
					}
				}
			}
		}
		stage('SonarQube Quality Gate') {
			steps {
				script {
					sonarqubeNode.qualityGate()
				}
			}
		}
		stage('Trivy FS Scan') {
			steps {
				container('trivy') {
					script {
						trivy.filesystemScan()
					}
				}
			archiveArtifacts artifacts: 'fs-report.txt', allowEmptyArchive: true
			}
		}
		stage('Build image') {
			steps {
				container('buildkit') {
					script {
						buildctl.build(
							env.BUILDKIT_ADDR,
							env.S3_ENDPOINT,
							params.SERVICE_NAME
						)
					}
				}
			}
		}
		stage('Trivy Image Scan') {
			steps {
				container('trivy') {
					script {
						trivy.imageScan()
					}
				}
			archiveArtifacts artifacts: 'image-report.txt', allowEmptyArchive: true
			}
		}
		stage('Push image') {
			steps {
				container('buildkit') {
					script {
						buildctl.push(
							env.BUILDKIT_ADDR,
							env.S3_ENDPOINT,
							params.SERVICE_NAME,
							env.FULL_IMAGE
						)
					}
				}
			}
		}
    }
}