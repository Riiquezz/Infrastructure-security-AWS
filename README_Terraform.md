# Terraform Example

# 1. Pick an infrastructure as a code tool of choice
Terraform

# 2. Provision example single page application following the architecture diagram below
Terraform Template (main.tf)
```
provider "aws" {
  region = "us-east-1"
}

# S3 bucket for static content
resource "aws_s3_bucket" "spa_bucket" {
  bucket = "your-spa-bucket"
  acl    = "public-read"

  website {
    index_document = "index.html"
    error_document = "error.html"
  }
}

resource "aws_s3_bucket_policy" "spa_bucket_policy" {
  bucket = aws_s3_bucket.spa_bucket.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = "*"
        Action = "s3:GetObject"
        Resource = "${aws_s3_bucket.spa_bucket.arn}/*"
      }
    ]
  })
}

# CloudFront distribution
resource "aws_cloudfront_distribution" "cdn" {
  origin {
    domain_name = aws_s3_bucket.spa_bucket.bucket_regional_domain_name
    origin_id   = "S3-spa-bucket"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.oai.cloudfront_access_identity_path
    }
  }

  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-spa-bucket"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
  }

  price_class = "PriceClass_100"

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

resource "aws_cloudfront_origin_access_identity" "oai" {
  comment = "OAI for SPA bucket"
}

# API Gateway
resource "aws_api_gateway_rest_api" "api_gateway" {
  name        = "SPA_API"
  description = "API for SPA"
}

resource "aws_api_gateway_resource" "api_resource" {
  rest_api_id = aws_api_gateway_rest_api.api_gateway.id
  parent_id   = aws_api_gateway_rest_api.api_gateway.root_resource_id
  path_part   = "resource"
}

resource "aws_api_gateway_method" "api_method" {
  rest_api_id   = aws_api_gateway_rest_api.api_gateway.id
  resource_id   = aws_api_gateway_resource.api_resource.id
  http_method   = "GET"
  authorization = "NONE"
}

resource "aws_lambda_function" "api_lambda" {
  filename         = "lambda_function_payload.zip"
  function_name    = "SPA_Api_Lambda"
  handler          = "index.handler"
  source_code_hash = filebase64sha256("lambda_function_payload.zip")
  runtime          = "nodejs14.x"
  role             = aws_iam_role.lambda_exec.arn
}

resource "aws_lambda_permission" "api_gateway_invoke" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.api_lambda.function_name
  principal     = "apigateway.amazonaws.com"
}

resource "aws_api_gateway_integration" "lambda_integration" {
  rest_api_id             = aws_api_gateway_rest_api.api_gateway.id
  resource_id             = aws_api_gateway_resource.api_resource.id
  http_method             = aws_api_gateway_method.api_method.http_method
  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.api_lambda.invoke_arn
}

# IAM role and policy for Lambda
resource "aws_iam_role" "lambda_exec" {
  name = "lambda_exec_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action = "sts:AssumeRole",
      Effect = "Allow",
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "lambda_exec_policy" {
  name   = "lambda_exec_policy"
  role   = aws_iam_role.lambda_exec.id
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action = [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      Effect = "Allow",
      Resource = "arn:aws:logs:*:*:*"
    }]
  })
}
```

# 3. Write down security recommendation for each service
## S3 Bucket:

- Use a private bucket and restrict access using bucket policies and IAM roles.
- Enable S3 server-side encryption to encrypt data at rest.
- Enable versioning to protect against accidental overwrites and deletions.

## CloudFront:

- Use HTTPS by setting the viewer protocol policy to redirect HTTP to HTTPS.
- Restrict access to the S3 bucket through an Origin Access Identity (OAI).
- Use WAF (Web Application Firewall) to protect against common web exploits.

## API Gateway:

- Enable API Gateway logging and monitoring.
- Implement throttling to prevent abuse and Denial of Service (DoS) attacks.
- Use authorizers (like Cognito) to secure API access if needed.

## Lambda Function:

- Use IAM roles with least privilege for Lambda functions.
- Enable encryption for environment variables.
- Monitor and log Lambda execution.

# 4. Suggest logging and monitoring
## S3:

- Enable server access logging to log requests for access to your bucket.
- Enable AWS CloudTrail to log all S3 bucket API activity.

## CloudFront:

- Enable CloudFront standard logs to monitor all requests to your distribution.
- Use CloudWatch metrics to monitor CloudFront usage and performance.

## API Gateway:

- Enable CloudWatch Logs for API Gateway to capture request and response logs.
- Enable CloudWatch Metrics to monitor API Gateway performance.

## Lambda:

- Enable CloudWatch Logs to capture all log output from Lambda functions.
- Set up CloudWatch Alarms to trigger notifications on Lambda failures or performance issues.
