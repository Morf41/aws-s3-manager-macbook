pipeline {
  agent any

  options { timestamps() }

  environment {
    AWS_REGION        = 'us-east-1'          // change if needed
    BUCKET_PREFIX     = 'anangafac-s3-bucket'  // make globally unique
    IAM_USER_BASE     = 'anangafac'       // base IAM username
    // Dangerous toggles - keep false in CI for safety
    CREATE_ACCESS_KEY = 'false'  // set to 'true' to create an access key (NOT recommended in CI)
    ARCHIVE_ACCESS_KEY = 'false' // set to 'true' to archive iam_access_key.json (VERY RISKY)
    // Optional: a prefix for policy names to keep within IAM limits (avoid very long names)
    POLICY_NAME_PREFIX = 'S3WriteTo'
  }

  stages {
    stage('Create S3 Bucket + IAM User + Attach User Policy (main only)') {
      when {
        anyOf {
          branch 'main'
          expression {
            def gb = env.GIT_BRANCH ?: ''
            def bn = env.BRANCH_NAME ?: ''
            return gb == 'origin/main' || gb == 'main' || bn == 'main'
          }
        }
      }

      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail

# Simple retry helper
retry_cmd() {
  local attempts=${1:-3}
  shift
  local i=0
  until "$@"; do
    i=$((i+1))
    if [ "$i" -ge "$attempts" ]; then
      return 1
    fi
    sleep 2
    echo "Retrying ($i/$attempts): $*"
  done
  return 0
}

if ! command -v aws >/dev/null 2>&1; then
  echo "ERROR: aws CLI not found on agent. Aborting."
  exit 2
fi

SHORT_SHA="$(git rev-parse --short=7 HEAD 2>/dev/null || echo "${BUILD_NUMBER}")"
BUCKET="$(tr "[:upper:]" "[:lower:]" <<< "${BUCKET_PREFIX}-${SHORT_SHA}")"
ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
USER_NAME="${IAM_USER_BASE}-${SHORT_SHA}"

echo "Region      : ${AWS_REGION}"
echo "Bucket      : ${BUCKET}"
echo "IAM User    : ${USER_NAME}"
echo "Account     : ${ACCOUNT_ID}"

# Check whether the bucket is already owned by THIS account
OWNED_BUCKET_COUNT="$(aws s3api list-buckets --query "length(Buckets[?Name=='${BUCKET}'])" --output text || echo "0")"
if [[ "${OWNED_BUCKET_COUNT}" -gt 0 ]]; then
  echo "Bucket ${BUCKET} already exists and is owned by this account. Reusing it."
else
  echo "Bucket ${BUCKET} not found in this account. Attempting to create it."

  if [[ "${AWS_REGION}" == "us-east-1" ]]; then
    if ! retry_cmd 3 aws s3api create-bucket --bucket "${BUCKET}"; then
      echo "create-bucket failed. The bucket name may already exist in another account. Checking..."
      # If create-bucket failed, check if it exists at all (other account)
      if aws s3api head-bucket --bucket "${BUCKET}" >/dev/null 2>&1; then
        echo "Bucket ${BUCKET} exists and is accessible (unexpected)."
      else
        echo "Bucket ${BUCKET} appears to exist but is not owned by this account — aborting to avoid accidental writes to another account's bucket."
        exit 3
      fi
    fi
  else
    if ! retry_cmd 3 aws s3api create-bucket --bucket "${BUCKET}" --create-bucket-configuration LocationConstraint="${AWS_REGION}"; then
      echo "create-bucket failed. The bucket name may already exist in another account. Checking..."
      if aws s3api head-bucket --bucket "${BUCKET}" >/dev/null 2>&1; then
        echo "Bucket ${BUCKET} exists and is accessible (unexpected)."
      else
        echo "Bucket ${BUCKET} appears to exist but is not owned by this account — aborting to avoid accidental writes to another account's bucket."
        exit 3
      fi
    fi
  fi
fi

# Apply strict public access block
aws s3api put-public-access-block \
  --bucket "${BUCKET}" \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true || echo "Warning: put-public-access-block may have failed"

# ----- Ensure IAM user -----
if aws iam get-user --user-name "${USER_NAME}" >/dev/null 2>&1; then
  echo "IAM user exists: ${USER_NAME}"
else
  echo "Creating IAM user: ${USER_NAME}"
  aws iam create-user --user-name "${USER_NAME}"
fi

# ----- Attach inline user policy (scoped to bucket) -----
# Sanitize policy name and ensure not too long
POLICY_NAME="${POLICY_NAME_PREFIX}-${BUCKET}"
# AWS policy names are limited; truncate if necessary
POLICY_NAME="${POLICY_NAME:0:128}"

cat > user-s3-write-policy.json <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListTheBucket",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::${BUCKET}"
    },
    {
      "Sid": "WriteObjectsToBucket",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::${BUCKET}/*"
    }
  ]
}
POLICY

aws iam put-user-policy \
  --user-name "${USER_NAME}" \
  --policy-name "${POLICY_NAME}" \
  --policy-document file://user-s3-write-policy.json

echo "Attached inline user policy ${POLICY_NAME} granting write to s3://${BUCKET}"

# ----- Optional: Create access key (dangerous) -----
if [[ "${CREATE_ACCESS_KEY:-false}" == "true" ]]; then
  KEY_COUNT="$(aws iam list-access-keys --user-name "${USER_NAME}" --query 'length(AccessKeyMetadata)' --output text)"
  if [[ "${KEY_COUNT}" -eq 0 ]]; then
    echo "Creating access key for ${USER_NAME}"
    aws iam create-access-key --user-name "${USER_NAME}" > iam_access_key.json
    echo "WROTE iam_access_key.json - secret only returned once. Handle securely."
    # If not explicitly archiving, immediately remove the file to avoid leakage.
    if [[ "${ARCHIVE_ACCESS_KEY}" != "true" ]]; then
      echo "ARCHIVE_ACCESS_KEY is not true; moving iam_access_key.json to ephemeral and deleting to avoid leakage."
      # Print a short notice with the AccessKeyId (but not SecretAccessKey) to let operator find it in logs if needed (be careful)
      cat iam_access_key.json | jq -r '.AccessKey | {AccessKeyId:.AccessKeyId}' || true
      rm -f iam_access_key.json
    else
      echo "ARCHIVE_ACCESS_KEY=true - iam_access_key.json will be left in workspace to be archived by Jenkins post actions. Ensure this is stored securely and rotated."
    fi
  else
    echo "User already has ${KEY_COUNT} access key(s); not creating another (AWS max 2)."
  fi
else
  echo "CREATE_ACCESS_KEY is false; not creating keys in CI (recommended)."
fi
'''
      }
    }
  }

  post {
    success {
      script {
        if (fileExists('iam_access_key.json')) {
          if (env.ARCHIVE_ACCESS_KEY == 'true') {
            archiveArtifacts artifacts: 'iam_access_key.json', onlyIfSuccessful: true
            echo 'Archived iam_access_key.json (download and store securely).'
          } else {
            // remove file just in case the script left it
            sh "rm -f iam_access_key.json || true"
            echo 'Cleaned iam_access_key.json from the workspace to avoid accidental leakage.'
          }
        }
      }
    }
    always {
      script {
        // Clean up temp policy file if present
        sh "rm -f user-s3-write-policy.json || true"
      }
    }
  }
}
