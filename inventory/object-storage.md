# Object Storage Inventory

This file documents the AWS CLI profiles and Wasabi S3 endpoints configured for the `dev-agent` server.

## AWS CLI Profiles

The AWS CLI is already configured on `dev-agent`.

| Profile | Purpose | Usage |
|---|---|---|
| `default` | AWS account | Used automatically when no `--profile` is specified |
| `wasabi` | Wasabi account | Always use `--profile wasabi` and the endpoint matching the bucket region |

> Do not store access keys or secret keys in this inventory file.

## Wasabi CLI Usage

When accessing a Wasabi bucket, both of these must be specified:

1. `--profile wasabi`
2. `--endpoint-url` for the region where the bucket is located

Example for the `ai-image-media` bucket:

```bash
aws s3 ls s3://ai-image-media/ \
  --profile wasabi \
  --endpoint-url https://s3.eu-central-1.wasabisys.com
```

`ai-image-media` is located in `eu-central-1`.

General pattern:

```bash
aws s3 <command> s3://<bucket-name>/ \
  --profile wasabi \
  --endpoint-url https://s3.<bucket-region>.wasabisys.com
```

## Wasabi Region Endpoints

The current bucket inventory uses these regions:

| Region | Wasabi S3 endpoint |
|---|---|
| `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `eu-central-2` | `https://s3.eu-central-2.wasabisys.com` |
| `eu-west-1` | `https://s3.eu-west-1.wasabisys.com` |
| `eu-west-2` | `https://s3.eu-west-2.wasabisys.com` |
| `us-central-1` | `https://s3.us-central-1.wasabisys.com` |
| `us-west-1` | `https://s3.us-west-1.wasabisys.com` |

## Wasabi Bucket Inventory

Source: `all-bucket-utilization-2026-09-24.csv`

| Bucket | Region | Endpoint |
|---|---|---|
| `rendedvideos` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `rf-upload-test` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `rf-backup-glacier` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `videostatic.rfstat.com` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `stock-footage-to-be-deleted` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `uploads-test-bucket-old` | `us-central-1` | `https://s3.us-central-1.wasabisys.com` |
| `tts-cache` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `rf-gm-cache` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `uploads-test-bucket` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `cdn-stocks.renderforest.com` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `ai-images` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `ai-generator-models` | `us-west-1` | `https://s3.us-west-1.wasabisys.com` |
| `rf-transformed-images-de` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `gm-media` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `mockups` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `ai-image-media` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `stocks.renderforest.com` | `eu-central-2` | `https://s3.eu-central-2.wasabisys.com` |
| `rend-messages` | `eu-central-1` | `https://s3.eu-central-1.wasabisys.com` |
| `stock-footage-ec2` | `eu-central-2` | `https://s3.eu-central-2.wasabisys.com` |
| `rf-video-cache` | `eu-central-2` | `https://s3.eu-central-2.wasabisys.com` |
| `rf-public` | `eu-west-2` | `https://s3.eu-west-2.wasabisys.com` |
| `ai-videos.renderforest.com` | `eu-west-1` | `https://s3.eu-west-1.wasabisys.com` |

## Rule for the AI Agent

Before running an AWS CLI command against Wasabi:

1. Identify the bucket.
2. Find the bucket region in this inventory.
3. Use the `wasabi` AWS CLI profile.
4. Pass the S3 endpoint matching that region.

For normal AWS operations, use the default AWS CLI profile unless another AWS profile is explicitly required.
