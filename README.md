# RDS-Aurora-PosgreSQL-CLI
AWS CLIでRDS Aurora PosgreSQLを作成する手順
# 作成手順
## (1)事前設定
###(a) 作業環境の準備
下記を準備します。
* aws-cliのセットアップ
* AdministratorAccessポリシーが付与され実行可能な、aws-cliのProfileの設定
### (b) CLI実行用の事前準備
これ以降のAWS-CLIで共通で利用するパラメータを環境変数で設定しておきます。
```shell
export PROFILE=<設定したプロファイル名称を指定。デフォルトの場合はdefaultを設定>
```
## (2)VPCの作成(CloudFormation利用)
IGWでインターネットアクセス可能で、パブリックアクセス可能なサブネットx2、プライベートなサブネットx2の合計４つのサブネットを所有するVPCを作成します。
### (a)テンプレートのダウンロード
私が作成し利用しているVPC作成用のCloudFormationテンプレートを利用します。まず、githubからテンプレートをダウンロードします。
```shell
curl -o vpc-4subnets.yaml https://raw.githubusercontent.com/Noppy/CfnCreatingVPC/master/vpc-4subnets.yaml
```
### (b)CloudFormationによるVPC作成
ダウンロードしたテンプレートを利用し、VPCをデプロイします。
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "DnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "DnsSupport",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "EnableNatGW",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "InternetAccess",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "RdsMultiAZsTestVPC"
  },
  {
    "ParameterKey": "VpcInternalDnsName",
    "ParameterValue": "rdstest."
  }
]'

aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name RDS-MultiAZs-Test-VPC \
    --template-body "file://./vpc-4subnets.yaml" \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --capabilities CAPABILITY_IAM ;
```

## (3)セキュリティーグループとBastion用EC2インスタンス作成
RDSで利用するセキュリティーグループと、RDSの接続テスト用のBastion用のEC2インスタンスを作成します。
### (a)利用情報の設定
```shell
# CloudFormationで作成したVPCのID取得
VPCID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name RDS-MultiAZs-Test-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcId`].[OutputValue]')

PubSub1ID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name RDS-MultiAZs-Test-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PublicSubnet1Id`].[OutputValue]')

echo $VPCID $PubSub1ID
```
### (b)Bastion用セキュリティグループ作成
```shell
# Bastionサーバ用セキュリティーグループ作成
BASTION_SG_ID=$(aws --profile ${PROFILE} --output text \
    ec2 create-security-group \
        --group-name BastionSG \
        --description "Allow ssh" \
        --vpc-id ${VPCID}) ;

aws --profile ${PROFILE} \
    ec2 create-tags \
        --resources ${BASTION_SG_ID} \
        --tags "Key=Name,Value=BastionSG" ;

#  Bastionサーバ用セキュリティーグループにSSHのinboundアクセス許可を追加
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${BASTION_SG_ID} \
        --protocol tcp \
        --port 22 \
        --cidr 0.0.0.0/0 ;
```
### (c)RDS用セキュリティグループ作成
```shell
# RDS用セキュリティーグループ作成
RDS_SG_ID=$(aws --profile ${PROFILE} --output text \
    ec2 create-security-group \
        --group-name RdsSG \
        --description "Allow ssh" \
        --vpc-id ${VPCID}) ;

aws --profile ${PROFILE} \
    ec2 create-tags \
        --resources ${RDS_SG_ID} \
        --tags "Key=Name,Value=RdsSG" ;

# RDS用セキュリティーグループにPosgreSQLのinboundアクセス許可を追加
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${RDS_SG_ID} \
        --protocol tcp \
        --port 5432 \
        --source-group ${BASTION_SG_ID} ;
```
### (d)Bastionインスタンスの作成
```shell
KEYNAME="CHANGE_KEY_PAIR_NAME"  #環境に合わせてキーペア名を設定してください。  

#最新のAmazon Linux2のAMI IDを取得します。
AMIID=$(aws --profile ${PROFILE} --output text \
    ec2 describe-images \
        --owners amazon \
        --filters 'Name=name,Values=amzn2-ami-hvm-2.0.????????.?-x86_64-gp2' \
                  'Name=state,Values=available' \
        --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' ) ;

TAGJSON='
[
    {
        "ResourceType": "instance",
        "Tags": [
            {
                "Key": "Name",
                "Value": "Bastion"
            }
        ]
    }
]'

aws --profile ${PROFILE} \
    ec2 run-instances \
        --image-id ${AMIID} \
        --instance-type t2.micro \
        --key-name ${KEYNAME} \
        --subnet-id ${PubSub1ID} \
        --security-group-ids ${BASTION_SG_ID} \
        --associate-public-ip-address \
        --tag-specifications "${TAGJSON}" ;
```        

## (4)RDS Aurora PosgreSQLの作成

### (a)利用情報の設定
```shell
# VPC,Subnet,SecurityGruopのID取得
VPCID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name RDS-MultiAZs-Test-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcId`].[OutputValue]')

PrvSub1ID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name RDS-MultiAZs-Test-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet1Id`].[OutputValue]')

PrvSub2ID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name RDS-MultiAZs-Test-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet2Id`].[OutputValue]')

RDS_SG_ID=$(aws --profile ${PROFILE} --output text \
    ec2 describe-security-groups \
        --filters "Name=group-name,Values=RdsSG" \
        --query "SecurityGroups[].GroupId")

echo $VPCID $PrvSub1ID $PrvSub2ID $RDS_SG_ID
```
### (b)RDS サブネットグループの作成
```shell
aws --profile ${PROFILE} \
    rds create-db-subnet-group \
        --db-subnet-group-name "multiazs-test-subnetgrp" \
        --db-subnet-group-descriptio "Multi AZs test" \
        --subnet-ids ${PrvSub1ID} ${PrvSub2ID} ;
```

### (c)RDS パラメータグループの作成
```shell
#パラメータグループファミリーの設定
RDS_PARAMETER_GROUP_FAMILY="aurora-postgresql10"

#クラスター パラメータグループの作成
aws --profile ${PROFILE} \
    rds create-db-cluster-parameter-group \
        --db-cluster-parameter-group-name "multiazs-test-cluster-parametergrp" \
        --db-parameter-group-family ${RDS_PARAMETER_GROUP_FAMILY} \
        --description  "Multi AZs test"

#パラメータグループの作成
aws --profile ${PROFILE} \
    rds create-db-parameter-group \
        --db-parameter-group-name "multiazs-test-parametergrp" \
        --db-parameter-group-family ${RDS_PARAMETER_GROUP_FAMILY} \
        --description "Multi AZs test"
```

### (d)RDS クラスターの作成
```shell
RDS_ENGINE_NAME='aurora-postgresql'
RDS_ENGINE_VERSION='10.7'
RDS_DB_NAME='multiaztestdb'
RDS_USER_NAME='root'
RDS_USER_PASS='dbPassword#'

#DBクラスターの作成
aws --profile ${PROFILE} \
    rds create-db-cluster \
        --engine ${RDS_ENGINE_NAME} \
        --engine-version ${RDS_ENGINE_VERSION} \
        --database-name ${RDS_DB_NAME} \
        --db-cluster-identifier "multi-azs-test-aurora-posgre-cluster" \
        --db-cluster-parameter-group-name "multiazs-test-cluster-parametergrp" \
        --vpc-security-group-ids ${RDS_SG_ID} \
        --db-subnet-group-name "multiazs-test-subnetgrp" \
        --master-username ${RDS_USER_NAME} \
        --master-user-password ${RDS_USER_PASS} \
        --backup-retention-period 3 \
        --preferred-backup-window "15:30-16:00" \
        --preferred-maintenance-window "Mon:15:00-Mon:15:30" ;

while true;
do
    STATUS=$(aws --profile ${PROFILE} --output text \
        rds describe-db-clusters \
            --db-cluster-identifier "multi-azs-test-aurora-posgre-cluster" \
        --query 'DBClusters[].Status'; );
    if [ $STATUS == "available" ]; then
        break
    fi
    echo "STATUS=${STATUS}. waiting..."
    sleep 1
done
echo "Done!!!!!"


```
### (d)RDS クラスターの作成
```shell
RDS_INSTANCE_CLASS='db.r5.large'
RDS_STORAGE_SIZE='100'

#DBインスタンスの作成
aws --profile ${PROFILE} \
    rds create-db-instance \
        --db-instance-identifier "multi-azs-test-aurora-posgre-instance-1" \
        --db-instance-class ${RDS_INSTANCE_CLASS} \
        --engine ${RDS_ENGINE_NAME} \
        --engine-version ${RDS_ENGINE_VERSION} \
        --db-parameter-group-name "multiazs-test-parametergrp" \
        --no-auto-minor-version-upgrade \
        --no-publicly-accessible \
        --db-cluster-identifier "multi-azs-test-aurora-posgre-cluster" \
        --preferred-maintenance-window "Mon:15:00-Mon:15:30"

aws --profile ${PROFILE} \
    rds create-db-instance \
        --db-instance-identifier "multi-azs-test-aurora-posgre-instance-2" \
        --db-instance-class ${RDS_INSTANCE_CLASS} \
        --engine ${RDS_ENGINE_NAME} \
        --engine-version ${RDS_ENGINE_VERSION} \
        --db-parameter-group-name "multiazs-test-parametergrp" \
        --no-auto-minor-version-upgrade \
        --no-publicly-accessible \
        --db-cluster-identifier "multi-azs-test-aurora-posgre-cluster" \
        --preferred-maintenance-window "Mon:15:00-Mon:15:30"
```
