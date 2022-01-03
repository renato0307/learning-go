#


```sh
export AWS_PROFILE=my_aws_profile
export AWS_REGION=eu-west-1
```

```sh
aws cognito-idp create-user-pool \
    --pool-name learning-go-pool \
    --profile $AWS_PROFILE \
    --region $AWS_REGION
```