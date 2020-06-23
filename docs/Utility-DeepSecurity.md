# Trend Micro Deep Security Research Notes


## Stacks and Resources (in order of which they are built)
Deep-Security (Stack)
+-Deep-Security-MasterMP (Stack)
  +-DSM1CompleteWaitHandle (WaitHandle)
  +-Deep-Security-MasterMP-DSIELB (Stack)
  | +-Deep-Security-MasterMP-DSIELB-ELBSG (Stack)
  | | +-ELBSG (SecurityGroup)
  | +-DSMELB (LoadBalancer)
  +-Deep-Security-MasterMP-DSIDSMSecurityGroup (Stack)
  |
  |
  +-Deep-Security-MasterMP-DSIRDSSecurityGroup (Stack)
  |
  |
  +-Deep-Security-MasterMP-DSDatabaseAbstract (Stack)
  | +-Deep-Security-MasterMP-DSDatabaseAbstract-DSPostgreSQLRDS (Stack)
  |
  +-Deep-Security-MasterMP-DSIDSMSecurityGroupIngressRules (Stack)
  |
  |
  +-Deep-Security-MasterMP-DSMNode1 (Stack)
  |
  |
  +-Deep-Security-MasterMP-DSMNode2 (Stack)

## Parameters in Original Template (with default values)
- Deep Security Manager Configuration   Template        CallerTemplate     Validation
  - DSCAdminName                        MasterAdmin     MasterAdmin        # Web console access
  - DSCAdminPassword                    (none)          (passed here)      '[a-zA-Z0-9!^*\-_+]*', min8,max41
  - DSIPInstanceType                    m3.xlarge
  - AWSIKeyPairName                     (none)                             Valid KeyPair
  - DSIPLicense                         (none)          PerHost            PerHost or BYOL
  - DSIMultiNode                        (none)          2                  1 or 2
  - DSIPLicenseKey                      (none)                              '[A-Z0-9]{2}-[A-Z0-9]{4}-[A-Z0-9]{5}-[A-Z0-9]{5}-[A-Z0-9]{5}-[A-Z0-9]{5}-[A-Z0-9]{5}' min37,max37
  - DSIPGUIPort                         443
  - DSIPHeartbeatPort                   4120
  - DSCLicenseType                      Enterprise                         Enterprise or Network

- Network Configuration
  - AWSIVPC                             (none)                             Valid VPC Id
  - DSISubnetID                         (none)                             Valid Subnet Id, public, same AZ as primary RDS Subnet
  - DBISubnet1                          (none)                             Valid Subnet Id, private, RDS, same AZ as DSM
  - DBISubnet2                          (none)                             Valid Subnet Id, private, RDS, second subnet
  - DSELBPosture                        Internet-facing                    Internet-facing or Internal
  - DSProxyUrl                          ''                                 Not sure where used

- Database Configuration
  - DBICAdminName                       dsadmin          dsadmin           1 to 16 alphanumeric
  - DBICAdminPassword                   (none)           (passwd)          '[a-zA-Z0-9!^*\-_+]*', min8,max41
  - DBPName                             dsm              dsm               database name
  - DBPMultiAZ                          false            true              true or false, for database MultiAZ
  - DBPEngine                           PostgreSQL       PostgreSQL        SQL, Oracle, PostgreSQL
  - DBPCreateDbInstance                 Yes                                Only Yes is allowed
  - DBIRDSInstanceSize                  db.m3.large
  - DBIStorageAllocation                100                                min 200 for SQL Server, 10 for Oracle
  - DBPBackupDays                       1               5                  0 to 35
  - DBPEndpoint                         Enter the ...                      Only used for existing Oracle instance

On Master Template, there is
  - ProtectedInstances                  (none)                             1-100, 101-500, 501-1000, 1001-2000
    This is used to pick one of 4 sizes for InstanceType and Storage amount
    The number ranges above are converted to 1, 2, 3, 4, then used to pick from the sizing tables
    1 = m4.large, db.m4.large, 50 RDS Storage
    2 = m4.large, db.m4.large, 150 RDS Storage
    3 = m4.xlarge, db.m4.xlarge, 200 RDS Storage
    4 = m4.xlarge, db.m4.xlarge, 300 RDS Storage

## Conditions
Conditions:
  DBTypeIsOracle: !Equals [ !Ref DBPEngine, Oracle ]
  DBTypeIsSQL: !Equals [ !Ref DBPEngine, SQL ]
  DBTypeIsEmbedded: !Equals [ !Ref DBPEngine, Embedded ]
  DBTypeIsPostgreSQL: !Equals [ !Ref DBPEngine, PostgreSQL ]

  IsFirstNode: !Equals [ !Ref DSMPMNode, No ]
  DoSQLSetup: !And [ !Condition DBTypeIsSQL, !Condition IsFirstNode ]

  AddToELB: !Not [ !Equals [ !Ref DSIELB, '' ]]
  IsFirstNodePlusELB: !And [ !Condition IsFirstNode, !Condition AddToELB ]

  UseBYOL: !Equals [ !Ref DSIPLicense, BYOL ]
  UsePerHost: !Equals [ !Ref DSIPLicense, PerHost ]
  PPUNotSelected: !Or [ !Condition UsePerHost, !Condition UseBYOL ]

  WaitNotProvided: !Equals [ DSM1CompleteWaitHandle, '' ]

  UseProxy: !Not [ !Equals, !Ref DSProxyUrl, '' ]

  SQLplusELB: !And [ !Condition AddToELB, !Condition DoSQLSetup ]
  AddNodePlusELB: !And [ !Not [ !Condition IsFirstNode ], !Condition AddToELB ]
  KeyProvided: !Not [ !Equals [ !Ref DSIPLicenseKey, XX-XXXX-XXXXX-XXXXX-XXXXX-XXXXX-XXXXX ]]

  AddAcAnswer: !And [ !Condition KeyProvided, !Condition PPUNotSelected ]
  InternetFacingELB: !Equals [ !Ref DSELBPosture, Internet-facing ]
  NetworkOnlyLicense: !Equals [ !Ref DSCLicenseType, Network ]
  RegionIsUsGovWest1: !Equals [ !Ref AWS::Region, us-gov-west-1 ]
  GovCloudCondition: !Equals [ !Ref AWS::Region, us-gov-west-1 ]

## Download Scripts
- https://aws-quickstart.s3.amazonaws.com/quickstart-trendmicro-deepsecurity/scripts/set-aia-settings.sh
- https://aws-quickstart.s3.amazonaws.com/quickstart-trendmicro-deepsecurity/scripts/kill-mp-web-installer.sh
- https://aws-quickstart.s3.amazonaws.com/quickstart-trendmicro-deepsecurity/scripts/add-aws-account-with-instance-role.sh
- https://aws-quickstart.s3.amazonaws.com/quickstart-trendmicro-deepsecurity/scripts/create-dsm-db.py
- https://aws-quickstart.s3.amazonaws.com/quickstart-trendmicro-deepsecurity/scripts/create-console-listener.sh
- https://aws-quickstart.s3.amazonaws.com/quickstart-trendmicro-deepsecurity/scripts/set-lb-settings.sh
- https://aws-quickstart.s3.amazonaws.com/quickstart-trendmicro-deepsecurity/scripts/reactivate-manager.sh

## Build Notes for Quickstart running in Testing Environment
Node 1: i-056185401c33d285c, private:172.21.80.8,  public:18.222.20.192 (agent not running), runs: default, setupLocalELB, setupGlobalELB
Node 2: i-049729924e5d2be6b, private:172.21.80.28, public:18.216.94.13  (agent running), runs: fixManagerLocalLbAddress, addDsmNode, setupLocalELB
ELB:
Logs saved in logs folder
Params passed to each stack also in logs to see how params affect result
