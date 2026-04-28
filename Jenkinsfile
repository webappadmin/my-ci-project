pipeline {
    agent any

    // 🔥 ADD THIS TRIGGER FOR AUTOMATIC BUILDS
    triggers {
        // Trigger on push to main branch
        pollSCM('H/5 * * * *')  // Polls every 5 minutes as fallback
    }
    
    parameters {
        string(
            name: 'RESOURCE_GROUP_NAME',
            defaultValue: 'my-resource-group04282025',
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
        
        stage('Verify Azure CLI') {
            steps {
                script {
                    if (isUnix()) {
                        sh '''
                            # Check if Azure CLI is installed
                            if command -v az &> /dev/null; then
                                az --version
                                echo "✅ Azure CLI is already installed"
                            else
                                echo "❌ Azure CLI not found. Please install Azure CLI on the Jenkins agent VM"
                                echo "Run: curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash"
                                exit 1
                            fi
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
                            # Login using Azure VM Managed Identity (no secrets!)
                            az login --identity
                            
                            # Show current account info for verification
                            echo "✅ Successfully logged in with Managed Identity"
                            az account show --query "{Name:name, SubscriptionId:id}" --output table
                        '''
                    } else {
                        bat '''
                            az login --identity
                            echo "Successfully logged in with Managed Identity"
                            az account show --output table
                        '''
                    }
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
                            echo "=== Resource Group Details ==="
                            az group show --name ${rgName} --output table
                            echo ""
                            echo "=== Resource Group Tags ==="
                            az group show --name ${rgName} --query tags --output table
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
                    
                    // Get subscription info
                    def subscriptionInfo = sh(script: "az account show --query '{Name:name, ID:id}' --output json", returnStdout: true).trim()
                    
                    // Export to file for reference
                    writeFile file: 'resource-group-info.txt', text: """
                    Resource Group Details:
                    ======================
                    Name: ${rgName}
                    Location: ${params.LOCATION}
                    Environment: ${params.ENVIRONMENT}
                    Build Number: ${env.BUILD_NUMBER}
                    Created: ${new Date()}
                    Subscription Info: ${subscriptionInfo}
                    
                    Azure CLI Commands:
                    ------------------
                    az group show --name ${rgName}
                    az group delete --name ${rgName}
                    az group export --name ${rgName}
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
                
                📋 Useful Commands:
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
                1. Azure CLI is installed on Jenkins agent: 'az --version'
                2. VM Managed Identity is enabled: Check Azure Portal > VM > Identity
                3. Managed Identity has Contributor role on subscription
                4. Jenkins agent has network access to Azure
                
                Debug Commands:
                az login --identity --debug
                az account show
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