pipeline {
    agent any
    
    parameters {
        string(
            name: 'RESOURCE_GROUP_NAME',
            defaultValue: 'my-resource-group-04272026',
            description: 'Name of the Azure resource group'
        )
        string(
            name: 'LOCATION',
            defaultValue: 'eastus',
            description: 'Azure region (eastus, westeurope, centralus, etc.)'
        )
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'test', 'prod'],
            description: 'Environment type'
        )
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "Building on branch: ${env.BRANCH_NAME}"
            }
        }
        
        stage('Install Azure CLI') {
            steps {
                script {
                    if (isUnix()) {
                        sh '''
                            # Install Azure CLI if not present
                            if ! command -v az &> /dev/null; then
                                echo "Installing Azure CLI..."
                                curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
                            fi
                            az --version
                        '''
                    } else {
                        bat 'az --version'
                    }
                }
            }
        }
        
        stage('Login with Managed Identity') {
            steps {
                script {
                    if (isUnix()) {
                        sh '''
                            # Login using Azure VM Managed Identity
                            az login --identity
                            
                            # Show current account info for verification
                            az account show --output table
                        '''
                    } else {
                        bat '''
                            az login --identity
                            az account show --output table
                        '''
                    }
                    echo "Successfully logged in with Managed Identity"
                }
            }
        }
        
        stage('Create Resource Group') {
            steps {
                script {
                    // Generate unique resource group name with timestamp
                    def timestamp = new Date().format('yyyyMMdd-HHmmss')
                    def rgName = "${params.RESOURCE_GROUP_NAME}-${params.ENVIRONMENT}-${timestamp}"
                    def location = params.LOCATION
                    
                    echo "Creating resource group: ${rgName}"
                    echo "Location: ${location}"
                    echo "Environment: ${params.ENVIRONMENT}"
                    
                    if (isUnix()) {
                        sh """
                            az group create \
                                --name ${rgName} \
                                --location ${location} \
                                --tags \
                                    Environment=${params.ENVIRONMENT} \
                                    CreatedBy=Jenkins \
                                    BuildNumber=${env.BUILD_NUMBER} \
                                    ManagedIdentity=true
                        """
                    } else {
                        bat """
                            az group create ^
                                --name ${rgName} ^
                                --location ${location} ^
                                --tags Environment=${params.ENVIRONMENT} CreatedBy=Jenkins BuildNumber=${env.BUILD_NUMBER} ManagedIdentity=true
                        """
                    }
                    
                    // Save resource group name for later stages
                    env.RESOURCE_GROUP_CREATED = rgName
                    echo "✅ Resource group '${rgName}' created successfully"
                }
            }
        }
        
        stage('Verify Resource Group') {
            steps {
                script {
                    def rgName = env.RESOURCE_GROUP_CREATED
                    
                    echo "Verifying resource group: ${rgName}"
                    
                    if (isUnix()) {
                        sh """
                            az group show --name ${rgName} --output table
                            az group list --query "[?name=='${rgName}']" --output table
                        """
                    } else {
                        bat """
                            az group show --name ${rgName} --output table
                        """
                    }
                }
            }
        }
        
        stage('Export Resource Group Info') {
            steps {
                script {
                    def rgName = env.RESOURCE_GROUP_CREATED
                    
                    // Export to file for reference
                    writeFile file: 'resource-group-info.txt', text: """
                    Resource Group Details:
                    ======================
                    Name: ${rgName}
                    Location: ${params.LOCATION}
                    Environment: ${params.ENVIRONMENT}
                    Build Number: ${env.BUILD_NUMBER}
                    Created: ${new Date()}
                    Azure Subscription: ${getSubscriptionId()}
                    """
                    
                    archiveArtifacts artifacts: 'resource-group-info.txt', fingerprint: true
                }
            }
        }
    }
    
    post {
        success {
            script {
                echo """
                ┌─────────────────────────────────────────┐
                │  ✅ RESOURCE GROUP CREATED SUCCESSFULLY  │
                └─────────────────────────────────────────┘
                
                Resource Group: ${env.RESOURCE_GROUP_CREATED}
                Location: ${params.LOCATION}
                Environment: ${params.ENVIRONMENT}
                
                Azure CLI commands to manage:
                az group show --name ${env.RESOURCE_GROUP_CREATED}
                az group delete --name ${env.RESOURCE_GROUP_CREATED}
                """
            }
        }
        
        failure {
            script {
                error """
                ┌─────────────────────────────────────┐
                │  ❌ RESOURCE GROUP CREATION FAILED  │
                └─────────────────────────────────────┘
                
                Please check:
                1. VM Managed Identity is enabled
                2. Managed Identity has Contributor role on subscription
                3. Azure CLI version is up to date
                4. Network connectivity to Azure
                """
            }
        }
        
        always {
            echo """
            Pipeline execution completed at: ${new Date()}
            Status: ${currentBuild.currentResult}
            """
        }
    }
}

// Helper function to check Unix/Linux environment
def isUnix() {
    return !env.OS || env.OS.toLowerCase().contains('linux') || env.OS.toLowerCase().contains('mac')
}

// Helper function to get subscription ID
def getSubscriptionId() {
    try {
        if (isUnix()) {
            def subId = sh(script: 'az account show --query id -o tsv', returnStdout: true).trim()
            return subId
        }
    } catch (Exception e) {
        return "Unable to fetch subscription ID"
    }
    return "Unknown"
}