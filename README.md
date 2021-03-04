# awsrdstest
from medium.com saving money on RDS. 
RDS test

aws rds create-db-instance --db-instance-identifier testdb --db-instance-class db.t2.micro --engine mysql --allocated-storage 20 --master-username admin --master-user-password adminPwd

Response: 
{
    "DBInstance": {
        "DBInstanceIdentifier": "testdb",
        "DBInstanceClass": "db.t2.micro",
        "Engine": "mysql",
        "DBInstanceStatus": "creating",
        "MasterUsername": "admin",
        "AllocatedStorage": 20,
        "PreferredBackupWindow": "07:09-07:39",
        "BackupRetentionPeriod": 1,
        "DBSecurityGroups": [],
        "VpcSecurityGroups": [
            {
                "VpcSecurityGroupId": "sg-7684e91b",
                "Status": "active"
            }
        ],
        "DBParameterGroups": [
            {
                "DBParameterGroupName": "default.mysql8.0",
                "ParameterApplyStatus": "in-sync"
            }
        ],
        "DBSubnetGroup": {
            "DBSubnetGroupName": "default",
            "DBSubnetGroupDescription": "default",
            "VpcId": "vpc-371a755f",
            "SubnetGroupStatus": "Complete",
            "Subnets": [
                {
                    "SubnetIdentifier": "subnet-74fe570e",
                    "SubnetAvailabilityZone": {
                        "Name": "ca-central-1b"
                    },
                    "SubnetOutpost": {},
                    "SubnetStatus": "Active"
                },
                {
                    "SubnetIdentifier": "subnet-4548d52d",
                    "SubnetAvailabilityZone": {
                        "Name": "ca-central-1a"
                    },
                    "SubnetOutpost": {},
                    "SubnetStatus": "Active"
                },
                {
                    "SubnetIdentifier": "subnet-b25d61ee",
                    "SubnetAvailabilityZone": {
                        "Name": "ca-central-1d"
                    },
                    "SubnetOutpost": {},
                    "SubnetStatus": "Active"
                }
            ]
        },
        "PreferredMaintenanceWindow": "fri:05:26-fri:05:56",
        "PendingModifiedValues": {
            "MasterUserPassword": "****"
        },
        "MultiAZ": false,
        "EngineVersion": "8.0.20",
        "AutoMinorVersionUpgrade": true,
        "ReadReplicaDBInstanceIdentifiers": [],
        "LicenseModel": "general-public-license",
        "OptionGroupMemberships": [
            {
                "OptionGroupName": "default:mysql-8-0",
                "Status": "in-sync"
            }
        ],
        "PubliclyAccessible": true,
        "StorageType": "gp2",
        "DbInstancePort": 0,
        "StorageEncrypted": false,
        "DbiResourceId": "db-DH4TPTL43R7AU7K3ZAW6BRA6EM",
        "CACertificateIdentifier": "rds-ca-2019",
        "DomainMemberships": [],
        "CopyTagsToSnapshot": false,
        "MonitoringInterval": 0,
        "DBInstanceArn": "arn:aws:rds:ca-central-1:254632925375:db:testdb",
        "IAMDatabaseAuthenticationEnabled": false,
        "PerformanceInsightsEnabled": false,
        "DeletionProtection": false,
        "AssociatedRoles": [],
        "TagList": [],
        "CustomerOwnedIpEnabled": false
    }
}



aws s3 mb s3://ahc-auto-aws-db-shutdown-deployment-artifacts 
Response:
make_bucket: ahc-auto-aws-db-shutdown-deployment-artifacts

ngraham:~/environment $ aws s3 ls 
2021-03-03 15:58:29 ahc-auto-aws-db-shutdown-deployment-artifacts
2020-05-25 15:08:15 ahc-billing-reporting
2019-12-12 19:52:18 ahc-scalableec2-terraform-state
2021-03-02 02:10:19 aws-sam-cli-managed-default-samclisourcebucket-g38a8zguhg6e

Create the template.yaml file for the lambda:


Create the node.js scripts to start up and shutdown the dbs (see app.js/start.js/stop.js)
Update the cfn template.yaml file to create the lambdas

Test locally using SAM

sam local invoke --no-event DBShutDownFunction 

   (invokes the DBShutDownFunction defined in the template, which invokes the lambda handler found in shut-down/app.js (which in turn calls the function in stop.js)
   
   You should get a 200 response with the db details

Note that it can take a loooong time (hours) to shutdown the DB as it automatically takes a backup before stopping (this is AWS behaviour).  

Once stopped, start it using 

sam local invoke --no-event DBStartUpFunction
    You should get a 200 response with the db details
    

To deploy the functions to lambda, 

1. package them up first.

sam package --template-file template.yaml --s3-bucket ahc-auto-aws-db-shutdown-deployment-artifacts --output-template-file outputTemplate.yaml 

This should upload the artifcats (zip files) to the s3 bucket and create an outputTemplate.yaml file that containst the URI in s3 of where the lambda artifacts can be found for the deploy.

ngraham:~/environment/awsrdstest (main) $ aws s3 ls s3://ahc-auto-aws-db-shutdown-deployment-artifacts
2021-03-03 19:50:44        593 a0c6e1451e42cc1ff22714bb83c7f4bd
2021-03-03 19:50:44        594 e29165475c85cc55e1d17037715cdeea

2. deploy the lambdas

sam deploy --template-file outputTemplate.yaml --stack-name auto-db-shutdown-stack --capabilities CAPABILITY_IAM

The CAPABILITY_IAM allows for the roles needed to be created for the lambdas

ngraham:~/environment/awsrdstest (main) $ sam deploy --template-file outputTemplate.yaml --stack-name auto-db-shutdown-stack --capabilities CAPABILITY_IAM

        Deploying with following values
        ===============================
        Stack name                   : auto-db-shutdown-stack
        Region                       : None
        Confirm changeset            : False
        Deployment s3 bucket         : None
        Capabilities                 : ["CAPABILITY_IAM"]
        Parameter overrides          : {}
        Signing Profiles             : {}

Initiating deployment
=====================

Waiting for changeset to be created..

CloudFormation stack changeset
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Operation                                                    LogicalResourceId                                            ResourceType                                                 Replacement                                                
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+ Add                                                        DBShutDownFunctionRole                                       AWS::IAM::Role                                               N/A                                                        
+ Add                                                        DBShutDownFunction                                           AWS::Lambda::Function                                        N/A                                                        
+ Add                                                        DBStartUpFunctionRole                                        AWS::IAM::Role                                               N/A                                                        
+ Add                                                        DBStartUpFunction                                            AWS::Lambda::Function                                        N/A                                                        
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Changeset created successfully. arn:aws:cloudformation:ca-central-1:254632925375:changeSet/samcli-deploy1614801389/8ad0e7cf-e4e2-4e43-8421-9908139f62f2


2021-03-03 19:56:34 - Waiting for stack create/update to complete

CloudFormation events from changeset
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ResourceStatus                                               ResourceType                                                 LogicalResourceId                                            ResourceStatusReason                                       
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
CREATE_IN_PROGRESS                                           AWS::IAM::Role                                               DBStartUpFunctionRole                                        Resource creation Initiated                                
CREATE_IN_PROGRESS                                           AWS::IAM::Role                                               DBShutDownFunctionRole                                       Resource creation Initiated                                
CREATE_IN_PROGRESS                                           AWS::IAM::Role                                               DBStartUpFunctionRole                                        -                                                          
CREATE_IN_PROGRESS                                           AWS::IAM::Role                                               DBShutDownFunctionRole                                       -                                                          
CREATE_COMPLETE                                              AWS::IAM::Role                                               DBShutDownFunctionRole                                       -                                                          
CREATE_COMPLETE                                              AWS::IAM::Role                                               DBStartUpFunctionRole                                        -                                                          
CREATE_IN_PROGRESS                                           AWS::Lambda::Function                                        DBShutDownFunction                                           -                                                          
CREATE_IN_PROGRESS                                           AWS::Lambda::Function                                        DBStartUpFunction                                            -                                                          
CREATE_IN_PROGRESS                                           AWS::Lambda::Function                                        DBShutDownFunction                                           Resource creation Initiated                                
CREATE_COMPLETE                                              AWS::Lambda::Function                                        DBStartUpFunction                                            -                                                          
CREATE_IN_PROGRESS                                           AWS::Lambda::Function                                        DBStartUpFunction                                            Resource creation Initiated                                
CREATE_COMPLETE                                              AWS::Lambda::Function                                        DBShutDownFunction                                           -                                                          
CREATE_COMPLETE                                              AWS::CloudFormation::Stack                                   auto-db-shutdown-stack                                       -                                                          
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Successfully created/updated stack - auto-db-shutdown-stack in None


that should create the stack


