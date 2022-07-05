
# Table of Contents

1.  [How to Cross Account between two AWS-Account](#org6d7960b)
    1.  [Intro](#org8d32edc)
    2.  [Requirements :](#orge6c200f)
    3.  [Create a Pipeline](#orgca34cfd)


<a id="org6d7960b"></a>

# How to Cross Account between two AWS-Account


<a id="org8d32edc"></a>

## Intro

Lets consider you have two aws accounts

-   Infrastructure Account
    -   Consist of all resources it does n't run any task,project but manages
    -   All developer will push code in this account
-   Production Account
    -   The task like build, deploy should be done in the Production there is no data(source code) link between two account

[GOAL]: To create a pipeline in Infrastructure Account and the build, deploy should be done in Production, Let make it simple

-   Goal is create Pipeline in Infrastructure Account and Deploy in Production Account


<a id="orge6c200f"></a>

## Requirements :

-   Requirements
    -   Infrastructure Account Id : << Account-A >> 918175365727
    -   Production Account Id : << Account-B>> 548593215839
    -   In Infrastructure Account(Account-A)
        -   [Role] :  **CrossAccount-Role-of-A**
            -   Trust relationship
                ``` json
                
                    {
                        "Version": "2012-10-17",
                        "Statement": [
                    	{
                    	    "Effect": "Allow",
                    	    "Principal": {
                    		"Service": "codepipeline.amazonaws.com"
                    	    },
                    	    "Action": "sts:AssumeRole"
                    	}
                        ]
                    }
                ```
            -   Role : Own AWS Account
            -   Policy:
            -   S3FullAccess
            -   CodeCommitFullAccess
            -   StsAssumeRole (In Prod-Account):
                [NOTE:] This  Role allow to communicate with Resource (roles) in other Account (account-B)

                ``` json
                    {
                        "Version": "2012-10-17",
                        "Statement": {
                    	"Effect": "Allow",
                    	"Action": "sts:AssumeRole",
                    	"Resource": [
                    	    "arn:aws:iam::548593215839:role/*"
                    	]
                        }
                    }

                ```
        -   [KMS] **Custom Manage Keys**
            -   User : root,user1
            -   Permission: CrossAccountRole-of-A
            -   Access to Other AWS Account : Account-B
        -   [S3] **artifact-source** Share BuildArtifacts,Keys, Using S3 Buckets
            -   Permission : - Get, Put data in S3 Bucket by Account- B (548593215839)
            -   Read Listbucket by Account-B   (548593215839)
                ``` json                
                    {
                        "Version": "2012-10-17",
                        "Id": "Policy1553183091390",
                        "Statement": [
                    	{
                    	    "Sid": "",
                    	    "Effect": "Allow",
                    	    "Principal": {
                    		"AWS": "arn:aws:iam::548593215839:root"
                    	    },
                    	    "Action": [
                    		"s3:Get*",
                    		"s3:Put*"
                    	    ],
                    	    "Resource": "arn:aws:s3:::artifact-source/*"
                    	},
                    	{
                    	    "Sid": "",
                    	    "Effect": "Allow",
                    	    "Principal": {
                    		"AWS": "arn:aws:iam::548593215839:root"
                    	    },
                    	    "Action": "s3:ListBucket",
                    	    "Resource": "arn:aws:s3:::artifact-source"
                    	}
                        ]
                    }
                ```
    -   In Production Account (Account-B)
        -   [Policies ]:
            -   CrossAccount-S3-Access-Policy (To Share Pipeline-BuildArtifact Bucket to share data include (keys, buildartifacts&#x2026;etc)
                ``` json
                    {
                        "Version": "2012-10-17",
                        "Statement": [
                    	{
                    	    "Effect": "Allow",
                    	    "Action": [
                    		"s3:Get*",
                    		"s3:Put*",
                    		"s3:ListBucket"
                    	    ],
                    	    "Resource": [
                    		"arn:aws:s3:::artifact-source/*"
                    	    ]
                    	}
                        ]
                    }
                ```
            -   CrossAccount-IAM-Role-PassRole Policy : (Cloudformation) : Allow to pass the (Cloudformation)role form one account to other
                ``` json                
                    {
                        "Version": "2012-10-17",
                        "Statement": [
                    	{
                    	    "Effect": "Allow",
                    	    "Action": [
                    		"cloudformation:*",
                    		"iam:PassRole"
                    	    ],
                    	    "Resource": "*"
                    	}
                        ]
                    }
                ```
            -   CrossAccount-KMS-Key-Access Policy  :
                Allow to Encrypt,Decrpyt,GenerateDatakey,Describekey for secure transmission and storage of data 
                ``` json                
                    {
                        "Version": "2012-10-17",
                        "Statement": [
                    	{
                    	    "Effect": "Allow",
                    	    "Action": [
                    		"kms:DescribeKey",
                    		"kms:GenerateDataKey*",
                    		"kms:Encrypt",
                    		"kms:ReEncrypt*",
                    		"kms:Decrypt"
                    	    ],
                    	    "Resource": [
                    		"arn:aws:kms:eu-west-1:918175365727:key/7bc79cce-3b1a-4a61-862e-747990aa1173"
                    	    ]
                    	}
                        ]
                    }

                ```
        -   [Role]:  **CrossAccount-Role-of-B**
            -   Access to other AWS Account : **Account-A**
            -   Policies :
                -   **CrossAccount-S3-Access Policy** : To share  Pipeline-BuildArtifact, share data(key&#x2026;etc), communicate with other roles,
                -   **CrossAccount-IAM-Role-PassRole Policy** :  Allow to pass CloudFormation Role to Account-B
                -   **CrossAccount-KMS-Key-Access Policy** :  Allow to Encrypt, Depcrpyt, Generatedatakey
        -   [Role]: **CrossAccount-RunBlock-Role-CloudformationExecutionRole** :
            Allow to run Block of pipeline in Account B
            -   **CloudFormationExecutionRole** : Need to root permission to create Infrastructures
                Policy : AdministratorAccess


<a id="orgca34cfd"></a>

## Create a Pipeline

-   Create a Pipeline in Infrastructure Account and Run Cloudformation in Production Account
    
    Steps to create Pipeline

    ``` yaml
    
        - Pipeline:
            Description:
              Name:
        	  RoleName: *cross-account-role-A*
        	  BuildArtifact location : *artifact-source*
        	  Encryptionkey: *Cross-account-key*
        	    Type: KMS
            Stages:
              Stage :
        	    Name: Source
        	    RepositoryName:
        	    BranchName:
              Stage:
        	    Name: Deploy
        	    DeployType: CloudFormation
        	    Action : Create and Update
        	    Role: *CrossAccount-BlockRun-Role-CloudformationExecutionRole* in Account B
        	    StackName:
        	    TemplatePath": "SourceArtifact::aws-s3-cf.yaml
        	  Role: *CrossAccount-Role-B*
     ```
        
Above Pipline will give error so we need to get the pipeline json file and edit and update it to aws

We can get the pipeline json file by

    ``` bash

    # To get the list of pipeline running in give account, given region 
    aws codepipeline list-pipelines --region us-east-1 --profile dan2505
    
    # To get the pipeline json file
    aws codepipeline get-pipeline --region eu-west-1 --name Cross-Account-CloudFormation-CICD --profile dan2505 > failed-cross-pipeline.json

    ```

Change your json file as follow

    ``` json

    {
        "pipeline": {
    	"name": "Cross-Account-CloudFormation-CICD",
    	"roleArn": "arn:aws:iam::918175365727:role/cross-account-role-A",
    	"artifactStore": {
    	    "type": "S3",
    	    "location": "artifact-source",
    	    "encryptionKey": {
    	      "id": "arn:aws:kms:eu-west-1:918175365727:key/7bc79cce-3b1a-4a61-862e-747990aa1173",
    		"type": "KMS"
    	    }
    	},
    	"stages": [
    	    {
    		"name": "Source",
    		"actions": [
    		    {
    			"name": "Source",
    			"actionTypeId": {
    			    "category": "Source",
    			    "owner": "AWS",
    			    "provider": "CodeCommit",
    			    "version": "1"
    			},
    			"runOrder": 1,
    			"configuration": {
    			    "BranchName": "master",
    			    "OutputArtifactFormat": "CODE_ZIP",
    			    "PollForSourceChanges": "false",
    			    "RepositoryName": "Cross-Account-CF"
    			},
    			"outputArtifacts": [
    			    {
    				"name": "SourceArtifact"
    			    }
    			],
    			"inputArtifacts": [],
    			"region": "eu-west-1",
    			"namespace": "SourceVariables"
    		    }
    		]
    	    },
    	    {
    		"name": "Deploy",
    		"actions": [
    		    {
    			"name": "Deploy",
    			"actionTypeId": {
    			    "category": "Deploy",
    			    "owner": "AWS",
    			    "provider": "CloudFormation",
    			    "version": "1"
    			},
    			"runOrder": 1,
    			"configuration": {
    			    "ActionMode": "CREATE_UPDATE",
    			    "RoleArn": "arn:aws:iam::548593215839:role/CloudformationExecutionRole",
    			    "StackName": "Cross-Account-CloudFormation-CICD",
    			    "TemplatePath": "SourceArtifact::aws-s3-cf.yaml"
    			},
    			"outputArtifacts": [],
    			"inputArtifacts": [
    			    {
    				"name": "SourceArtifact"
    			    }
    			],
    			"roleArn": "arn:aws:iam::548593215839:role/cross-account-role-B",
    			"region": "eu-west-1",
    			"namespace": "DeployVariables"
    		    }
    		]
    	    }
    	],
    	"version": 2
        }
    }

    ```
After editing the pipeline file update by aws-cli cmd

    ``` sh

    aws codepipeline update-pipeline --cli-input-json file://failed-cross-pipeline.json --profile dan2505

    ```


[NOTE]: This cmd is not working in Ubuntu but working in windows

