# Quickstart — Deploy

This guide deploys the full self-service GPU inference lab into your AWS account.

> **Important build-order note:** the CDK stack references `../frontend/dist`
> (`Source.asset(path.join(__dirname, '../frontend/dist'))`). You **must build the frontend
> first** — otherwise `cdk synth`, `cdk deploy`, and even `npm test` fail with
> `Cannot find asset ... /frontend/dist`.

## 1. Install tooling

```bash
# global CLI tooling (synth uses ts-node)
npm i -g aws-cdk ts-node
```

You also need Node.js 22, npm, and AWS credentials configured (`aws configure` or
environment variables) for the target account/region.

## 2. Build the frontend (do this FIRST)

```bash
cd cdk/frontend
npm install
npm run build      # emits cdk/frontend/dist, which the stack packages
```

## 3. Install CDK dependencies

```bash
cd ..              # now in cdk/
npm install
npm run build      # tsc; optional but confirms the control plane compiles
```

## 4. (Optional) run the tests

The Jest tests synthesize the stack, so they also require `frontend/dist` to exist
(built in step 2).

```bash
npm test
```

## 5. Bootstrap and deploy

Provide your account and region via CDK context or the `CDK_DEFAULT_*` environment
variables.

```bash
# bootstrap once per account/region
npx cdk bootstrap -c account=<ACCOUNT_ID> -c region=<REGION>

# deploy
npx cdk deploy -c account=<ACCOUNT_ID> -c region=<REGION>
```

Equivalent with environment variables:

```bash
export CDK_DEFAULT_ACCOUNT=<ACCOUNT_ID>
export CDK_DEFAULT_REGION=<REGION>
npx cdk deploy
```

Note the stack outputs printed at the end of `deploy` — you will use
`FrontendBucketName`, `ComposeBucketName`, `ApiUrl`, and `CloudFrontDomain`.

## 6. Configure the poller subnet map

The `poller` Lambda launches Spot instances into specific subnets per AZ. Set the
`SUBNET_MAP_CONFIG` environment variable on the `tpot-booking-capacity-poller` function to
your own VPC's AZ→subnet mapping (comma-separated `az=subnet-id` pairs):

```bash
aws lambda update-function-configuration \
  --function-name tpot-booking-capacity-poller \
  --environment "Variables={SUBNET_MAP_CONFIG='us-east-1a=subnet-aaaa,us-east-1c=subnet-bbbb,us-west-2a=subnet-cccc'}"
```

If unset, the defaults hard-coded in `cdk/lambda/poller/handler.py` are used — override
them for your account.

## 7. Provision the portal login credential

The API is protected by a Lambda authorizer that reads an SSM SecureString parameter
`/tpot-booking/basic-auth-credentials` in `username:password` form:

```bash
aws ssm put-parameter \
  --name /tpot-booking/basic-auth-credentials \
  --type SecureString \
  --value 'myuser:mystrongpassword'
```

## 8. Upload the built SPA

Sync the frontend build into the frontend bucket from the `FrontendBucketName` output:

```bash
FRONTEND_BUCKET=$(aws cloudformation describe-stacks \
  --stack-name TpotBookingStack \
  --query "Stacks[0].Outputs[?OutputKey=='FrontendBucketName'].OutputValue" \
  --output text)

aws s3 sync cdk/frontend/dist "s3://$FRONTEND_BUCKET" --delete
```

> The stack also deploys the SPA via a `BucketDeployment` at `cdk deploy` time; this manual
> sync is how you push subsequent frontend rebuilds without a full redeploy.

## 9. Open the portal and configure Feishu notifications

1. Open the `CloudFrontDomain` URL in a browser and sign in with the credential from step 7.
2. Go to the **Settings** page and paste your **Feishu (Lark) bot webhook** URL
   (e.g. `https://open.feishu.cn/open-apis/bot/v2/hook/xxxxx`). Use **Test Webhook** to
   verify, then save. The webhook is stored in the DynamoDB notification-config table
   (`configId=global`) — it is never baked into code.
3. Create a booking (pick an instance type + deployment plan). The poller grabs Spot
   capacity, the deployer runs the SGLang docker-compose topology on NVMe, and you get a
   Feishu notification plus a **History** entry when it is ready.

## Updating deployment topologies without a redeploy

To change the SGLang `docker-compose-*.yaml` topologies in place, see
[`../cdk/docs/UPDATE-COMPOSE.md`](../cdk/docs/UPDATE-COMPOSE.md).

## Cleaning up

When you are done, follow [TEARDOWN.md](TEARDOWN.md) to destroy the stack and avoid
ongoing GPU charges.
