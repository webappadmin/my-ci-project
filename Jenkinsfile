pipeline {
    agent any
    
    environment {
        // Dynamic resource group name with branch and build number
        RESOURCE_GROUP_NAME = "rg-${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
        LOCATION = 'eastus'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "📦 Checking out code from GitHub..."
                echo "Current branch: ${env.BRANCH_NAME}"
                echo "Build number: ${env.BUILD_NUMBER}"
                checkout scm
            }
        }
        
        stage('Login to Azure using Managed Identity') {
            steps {
                echo "🔐 Logging into Azure with Managed Identity..."
                script {
                    try {
                        sh 'az login --identity'
                        echo "✅ Successfully logged into Azure"
                        sh 'az account show --output table'
                    } catch (Exception e) {
                        error "❌ Failed to login with Managed Identity: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Create Resource Group') {
            steps {
                echo "🏗️ Creating resource group: ${RESOURCE_GROUP_NAME}"
                script {
                    // Get timestamp using sh and store in variable
                    def timestamp = sh(script: "date +%Y-%m-%d-%H-%M-%S", returnStdout: true).trim()
                    
                    sh """
                        az group create \
                            --name ${RESOURCE_GROUP_NAME} \
                            --location ${LOCATION} \
                            --tags "CreatedBy=Jenkins" "Branch=${env.BRANCH_NAME}" "BuildNumber=${env.BUILD_NUMBER}" "Timestamp=${timestamp}"
                    """
                }
                echo "✅ Resource group created successfully!"
            }
        }
        
        stage('Verify Resource Group') {
            steps {
                echo "🔍 Verifying resource group..."
                sh """
                    az group show \
                        --name ${RESOURCE_GROUP_NAME} \
                        --output table
                """
            }
        }
        
        stage('Resource Group Details') {
            steps {
                echo "📋 Resource Group Details:"
                script {
                    def subscriptionName = sh(script: "az account show --query name -o tsv", returnStdout: true).trim()
                    sh """
                        echo "Resource Group Name: ${RESOURCE_GROUP_NAME}"
                        echo "Location: ${LOCATION}"
                        echo "Subscription: ${subscriptionName}"
                        echo "Tags:"
                        az group show --name ${RESOURCE_GROUP_NAME} --query tags -o table
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo """
            ┌─────────────────────────────────────────────────────────┐
            │  🎉 PIPELINE SUCCESSFUL                                 │
            ├─────────────────────────────────────────────────────────┤
            │  Resource Group: ${env.RESOURCE_GROUP_NAME}            │
            │  Branch:         ${env.BRANCH_NAME}                    │
            │  Build Number:   ${env.BUILD_NUMBER}                   │
            │  Location:       ${env.LOCATION}                       │
            └─────────────────────────────────────────────────────────┘
            """
        }
        failure {
            echo """
            ┌─────────────────────────────────────────────────────────┐
            │  ❌ PIPELINE FAILED                                     │
            ├─────────────────────────────────────────────────────────┤
            │  Branch: ${env.BRANCH_NAME}                            │
            │  Build Number: ${env.BUILD_NUMBER}                     │
            │  Check console output for details                      │
            └─────────────────────────────────────────────────────────┘
            """
        }
    }
}