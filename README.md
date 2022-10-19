Push Docker image to ECR
```sh
aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/m2a7z1o1
aws ecr-public create-repository --repository-name frontend-app --region us-east-1
docker tag frontend-app:latest public.ecr.aws/m2a7z1o1/frontend-app:latest 
docker push public.ecr.aws/m2a7z1o1/frontend-app:latest 
```

Docker Context for Amazon ECS
```sh
docker context create ecs myecs
docker context ls
docker context use myecs
```


Creating a cluster with a Fargate Linux task using the Amazon CLI
```sh
aws ecs create-cluster --cluster-name demo-cluster --region eu-central-1
aws ecs describe-clusters --cluster demo-cluster
aws ecs register-task-definition --cli-input-json file://fargate-task.json
aws ecs list-task-definitions
aws ecs create-service --cluster demo-cluster --service-name frontend-service --task-definition frontend-app:1 --desired-count 1 --launch-type "FARGATE" --network-configuration "awsvpcConfiguration={subnets=[subnet-02caf3f4a7dab08f6],securityGroups=[sg-095938d5e717361ea],assignPublicIp=ENABLED}"
aws ecs create-service --cli-input-json file://fargate-service.json --region eu-central-1
aws ecs update-service --cluster demo-cluster --service frontend-service --task-definition frontend-app:2
aws ecs list-services --cluster demo-cluster
aws ecs describe-services --cluster demo-cluster --services frontend-service
aws ecs list-tasks --cluster demo-cluster --service frontend-service
aws ecs describe-tasks --cluster demo-cluster --tasks arn:aws:ecs:eu-central-1:632296647497:task/demo-cluster/74b9961d60e544309f4f9d5798b1eecd
aws ec2 describe-network-interfaces --network-interface-id  eni-050380a8228b471b3
```

Clean Up
```sh
aws ecs delete-service --cluster demo-cluster --service frontend-service --force
aws ecs delete-cluster --cluster demo-cluster 
```

Creating the task execution IAM role
```sh
aws iam create-role \
      --role-name ecsTaskExecutionRole \
      --assume-role-policy-document file://ecs-tasks-execution-role.json

aws iam create-policy \
      --policy-name CloudWatchFullAccessPolicy \
      --policy-document file://ecs-cloud-watch-policy.json

aws iam attach-role-policy \
      --role-name ecsTaskExecutionRole \
      --policy-arn arn:aws:iam::632296647497:policy/CloudWatchFullAccessPolicy
```

Create an Application Load Balancer
```sh
aws elbv2 create-load-balancer \
     --name frontend-alb \
     --subnets subnet-02caf3f4a7dab08f6 subnet-0e00855f4313be466 \
     --security-groups sg-095938d5e717361ea \
     --region eu-central-1

aws elbv2 create-target-group \
     --name frontend-app-target \
     --protocol HTTP \
     --port 8080 \
     --target-type ip \
     --vpc-id vpc-064f43e135e1ecbc0 \
     --region eu-central-1

aws elbv2 create-listener \
     --load-balancer-arn arn:aws:elasticloadbalancing:eu-central-1:632296647497:loadbalancer/app/frontend-alb/ba9c611e308dea4a \
     --protocol HTTP \
     --port 8080 \
     --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:eu-central-1:632296647497:targetgroup/frontend-app-target/0ebf7389864148d0 \
     --region eu-central-1
```

```
"loadBalancers": [
  {
      "targetGroupArn": "arn:aws:elasticloadbalancing:region:aws_account_id:targetgroup/bluegreentarget1/209a844cd01825a4",
      "containerName": "frontend-app",
      "containerPort": 80
  }
],
```

```sh
aws elbv2 delete-load-balancer --load-balancer-arn loadbalancer-arn
aws elbv2 delete-target-group --target-group-arn targetgroup-arn
```