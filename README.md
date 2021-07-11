<!-- note marker start -->
**NOTE**: This repo contains only the documentation for the private BoltsOps Pro repo code.
Original file: https://github.com/boltopspro/upcheck/blob/master/README.md
The docs are publish so they are available for interested customers.
For access to the source code, you must be a paying BoltOps Pro subscriber.
If are interested, you can contact us at contact@boltops.com or https://www.boltops.com

<!-- note marker end -->

# UpCheck Blueprint

[![BoltOps Badge](https://img.boltops.com/boltops/badges/boltops-badge.png)](https://www.boltops.com)

[Route53 Health Checks](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/welcome-health-checks.html) only supports monitoring external sites. This blueprint solves that issue by provisioning a [Lambda Function](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-upcheck-function.html) within your VPC that check URLs for their health status.

![](https://img.boltops.com/boltopspro/blueprints/upcheck/upcheck-dashboard.png)

The blueprint:

* Provisions Lambda Function that checks if sites are up and reports metrics to CloudWatch: 1) HealthCheckStatus and 2) ResponseTime
* Supports [VpcConfig](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-upcheck-function.html#cfn-upcheck-function-vpcconfig). This allows the check to run from within the network.
* By default, the template will create a CloudWatch Alarm and Route53 Health Check for each URL. You can monitor a little over 90 URLs. For more URLs, you can also launch another stack.
* You can also elect to disable the creation of the Route53 Health Checks and CloudWatch Alarms with `@enable_alarms = false` and `@enable_health_checks = false`. In this case, you're using the blueprint only to send custom CloudWatch metrics and can monitor thousands of URLs.
* You can subscribe to an SNS Topic to receive downtime alerts. You'll also get an alert when the URL is back up.  You can use an existing SNS Topic also.
* Supports [Lambda X-Ray tracing](https://docs.aws.amazon.com/upcheck/latest/dg/upcheck-x-ray.html)
* Customizable [ScheduledExpression](https://docs.aws.amazon.com/eventbridge/latest/userguide/scheduled-events.html) to control the frequency of the check. The default check interval is every 1 minute.
* CloudWatch Dashboard: The blueprint charts out the HealthCheckStatus and ResponseTime for all URLs.

## Usage

1. Add blueprint to Gemfile
2. Configure: configs/upcheck values
3. Deploy blueprint

## Add

Add the blueprint to your lono project's `Gemfile`.

```ruby
gem "upcheck", git: "git@github.com:boltopspro/upcheck.git"
```

## Configure

Use the [lono seed](https://lono.cloud/reference/lono-seed/) command to generate a starter config params files.

    LONO_ENV=development lono seed upcheck
    LONO_ENV=production  lono seed upcheck

The files in `config/upcheck` folder will look something like this:

    configs/upcheck/
    └── variables
        ├── development.rb
        └── production.rb

Configure the `configs/upcheck/variables` files.

```ruby
# If you're using internal sites, make sure the Lambda function is deployed to within a VPC
# that has network access to the internal site.
@sites = [
  {name: "demo-1", url: "https://external-site-1.com"},
  {name: "demo-2", url: "https://internal-site-2.local"},
]
```

## Deploy

Use the [lono cfn deploy](http://lono.cloud/reference/lono-cfn-deploy/) command to deploy.

    LONO_ENV=development lono cfn deploy upcheck --sure --no-wait
    LONO_ENV=production  lono cfn deploy upcheck --sure --no-wait

If you are using One AWS Account, use these commands instead: [One Account](docs/one-account.md).

### Getting Alerts

You can subscribe to an SNS Topic if to receive alerts when an URL is down and back up.  You can subscribe in the AWS console or with code:

configs/variables/development.rb:

```ruby
@subscription = [{
  Endpoint: "me@example.com", # String. Examples: http | https | email | email | sms | sqs | application | lambda
  Protocol: "email", # String
}]
```

Note, you will have to confirm the email subscription.

### Enable or Disable CloudWatch Alarms and Route53 Health Checks

By default, the template creates CloudWatch Alarms and Route53 Health Checks for each URL. You can globally disable the creation of CloudWatch Alarms and Route53 Health Checks with `@enable_alarms = false` and. `@enable_health_checks = false`. In this case, you're using the blueprint only to send CloudWatch metrics.

```ruby
@enable_alarms = true
@enable_health_checks = false
```

You can also selectively enable and disable Alarms and Health Checks on a per url basis:

```ruby
@sites = [
  {name: "demo-1", url: "https://external-site-1.com", alarm: false},
  {name: "demo-2", url: "https://internal-site-2.local", health_check: false},
]
```

Note, disabling an Alarm will also disable the Health Check, since each Health Check requires a corresponding Alarm.

Route53 Health Checks:

![](https://img.boltops.com/boltopspro/blueprints/upcheck/upcheck-healthcheck.png)

CloudWatch Alarms:

![](https://img.boltops.com/boltopspro/blueprints/upcheck/upcheck-alarms.png)

### Enable or Disable CloudWatch Dashboard

The blueprint creates an CloudWatch Dashboard charting out the HealthCheckStatus and ResponseTime for all URLs by default. You can disable this with:

```ruby
@enable_dashboard = false
```

![](https://img.boltops.com/boltopspro/blueprints/upcheck/upcheck-dashboard.png)

### CloudWatch Metric Names

Though the CloudWatch names like Namespace and Dimension can be set with Parameters, it is recommended to use the default values. Because once Custom Namespaces and Dimension are reported to CloudWatch metrics, they'll remain for 15 months. There's no current way to delete CloudWatch metrics.

### Schedule Expression

The default `ScheduleExpression` is `rate(1 minute)`.  You can override this with `@schedule_expression`.  Example:

configs/upcheck/variables/development.rb:

```ruby
@schedule_expression = "rate(5 minutes)"
```

The [ScheduledExpression](https://docs.aws.amazon.com/eventbridge/latest/userguide/scheduled-events.html) docs are helpful.

### Lambda Function in VPC

To configure the Lambda function with VpcConfig set `@subnet_ids` and either `@vpc_id` or `@security_group_ids`.

* When the `@vpc_id` is set, the template creates a managed security group for you, and the Lambda function is configured to use that security group.
* When `@security_group_ids` is set, the Lambda function will use those existing security groups.
* The subnet must be a private subnet with configured with a NAT.

Here's an example of the managed security group.

configs/upcheck/variables/development.rb:

```ruby
@subnet_ids = ["subnet-111"]
@vpc_id = "vpc-111"
```

For Lambda VPC to work, the subnet must be a private subnet configured with a NAT.

Note, Lambda functions configured with VPCs may take much longer to deploy, typically 30-45 minutes. This is because Lambda creates and attaches an ENI to the function to make the VPC feature possible. If the function is deleted or updated, requiring replacement, the ENI takes 30-45m to be removed. Because of this, it is recommended to write code for your Lambda function code without the VpcConfig first. Get it working and then add VpcConfig at the end.

### X-Ray Tracing

Uptime X-Ray tracing is set to `Active` by default. You can disable this by setting `@tracing_config_mode = false`. Example:

configs/upcheck/variables/development.rb:

```ruby
@tracing_config_mode = false
```

You can also change the mode with the same `@tracing_config_mode` variable:

```ruby
@tracing_config_mode = "Active" # or "PassThrough"
```
