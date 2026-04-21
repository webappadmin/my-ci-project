# My CI/CD Project - Azure Resource Group Test

This project creates an Azure resource group using Jenkins with Managed Identity.

## Purpose

Test Azure Managed Identity authentication from Jenkins by creating a resource group.

## Prerequisites

* Jenkins with Azure Managed Identity configured
* GitHub repository with this code

## Pipeline Stages

1. Checkout code from GitHub
2. Login to Azure using Managed Identity
3. Create resource group
4. Verify resource group

## Manual Trigger

Build manually from Jenkins dashboard or push to GitHub triggers automatic build.

## Resource Group Details

* **Name**: my-test-resource-group
* **Location**: East US
* **Tags**: CreatedBy=Jenkins, Purpose=Test3

