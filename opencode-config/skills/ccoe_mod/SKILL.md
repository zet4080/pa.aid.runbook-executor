---
name: ccoe_mod
description: Use when selecting or editing CCoE Terraform reusable modules for VPC, EKS, Aurora, Cognito, ALB, SQS, DynamoDB, ECR, WAF, ArgoCD, or monitoring
---
# Skill: @ccoe_mod Terraform Modules

## Purpose

Reference for CCoE Terraform reusable modules (VPC, EKS, Aurora, Cognito, ALB, SQS, DynamoDB, ECR, WAF, ArgoCD, monitoring). Covers purpose, inputs, outputs, and usage patterns.

## When to Use

- Selecting which CCoE module to use
- Editing or configuring CCoE Terraform modules
- Understanding module requirements and outputs
- Building infrastructure with reusable modules

---

## Overview

This skill provides knowledge about the CCoE (Cloud Centre of Excellence) Terraform modules available at `/workspace/ccoe_mod`. Use this to understand which modules exist, their purpose, required inputs, outputs, and usage patterns when building Terraform infrastructure.

All modules are sourced via SSH git:
```
git::ssh://git@git.proalpha.com/cce_mod/<module-name>.git
```

---

## Available Modules

### argocd
**Purpose:** Deploys ArgoCD on Kubernetes/EKS with OIDC integration, custom accounts, and resource limits.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/argocd.git//argocd`

Required inputs: `cluster_name`, `url`, `idp_argocd_name`, `idp_endpoint`, `idp_argocd_allowed_oauth_scopes`, `config_bucket_name`, `controller_resources`, `server_resources`, `redis_resources`  
Optional: `high_availability` (false), `metrics_enabled` (false), `dex_enabled` (true), `ingress_class_name` (traefik), `git_repos` ([]), `accounts` ([]), `groups` ([])

---

### aurora-db-config
**Purpose:** Creates a Lambda function that connects to Aurora PostgreSQL and creates database users.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/aurora-db-config.git`

Required inputs: `RDS_HOST`, `RDS_SECRET_NAME`, `RDS_USERNAME`, `create_users`, `vpc_id`, `vpc_subnet_ids`, `vpc_subnet_cidrs`  
Outputs: `lambda_function_arn`

---

### autoscaling-schedule
**Purpose:** Manages EC2 Auto Scaling group schedules for business hours scale up/down.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/autoscaling-schedule.git`

Required inputs: `autoscaling_group_filters`, `scale_up_max_node_count`, `scale_down_max_node_count`  
Optional: `scale_up_reccurence` ("0 7 * * MON-FRI"), `scale_down_reccurence` ("0 18 * * MON-FRI"), `timezone` ("Europe/Berlin"), min/desired counts (0)  
Outputs: `autoscaling_group_name`

---

### awx
**Purpose:** AWX automation platform deployment (submodules for CoreERP AWX).  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/awx.git`

---

### buckets
**Purpose:** Creates S3 buckets with KMS encryption and optional cross-account access.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/buckets.git`

Required inputs: `bucket_name`, `kms_alias_name`  
Optional: `versioning` (true), `acl` (private), `account_ids` ([]), `role_ids` ([]), `grant_write_permissions` (false), `custom_tags` ({}), `backup_tags` ({})  
Outputs: `bucket_arn`, `bucket_id`, `kms_arn`

---

### cert-manager
**Purpose:** Deploys cert-manager for Kubernetes TLS with Root CA -> Intermediate CA -> App Certificates chain.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/cert-manager.git`  
Requirements: terraform >= 0.13, kubectl >= 2.0.4. No configurable inputs.

---

### cognito
**Purpose:** Creates AWS Cognito user pools, user clients, and groups.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/cognito.git//<submodule>`  
Submodules: `user-pool`, `user-clients`, `lambda`

```hcl
module "cognito_user_pool" {
  source          = "git::ssh://git@git.proalpha.com/cce_mod/cognito.git//user-pool"
  user_pool       = local.cognito.user_pool
  group_name      = local.cognito.group.name
  lambda_func_arn = aws_lambda_function.cognito_pre_sign_up.arn
}
```

---

### configvariables
**Purpose:** Parses YAML config files into Terraform data structures.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/configvariables.git`

Required inputs: `configvariables` (use `yamldecode(file("./configvariables.yaml"))`)  
Outputs: `aws`, `config`, `tags`

```hcl
module "configvariables" {
  source          = "git::ssh://git@git.proalpha.com/cce_mod/configvariables.git"
  configvariables = yamldecode(file("./configvariables.yaml"))
}
```

---

### coreerp
**Purpose:** CoreERP infrastructure components.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/coreerp.git`

---

### dynamo_db
**Purpose:** Creates DynamoDB tables with tenant_id partition key for multitenant apps, with KMS encryption.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/dynamo_db.git`

Required inputs: `tables` (map of table configs)  
Optional: `encryption_mode` (customer_managed), `kms_alias` (alias/dynamodb_key)  
Outputs: `tables` (map of IDs and ARNs)

Table config fields: `billing_mode`, `key`, `hash_key`, `tenant_isolation`, `attributes` (list of {name, type}), `local_indexes`, `global_secondary_indexes`, `write_capacity`, `read_capacity`

```hcl
module "dynamodb_tables" {
  source = "git::ssh://git@git.proalpha.com/cce_mod/dynamo_db.git"
  tables = {
    SERVICE-EXAMPLE = {
      billing_mode = "PAY_PER_REQUEST"
      key          = "example_key"
      attributes   = [{ name = "example_key", type = "S" }]
    }
  }
}
```

---

### ecr
**Purpose:** Creates Amazon ECR repositories with lifecycle policies and cross-account access.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/ecr.git`

Required inputs: `name`, `tags`  
Optional: `image_tag_mutability` (MUTABLE), `scan_on_push` (true), `lifecycle_policy`, `is_public` (false), `read_access_list` ([]), `write_access_list` ([])  
Outputs: `ecr` (URLs/properties), `name`

---

### eks
**Purpose:** Creates AWS EKS cluster with managed node groups, Karpenter support, IAM roles, and VPC CNI.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/eks.git`

Required inputs: `vpc_id`, `subnet_cidrs`, `nodegroups`  
Optional: `cluster_name` (eks), `cluster_version` (1.23), `enable_karpenter` (false), `enable_public_api_access` (true), `enable_ssm_node_access` (false), `iam_roles`, `cni_options` ({}), `cluster_ip_family`, `cluster_log_types`

Nodegroup fields: `instance_types`, `capacity_type` (ON_DEMAND/SPOT), `min_capacity`, `max_capacity`, `desired_capacity`, `subnet_cidrs`, `disk_size`, `platform` (linux/windows)

Outputs: `cluster_name`, `cluster_endpoint`, `cluster_oidc_issuer_url`, `oidc_provider_arn`, `cluster_security_group_id`, `autoscaling-groups`, `autoscaling-groups-count`, `node_group_iam_role_arns`, `node_group_iam_role_names`

---

### elasticache
**Purpose:** Sets up ElastiCache clusters integrated with EKS (auto-detects network from EKS subnets).  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/elasticache.git`

Key inputs: `cache_name`, `application_names`, `eks_cluster_name`, `environment`, `account_id`

---

### eventbus
**Purpose:** Event bus infrastructure with emitter and receiver submodules.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/eventbus.git`  
Submodules: `emitter`, `receiver`

---

### ingress-controller
**Purpose:** Deploys Traefik as Kubernetes ingress controller via Helm.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/ingress-controller.git`

Optional: `load_balancer_ingress_ip` (1.2.3.4), `add_forwarded_headers` (false), `trusted_ips` ([]), `nodeSelector` ({})

---

### kubecost
**Purpose:** Deploys Kubecost for Kubernetes cost monitoring with CUR export and Athena integration.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/kubecost.git`  
Submodules: `cur`, `cur_crawler`, `kubecost`

---

### kubernetes-cron
**Purpose:** Creates Kubernetes CronJobs with flexible execution modes.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/kubernetes-cron.git`

Required inputs: `name`, `namespace`, `schedule`, `script`  
Optional: `exec_mode` (cronjob|pod), `image` (pa/platform/patools:latest), `pod_name`, `pod_selector`, `container_name`, `shell` (/bin/sh), `values` ({})

---

### kubernetes-dashboard
**Purpose:** Deploys Kubernetes Dashboard with OAuth2 proxy authentication.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/kubernetes-dashboard.git`

Required inputs: `zone_name`, `namespace`, `api`, `web`, `s3_oidc_config_bucket`, `s3_oidc_config_path`  
Optional: `subdomain` (k8sdashboard), `load_balancer_ingress_ip`, `ingress_class_name` (traefik)

---

### lambda-coralogix
**Purpose:** Deploys Lambda function with automatic Coralogix logging export.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/lambda-coralogix.git`

Key inputs: `path`, `function_name`, `role_policy_json`, `coralogix_private_key`, `coralogix_application`

```hcl
module "lambda" {
  source                = "git::ssh://git@git.proalpha.com/cce_mod/lambda-coralogix.git"
  path                  = "<bucket>/lambda.myfunction.py/BUILD-..."
  function_name         = "my-function"
  role_policy_json      = data.aws_iam_policy_document.this.json
  coralogix_private_key = data.aws_s3_object.coralogix_private_key.body
  coralogix_application = "my-app"
}
```

---

### lambda-edge-azure-auth
**Purpose:** Lambda@Edge function for Azure AD authentication on CloudFront distributions.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/lambda-edge-azure-auth.git`

Required inputs: `client_id`, `client_secret`, `tenant`, `redirect_uri`  
Optional: `function_name` (lambda-edge-azure-auth), `lambda_role_name`, `session_duration` (168h), `trailing_slash_redirects_enabled` (false), `simple_urls_enabled` (true)

---

### loadbalancer
**Purpose:** Creates ALB forwarding traffic to Kubernetes NodePorts (Traefik). Supports Lambda targets and auth.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/loadbalancer.git`

Required inputs: `lb_name`, `vpc_id`, `subnet_ids`, `hosts`, `rules`, `lambda_targets`  
Optional: `create_lb` (true), `target_groups` ({traefik: {port: 32080}}), `enable_logging` (false), `log_bucket`, `idle_timeout` (60)  
Outputs: `alb_arn`, `alb_security_group_id`, `dns_names`, `target_group_arns`

---

### location
**Purpose:** Manages CDN routing for multi-product deployments. Supports s3 (static) and url (backend) types.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/location.git`

Key inputs: `zone_name`, `products` (list with name, strip_prefix, paths)

```hcl
module "location" {
  source    = "git::ssh://git@git.proalpha.com/cce_mod/location.git"
  zone_name = "myproject-dev.proalpha.dev"
  products = [{
    name         = "myapp"
    strip_prefix = "myapp"
    paths = [
      { path = "/myapp/api", type = "url", backend = "api.myproject-dev.proalpha.dev" },
      { path = "/", type = "s3", backend = "ui-myapp-dev-content" }
    ]
  }]
}
```

---

### masstransit
**Purpose:** MassTransit message bus infrastructure.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/masstransit.git`

---

### metrics-server
**Purpose:** Deploys Kubernetes Metrics Server for HPA resource metrics.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/metrics-server.git`  
Optional: `replicas` (1)

---

### monitoring
**Purpose:** Monitoring stack deployment.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/monitoring.git`

---

### network-firewall
**Purpose:** Creates AWS Network Firewall with subnets, endpoints, and routing for protected subnets.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/network-firewall.git`

Required inputs: `name`, `fw_subnet_cidrs`, `protected_subnet_cidrs`, `protected_route_table_ids`, `igw_id`  
Optional: `vpc_id`, `rule_groups` ([]), `rule_order` (DEFAULT_ACTION_ORDER)  
Outputs: `firewall_endpoints` (list of {az, endpoint_id, subnet_id})

---

### pfsense
**Purpose:** Deploys pfSense firewall EC2 instance from AWS Marketplace.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/pfsense.git`

Required inputs: `subnet_id`, `allowed_management_cidrs`, `key_name`  
Optional: `instance_type` (t3a.medium), `create_key` (true), `s3_key_bucket`, `s3_key_prefix`, `ssm_param_name`

---

### pgadmin
**Purpose:** Deploys pgAdmin 4 in Kubernetes with OIDC authentication and optional persistent storage.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/pgadmin.git`

Required inputs: `url`, `config_bucket`  
Optional: `enable_pvs` (false), `servers` ([]), `wait_for_write` (nothing)  
Outputs: `db_settings`

---

### platform_project
**Purpose:** Creates project infrastructure: Bitbucket webhooks and ECR repositories.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/platform_project.git`

Required inputs: `project_name`, `bitbucket_project`, `bitbucket_repositories`, `bucket_webhook_secret`, `pipeline_host`, `ecr_repositories`, `ecr_read_access_list`  
Optional: `cost_center` (""), `bitbucket_owner` (proalpha), `webhooks_active_default` (true)

---

### platform-components-shared
**Purpose:** Terraform components for the shared account (general.shared.platform).  
**Source:** `git::ssh://git@bitbucket.org/proalpha/platform-components-shared.git//components/<component>`

Available components:
- CI/CD: ci-argocd-apps, ci-pipeline-roles, tekton-core, tekton-iam, tekton-logs, tekton-rbac
- Container Registry: ecr, ecr-extensions, ecr-registry-policies
- Platform Services: bitbucket, codeartifact, codesigning, n8n, pgadmin, renovate, sonarqube, storybook, wazuh
- Shared Infrastructure: ai-platform, aws-pre, cloudstats, cron-action, docker-credentials, iam-service-users, platform-agent, project-secrets, shared-buckets, testreports-azure-auth

---

### platform-components
**Purpose:** Platform component definitions.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/platform-components.git`

---

### privatelink
**Purpose:** AWS PrivateLink configuration.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/privatelink.git`

---

### s3-secret
**Purpose:** Loads files from S3 and creates Kubernetes secrets (avoids committing secrets to Git).  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/s3-secret.git`

Required inputs: `bucket`, `secret_file`, `name`, `namespace`  
Optional: `data_key` (null), `secret_type` (Opaque), `annotations` ({})

```hcl
module "access_token" {
  source      = "git::ssh://git@git.proalpha.com/cce_mod/s3-secret.git"
  bucket      = "my_bucket_123"
  secret_file = "secrets/secret_token.token"
  namespace   = "my_namespace"
  name        = "secret-token"
}
```

---

### security-groups
**Purpose:** Creates AWS security groups with custom ingress/egress rules.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/security-groups.git`

Required inputs: `sgroup_name`, `vpc_id`  
Optional: `rules` (default SSH), `tags` ({name: default})  
Outputs: `id`

Rule fields: `type` (ingress/egress), `pro` (tcp/udp/-1), `from`, `to`, `cidr`

```hcl
module "security-groups" {
  source      = "git::ssh://git@git.proalpha.com/cce_mod/security-groups.git"
  sgroup_name = "my-sg"
  vpc_id      = module.vpc.vpc_id
  rules = [
    { type = "ingress", pro = "tcp", to = "5432", from = "5432", cidr = "10.0.1.0/24" }
  ]
}
```

---

### ses
**Purpose:** Configures AWS SES with domain verification, DKIM, and optional cross-account domain management.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/ses.git`

Required inputs: `domain_name`  
Optional: `route53_zone_name`, `create_route53_records` (true), `enable_dns_verification` (true), `enable_mail_from_domain` (true), `mail_from_adress_domain`, `enable_cloudwatch_logging` (false), `cloudwatch_log_group_name` (/aws/ses/events), `cloudwatch_log_retention_days` (7), `configuration_set_name` (email-tracking), `ses_event_types`, `suppression_reasons` ([BOUNCE, COMPLAINT])  
Outputs: `domain_identity_arn`

---

### vpc
**Purpose:** Creates AWS VPC with subnets, NAT gateways, route tables, optional VPN gateway and flow logs.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/vpc.git`

Optional: `cidr` (10.0.0.0/16), `azs` ([]), `private_subnets` ([]), `public_subnets` ([]), `database_subnets` ([]), `elasticache_subnets` ([]), `enable_nat_gateway` (false), `single_nat_gateway` (false), `enable_vpn_gateway` (false), `enable_flow_log` (false), `enable_dns_hostnames` (true), `enable_dns_support` (true), `instance_tenancy` (default)

---

### waf
**Purpose:** Creates AWS WAFv2 Web ACL with IP whitelisting and ALB association.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/waf.git`

Required inputs: `scope` (REGIONAL/CLOUDFRONT), `rules`  
Optional: `name` (terraform-managed-wacl), `default_action` (allow), `count_only` (true), `alb_arn_list`, `whitelist_ip_set` ([]), `whitelist_eips` (false), `tags` ({})  
Outputs: `wacl_arn`

---

### yarp-ingress-controller
**Purpose:** Deploys YARP (Yet Another Reverse Proxy) ingress controller via custom Helm chart from S3.  
**Source:** `git::ssh://git@git.proalpha.com/cce_mod/yarp-ingress-controller.git`

Optional: `helm_values` ("")  
Notes: Uses Helm chart from s3://pa-private-helm-repo/ingresscontroller; deploys in `yarp` namespace; uses cert-manager

---

## Quick Reference by Category

| Category | Modules |
|----------|---------|
| **Networking** | vpc, loadbalancer, ingress-controller, yarp-ingress-controller, network-firewall, privatelink, location |
| **Compute** | eks, autoscaling-schedule, pfsense |
| **Storage** | buckets, dynamo_db, ecr |
| **Database** | aurora-db-config, elasticache, pgadmin |
| **Security** | security-groups, waf, cert-manager, s3-secret, lambda-edge-azure-auth |
| **Auth** | cognito, argocd |
| **Messaging** | ses, eventbus, masstransit |
| **Observability** | monitoring, kubecost, metrics-server, kubernetes-dashboard |
| **Platform** | platform_project, platform-components, platform-components-shared, configvariables |
| **Kubernetes** | kubernetes-cron, kubernetes-dashboard, metrics-server, ingress-controller, cert-manager |
| **Serverless** | lambda-coralogix, lambda-edge-azure-auth, aurora-db-config |
| **Automation** | awx, kubernetes-cron |
