pipeline {
    agent any
    
    environment {
        // Azure configuration - No secrets needed!
        RESOURCE_GROUP_NAME = 'my-test-resource-group-2'
        LOCATION = 'eastus'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "📦 Checking out code from GitHub..."
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
                    } catch (Exception e) {
                        error "❌ Failed to login with Managed Identity: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Create Resource Group') {
            steps {
                echo "🏗️ Creating resource group: ${RESOURCE_GROUP_NAME}"
                sh """
                    az group create \
                        --name ${RESOURCE_GROUP_NAME} \
                        --location ${LOCATION} \
                        --tags "CreatedBy=Jenkins" "Purpose=Test"
                """
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
    }
    
    post {
        success {
            echo "🎉 Pipeline completed! Resource group '${RESOURCE_GROUP_NAME}' is ready."
        }
        failure {
            echo "❌ Pipeline failed. Check the logs above."
        }
    }
}