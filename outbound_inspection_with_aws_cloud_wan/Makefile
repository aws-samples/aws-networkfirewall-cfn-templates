.PHONY: deploy deploy-base-policy update-cloudwan deploy-workloads undeploy undeploy-workloads undeploy-cloudwan

deploy: deploy-base-policy update-cloudwan deploy-workloads

deploy-base-policy:
	aws cloudformation deploy --stack-name egress-inspection-core-network --template-file core_network_base_policy.yaml --no-fail-on-empty-changeset --region us-east-1

update-cloudwan:
	aws cloudformation update-stack --stack-name egress-inspection-core-network --template-body file://core_network_update_policy_edge_locations.yaml --region us-east-1
	aws cloudformation wait stack-update-complete --stack-name egress-inspection-core-network --region us-east-1

deploy-workloads: CORENETWORK_ID = $(shell aws cloudformation describe-stacks --stack-name "egress-inspection-core-network" --query 'Stacks[0].Outputs[?OutputKey == `CoreNetworkId`].OutputValue' --output text --region us-east-1 )
deploy-workloads: CORENETWORK_ARN = $(shell aws cloudformation describe-stacks --stack-name "egress-inspection-core-network" --query 'Stacks[0].Outputs[?OutputKey == `CoreNetworkArn`].OutputValue' --output text --region us-east-1 )
deploy-workloads:
	aws cloudformation deploy --stack-name cwan-nvirginia --template-file region1_resources.yaml --parameter-overrides CoreNetworkId="$(CORENETWORK_ID)" CoreNetworkArn="$(CORENETWORK_ARN)" --capabilities CAPABILITY_NAMED_IAM --no-fail-on-empty-changeset --region us-east-1 & \
	aws cloudformation deploy --stack-name cwan-ohio --template-file region2_resources.yaml --parameter-overrides CoreNetworkId="$(CORENETWORK_ID)" CoreNetworkArn="$(CORENETWORK_ARN)" --capabilities CAPABILITY_NAMED_IAM --no-fail-on-empty-changeset --region us-east-2 & \
	aws cloudformation deploy --stack-name cwan-oregon --template-file region3_resources.yaml --parameter-overrides CoreNetworkId="$(CORENETWORK_ID)" CoreNetworkArn="$(CORENETWORK_ARN)" --capabilities CAPABILITY_NAMED_IAM --no-fail-on-empty-changeset --region us-west-2
	


undeploy: undeploy-workloads undeploy-cloudwan

undeploy-workloads:
	aws cloudformation delete-stack --stack-name cwan-nvirginia --region us-east-1 & \
	aws cloudformation delete-stack --stack-name cwan-ohio --region us-east-2 & \
	aws cloudformation delete-stack --stack-name cwan-oregon --region us-west-2
	aws cloudformation wait stack-delete-complete --stack-name cwan-nvirginia --region us-east-1 & \
	aws cloudformation wait stack-delete-complete --stack-name cwan-ohio --region us-east-2 & \
	aws cloudformation wait stack-delete-complete --stack-name cwan-oregon --region us-west-2

undeploy-cloudwan:
	aws cloudformation delete-stack --stack-name egress-inspection-core-network --region us-east-1
	aws cloudformation wait stack-delete-complete --stack-name egress-inspection-core-network --region us-east-1