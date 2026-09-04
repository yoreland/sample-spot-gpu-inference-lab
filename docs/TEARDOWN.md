# Teardown & Cost Governance

GPU instances are the most expensive part of this stack. **The single most important
operational rule is: destroy what you spin up.** This guide covers tearing down cleanly and
verifying nothing GPU-related is left running.

## Why Spot (the cost narrative)

- A test may need only ~2 hours of GPU time. A **Capacity Block** reservation typically
  requires waiting **at least a day** and paying for the whole reserved window.
- **Spot** capacity has no reservation lead time and costs a fraction of on-demand, which
  fits short-lived, ad-hoc tests. The trade-off is possible interruption, which is
  acceptable for throwaway experiments.
- The lab is designed so instances are **short-lived**: launched on demand by the poller,
  deployed by the deployer, and expected to be terminated promptly after the test. Leaving
  a GPU Spot instance running is the main way this sample can cost you money.

## 1. Destroy the stack

From the `cdk/` directory:

```bash
cd cdk
npx cdk destroy -c account=<ACCOUNT_ID> -c region=<REGION>
```

This removes the API Gateway, CloudFront distribution, Lambdas, EventBridge rules, DynamoDB
tables, and the S3 buckets. The frontend and compose buckets are created with
`autoDeleteObjects`, so their contents are removed with the stack.

> `cdk destroy` tears down the **control plane**. It does **not** by itself terminate GPU
> EC2 instances that the poller launched, because those are created imperatively at runtime
> (not as CloudFormation resources). You must confirm those are gone — see below.

## 2. Let (or force) the orphan-cleaner reap instances

The `orphan-cleaner` Lambda runs every 15 minutes and terminates stray Spot instances
tagged by this platform (`Project=tpot-benchmark`) that are no longer tied to an active
booking. It is a safety net, not a substitute for verification. If you have already
destroyed the stack, the scheduled cleaner is gone too — so verify manually.

## 3. Manually verify no GPU resources linger

Check each region you configured in the poller `REGIONS` list (default
`us-east-1,us-east-2,us-west-2`):

```bash
for R in us-east-1 us-east-2 us-west-2; do
  echo "== $R =="
  aws ec2 describe-instances --region "$R" \
    --filters "Name=tag:Project,Values=tpot-benchmark" \
              "Name=instance-state-name,Values=pending,running,stopping,stopped" \
    --query "Reservations[].Instances[].[InstanceId,InstanceType,State.Name]" \
    --output table
done
```

Terminate anything still listed:

```bash
aws ec2 terminate-instances --region <REGION> --instance-ids <i-xxxx> <i-yyyy>
```

Also confirm no leftover EBS volumes and no leftover buckets:

```bash
# orphaned EBS volumes tagged by the platform
aws ec2 describe-volumes --region <REGION> \
  --filters "Name=tag:Project,Values=tpot-benchmark" "Name=status,Values=available" \
  --query "Volumes[].[VolumeId,Size,State]" --output table

# any residual project buckets (should be gone after cdk destroy)
aws s3 ls | grep tpot-booking || echo "no residual tpot-booking buckets"
```

Instances launched here use `DeleteOnTermination=true` root volumes and
`InstanceInitiatedShutdownBehavior=terminate`, so terminating the instance releases its
storage — but verifying costs you nothing and can save a lot.

## 4. Optional: remove the login credential

If you created the SSM SecureString credential and no longer need it:

```bash
aws ssm delete-parameter --name /tpot-booking/basic-auth-credentials
```

## Checklist

- [ ] `cdk destroy` completed
- [ ] No `tpot-benchmark`-tagged EC2 instances in any configured region
- [ ] No leftover `available` EBS volumes
- [ ] No residual `tpot-booking-*` S3 buckets
- [ ] (Optional) basic-auth SSM parameter removed
