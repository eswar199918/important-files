
Checkov is used to scan Infrastructure as Code (IaC) files like Terraform.

It identifies security misconfigurations and compliance issues before deployment.
It is typically run in the CI/CD pipeline to enforce shift-left security.

Example : 

cat <<EOF > main.tf

resource "aws_s3_bucket" "public_bucket" {
  bucket = "my-public-demo-bucket"
}

resource "aws_s3_bucket_acl" "public_acl" {
  bucket = aws_s3_bucket.public_bucket.id
  acl    = "public-read"
}

resource "aws_s3_bucket_public_access_block" "public_access" {
  bucket                  = aws_s3_bucket.public_bucket.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

EOF

checkov -d .
