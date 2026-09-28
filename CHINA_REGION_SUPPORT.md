# China Region (aws-cn) Support

This document records how the four China-region (`aws-cn` partition) customizations
that previously lived on the fork's **v1.3 hand-written CloudFormation + `source/code/`**
architecture map onto the current **CDK / `source/app/` architecture** on `main`.

The upstream rewrite made the codebase **partition-aware by construction**, so all four
concerns are now handled natively. This file exists so a reviewer can verify each judgment
call without re-deriving it, and so future China deployers have a single reference.

## Summary

| # | Fork customization (old path)                                              | Status on current architecture | Where it is handled now |
|---|-----------------------------------------------------------------------------|--------------------------------|--------------------------|
| 1 | Region enumeration with `partition_name='aws-cn'` (`source/code/schedulers/instance_scheduler.py`) | **Already handled natively** | `source/app/instance_scheduler/util/session_manager.py` |
| 2 | `except ClientError` around RDS start/stop loops (`source/code/schedulers/rds_service.py`)          | **Already handled natively** (superset) | `source/app/instance_scheduler/scheduling/rds/rds.py` + `scheduling/scheduling_result.py` |
| 3 | `%arn_prefix%` = `arn:aws-cn` injection (`build-instance-scheduler-template.py`)                    | **Already handled natively** | CDK IAM policies use `Aws.PARTITION` / `AWS::Partition` throughout `source/instance-scheduler/lib/**` |
| 4 | Region-suffixed S3 bucket + `LocationConstraint` deploy (`makefile`, `build-s3-dist.sh`)           | **Handled by the CDK deploy model** (no port needed) | `deployment/build-s3-dist.sh` (CDK synth), standard CDK bootstrap/deploy |

No source code changes are required to run this solution in `cn-north-1` / `cn-northwest-1`.

---

## Item 1 — Region / partition awareness

**Fork (v1.3):** replaced
`self._valid_regions = boto3.Session().get_available_regions(service.service_name)`
with a branch that passed `partition_name='aws-cn'` when `AWS_REGION` was `cn-north-1`
or `cn-northwest-1`.

**Current architecture:** the app no longer enumerates available regions with
`get_available_regions` at all (the scheduler operates on operator-configured regions,
not a discovered list). Partition selection is centralized in
`source/app/instance_scheduler/util/session_manager.py`, which derives the partition from
the runtime region via boto3's own mapping:

```python
session = Session()
if session.get_partition_for_region(session.region_name) == "aws-cn":
    sts_regional_endpoint = "https://sts.{}.amazonaws.com.cn".format(session.region_name)
else:
    sts_regional_endpoint = "https://sts.{}.amazonaws.com".format(session.region_name)
```

```python
def get_role_arn(*, account_id, role_name):
    session = boto3.Session()
    partition = session.get_partition_for_region(session.region_name)   # -> "aws-cn" in CN
    return ":".join(["arn", partition, "iam", "", account_id, f"role/{role_name}"])
```

`AssumedRole.partition` is likewise `self.session.get_partition_for_region(self.region)`.
This is strictly better than the fork's hard-coded region check: it is partition-agnostic,
covers every partition, and additionally selects the CN-specific
`sts.<region>.amazonaws.com.cn` endpoint that the fork did not handle.

**Verdict: already native. Nothing to port.**

## Item 2 — RDS `ClientError` handling

**Fork (v1.3):** added `except ClientError as ce: self._logger.error(ce.response); return`
around the RDS start and stop loops in `rds_service.py`.

**Current architecture:** `source/app/instance_scheduler/scheduling/rds/rds.py`
processes each instance in `_process_decision`, and every `start_db_*` / `stop_db_*` call
is wrapped:

```python
try:
    self.rds_client.start_db_instance(DBInstanceIdentifier=runtime_info.resource_id)
    return SchedulingResult.success(decision)
except Exception as ex:
    logger.error(f"Error starting {runtime_info.arn}({str(ex)})")
    return SchedulingResult.client_exception(decision, ex)
```

`SchedulingResult.client_exception` returns a graceful result (records the ClientError
message, sets `action_taken=ERROR` and a failure `stored_state`) and never re-raises.
Because `ClientError` is a subclass of `Exception`, this catch covers exactly the fork's
case — and it is a **superset**: it is per-instance, so one instance's `ClientError` no
longer abandons the rest of the batch the way the fork's `return` did.

**Verdict: already native (and more robust). Nothing to port.**

## Item 3 — ARN partition prefix

**Fork (v1.3):** `build-instance-scheduler-template.py` injected
`%arn_prefix%` = `arn:aws-cn` for CN regions (else `arn:aws`) into the hand-written
CloudFormation templates.

**Current architecture:** those templates and that build step no longer exist. IAM
policies and every ARN are constructed in the CDK stacks under
`source/instance-scheduler/lib/**`, and they are uniformly partition-aware. A search for a
hard-coded `arn:aws:` literal across `source/instance-scheduler/lib/**/*.ts` returns
**zero matches**. Representative examples:

- `iam/rds-scheduling-permissions-policy.ts`: `Fn.sub("arn:${AWS::Partition}:rds:*:${AWS::AccountId}:db:*")`
- `iam/roles.ts`: `` `arn:${Aws.PARTITION}:iam::${accountId}:role/${roleName}` ``
- `iam/scheduler-role.ts`, `iam/ssm-params-region-registration-permission.ts`,
  `iam/asg-scheduling-permissions-policy.ts`, and the `lambda-functions/*.ts` policies
  all use `arn:${Aws.PARTITION}:...`.
- `observability/log-sns-forwarding.ts` uses `Stack.of(this).formatArn({...})`, which is
  partition-aware by definition.

At synth time `Aws.PARTITION` / `AWS::Partition` resolves to `aws-cn` when the stack is
deployed into a CN account/region, so every generated ARN is correct with no code change.

**Verdict: already native. Nothing to port.**

## Item 4 — China build / deploy

**Fork (v1.3):** patched the `makefile` and `deployment/build-s3-dist.sh` to take a
`region` argument, create a region-suffixed S3 bucket with a `LocationConstraint`, apply a
public-access-block, and copy the templates/lambda zip to that regional bucket.

**Current architecture:** the `makefile` is gone. `deployment/build-s3-dist.sh` now runs
`npm run synth` (CDK) and packages the synthesized templates and lambda assets into
`global-s3-assets` / `regional-s3-assets`; it is region/partition-agnostic and takes no
region argument. Deployment is the standard CDK flow (`cdk bootstrap` + `cdk deploy`, or
deploying the synthesized template), which places assets in the CDK bootstrap bucket of
whatever account/region — including a CN account — you are authenticated against. The
fork's bespoke bucket-creation / public-read logic is not needed under this model.

**Verdict: handled by the CDK deploy model. No source change required.** China deployers
should bootstrap and deploy against their CN account as they would in any other partition;
`Aws.PARTITION` and `get_partition_for_region` do the rest.

### China deployment notes

1. Configure the AWS CLI / credentials for a `cn-north-1` or `cn-northwest-1` account.
2. `cdk bootstrap` the target CN account/region (uses the CN partition automatically).
3. Build assets with `deployment/build-s3-dist.sh <bucket> <solution-name> <version>` and
   deploy the synthesized stack, or `cdk deploy` directly from `source/instance-scheduler`.
4. No `arn:aws-cn` edits, no `partition_name='aws-cn'` edits, and no region-suffixed
   bucket plumbing are required — all are resolved at runtime/synth time.

---

## Verification method

- Diffed the fork's custom commits against merge-base `v1.3` (`6f86b91`) to extract the
  exact semantics of each customization.
- Read the corresponding current-architecture code paths on `upstream/main`
  (`session_manager.py`, `rds/rds.py`, `scheduling_result.py`, `util/arn.py`,
  `source/instance-scheduler/lib/**`, `deployment/build-s3-dist.sh`).
- Confirmed by search that `get_available_regions` and hard-coded `arn:aws:` literals are
  absent from the app code and CDK library respectively.
