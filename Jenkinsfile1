pipeline {
  agent any

  options { timestamps() }   // keep simple; removed ansiColor

  environment {
    AWS_REGION     = 'us-east-1'          // change if needed
    BUCKET_PREFIX  = 'etech-main-bucket'  // make globally unique
    IAM_USER_BASE  = 'etech-writer'       // base IAM username
  }

  stages {
    stage('Create S3 Bucket + IAM User + Attach User Policy (main only)') {
      // Works for both “Pipeline script from SCM” and inline Pipeline jobs.
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

# ----- Derived names -----
SHORT_SHA="$(git rev-parse --short=7 HEAD || echo "${BUILD_NUMBER}")"
BUCKET="$(tr "[:upper:]" "[:lower:]" <<< "${BUCKET_PREFIX}-${SHORT_SHA}")"
ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
USER_NAME="${IAM_USER_BASE}-${SHORT_SHA}"

echo "Region      : ${AWS_REGION}"
echo "Bucket      : ${BUCKET}"
echo "IAM User    : ${USER_NAME}"
echo "Account     : ${ACCOUNT_ID}"

# ----- 1) Ensure S3 bucket -----
if [[ "${AWS_REGION}" == "us-east-1" ]]; then
  aws s3api create-bucket --bucket "${BUCKET}" || echo "Bucket may already exist"
else
  aws s3api create-bucket --bucket "${BUCKET}" \
    --create-bucket-configuration LocationConstraint="${AWS_REGION}" || echo "Bucket may already exist"
fi

aws s3api put-public-access-block \
  --bucket "${BUCKET}" \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# ----- 2) Ensure IAM user -----
if aws iam get-user --user-name "${USER_NAME}" >/dev/null 2>&1; then
  echo "IAM user exists: ${USER_NAME}"
else
  aws iam create-user --user-name "${USER_NAME}"
  echo "Created IAM user: ${USER_NAME}"
fi

# Create one access key if none exists (AWS max 2)
KEY_COUNT="$(aws iam list-access-keys --user-name "${USER_NAME}" --query 'length(AccessKeyMetadata)' --output text)"
if [[ "${KEY_COUNT}" -eq 0 ]]; then
  echo "Creating access key for ${USER_NAME}"
  aws iam create-access-key --user-name "${USER_NAME}" > iam_access_key.json
else
  echo "User already has ${KEY_COUNT} access key(s); not creating another."
fi

# ----- 3) Attach inline user policy (avoids bucket policy principal issues) -----
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
  --policy-name "S3WriteTo-${BUCKET}" \
  --policy-document file://user-s3-write-policy.json

echo "Attached inline user policy granting write to s3://${BUCKET}"
'''
      }
    }
  }

  post {
    success {
      script {
        if (fileExists('iam_access_key.json')) {
          archiveArtifacts artifacts: 'iam_access_key.json', onlyIfSuccessful: true
          echo 'Archived iam_access_key.json (download and store securely).'
        }
      }
    }
  }
}
