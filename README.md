### 前提
![CodeDeploy](https://cdn-ssl-devio-img.classmethod.jp/wp-content/uploads/2019/04/sa20190410-00.png)

### 前提
* EC2キーペアが作成済みであること
* CloudWatch Agentの設定がパラメータストアに格納済みであること

### AMI作成
$ AMI_NAME=*作成するAMI名*  
$ echo ${AMI_NAME}  
$ cd packer  
$ packer validate -var ami-name=${AMI_NAME} template.json  
$ packer build -var ami-name=${AMI_NAME} template.json  
$ LAUNCH_CONFIG_IMAGE_ID=\`aws ec2 describe-images \  
  --filters "Name=tag-key,Values=Name" Name=tag-value,Values=${AMI_NAME} | jq -r '.Images[].ImageId'\`  
$ echo ${LAUNCH_CONFIG_IMAGE_ID}

### スタック作成
$ STACK_NAME=*作成するスタック名*  
$ TEMPLATE_DIR=../cloudformation/  
$ TEMPLATE_NAME=template.yml  
$ PRM_NAME=*作成済みのSSMパラメータ名*  
$ PROJECT_NAME=*AWSリソース名に付与するプレフィックス*  
$ EC2_KEY_PAIR=*作成済みのEC2キーペア名*  
$ SNS_SUBSCRIPTION=*SNSサブスクリプションに設定するメールアドレス*  
$ aws cloudformation create-stack \  
  --stack-name ${STACK_NAME} \  
  --template-body file://${TEMPLATE_DIR}${TEMPLATE_NAME} \  
  --parameters \  
    ParameterKey=eC2KeyPair,ParameterValue=${EC2_KEY_PAIR} \  
    ParameterKey=launchConfigImageId,ParameterValue=${LAUNCH_CONFIG_IMAGE_ID} \  
    ParameterKey=projectName,ParameterValue=${PROJECT_NAME} \  
    ParameterKey=snsSubscription,ParameterValue=${SNS_SUBSCRIPTION} \  
    ParameterKey=parameterName,ParameterValue=${PRM_NAME} \  
  --capabilities 'CAPABILITY_NAMED_IAM'  

### デプロイグループ作成
$ GROUP_NAME=${PROJECT_NAME}-deploygroup-blue-green  
$ APP_NAME=${PROJECT_NAME}-codedeploy-app  
$ ASG_NAME=${PROJECT_NAME}-asg  
$ TARGET_GROUP_NAME=${PROJECT_NAME}-tg  
$ ACCOUNT_ID=\`aws sts get-caller-identity --query 'Account' --output text\`  
$ ROLE_ARN=arn:aws:iam::${ACCOUNT_ID}:role/${PROJECT_NAME}-codedeploy-role  
$ aws deploy create-deployment-group \  
  --application-name ${APP_NAME} \  
  --deployment-group-name ${GROUP_NAME} \  
  --service-role-arn ${ROLE_ARN} \  
  --auto-scaling-groups ${ASG_NAME} \  
  --deployment-style deploymentType="BLUE_GREEN",deploymentOption="WITH_TRAFFIC_CONTROL" \  
  --blue-green-deployment-configuration terminateBlueInstancesOnDeploymentSuccess={action="TERMINATE"},deploymentReadyOption={actionOnTimeout=CONTINUE_DEPLOYMENT},greenFleetProvisioningOption={action=COPY_AUTO_SCALING_GROUP} \  
  --load-balancer-info targetGroupInfoList=[{name=${TARGET_GROUP_NAME}}]  

### リビジョン作成／アップロード
$ cd ../codedeploy/  
$ zip -r revision.zip .  
$ aws s3 ls s3://${PROJECT_NAME}-bucket-${ACCOUNT_ID}  
$ aws s3 cp ./revision.zip s3://${PROJECT_NAME}-bucket-${ACCOUNT_ID}  
$ aws s3 ls s3://${PROJECT_NAME}-bucket-${ACCOUNT_ID}  

### デプロイ実行
$ aws deploy create-deployment \  
  --application-name ${APP_NAME} \  
  --deployment-config-name CodeDeployDefault.AllAtOnce \  
  --deployment-group-name ${GROUP_NAME} \  
  --s3-location bundleType="zip",bucket=${PROJECT_NAME}-bucket-${ACCOUNT_ID},key=revision.zip \  
  --file-exists-behavior "OVERWRITE"  
