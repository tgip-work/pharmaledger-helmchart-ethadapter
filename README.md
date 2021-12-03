# pharmaledger-helmchart-ethadapter

Helm Chart for Ethereum Adapter Service

## Requirements

- [helm 3](https://helm.sh/docs/intro/install/)
- These mandatory configuration values:
  - RPC Address - The URL of the quorum node, e.g. `http://quorum-node-0-rpc:8545`
  - Smart Contract Address - The address of the smart contract, e.g. `0x1783aBc71903919382EFca91`
  - Smart Contract Abi
  - Org Account JSON - The confidential private key and address in JSON format, e.g. `{"privateKey":"0x1234567890abcdef", "address":"0x0987654321AbCdEf"}`

## Installation

### Quick install with internal service of type ClusterIP

By default, this helm chart installs the Ethereum Adapter Service at an internal ClusterIP Service listening at port 3000.
This is to prevent exposing the service to the internet by accident!

It is recommended to put non-sensitive configuration values in an configuration file and pass sensitive/secret values via commandline.

1. Create configuration file *my-config.yaml*

    ```yaml
    config:
      rpcAddress: "rpcAddress_value"
      smartContractAddress: "smartContractAddress_value"
      smartContractAbi: "smartContractAbi_value"
    ```

2. Install via helm to namespace `default` either by passing sensitive *Org Account JSON* value in JSON format as escaped string

    ```bash
    helm upgrade my-release-name ./ethadapter \
        --install \
        --values my-config.yaml \
        --set-string secrets.orgAccountJson="\{ \"key\": \"value\" \}"
    ```

3. or pass sensitive *Org Account JSON* value in JSON format as base64 encoded string

    ```bash
    helm upgrade my-release-name ./ethadapter \
        --install \
        --values my-config.yaml \
        --set-string secrets.orgAccountJsonBase64="eyAia2V5IjogInZhbHVlIiB9"
    ```

### Expose Service via LoadBalancer by http (unencrypted, not https)

In order to expose the service **directly** by an Load Balancer, just **add** `service.type` with value `LoadBalancer` to your config file (in order to override the default value which is `ClusterIP`).

Configuration file *my-config.yaml*

```yaml
service:
  type: LoadBalancer

config:
  rpcAddress: "rpcAddress_value"
  smartContractAddress: "smartContractAddress_value"
  smartContractAbi: "smartContractAbi_value"
```

Additionally you can customize the port:

```yaml
service:
  type: LoadBalancer
  port: 4567

# further config
```

### Expose Service via Ingress and AWS Load Balancer Controller

Note: You need the [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) installed and configured properly.

1. Enable ingress
2. Add host and path */**
3. Add annotations for AWS LB Controller
4. A SSL certificate at AWS Certificate Manager (either for the hostname, here `ethadapter.mydomain.com` or wildcard `*.mydomain.com`)

Configuration file *my-config.yaml*

```yaml
ingress:
  enabled: true
  hosts:
    - host: ethadapter.mydomain.com
      # Path must be /* for ALB to match all paths
      paths: ["/*"]
  # For full list of annotations for AWS LB Controller, see https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/
  annotations:
    # The ARN of the existing SSL Certificate at AWS Certificate Manager
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:REGION:ACCOUNT_ID:certificate/CERTIFICATE_ID
    # The name of the ALB group, can be used to configure a single ALB by multiple ingress objects
    alb.ingress.kubernetes.io/group.name: default
    # Specifies the HTTP path when performing health check on targets.
    alb.ingress.kubernetes.io/healthcheck-path: /check
    # Specifies the port used when performing health check on targets. 
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    # Specifies the HTTP status code that should be expected when doing health checks against the specified health check path.
    alb.ingress.kubernetes.io/success-codes: "200"
    # Listen on HTTPS protocol at port 3000 at the ALB
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":3000}]'
    # Allow access from a specific IP address only, e.g. from the NAT Gateway of your EPI Cluster
    alb.ingress.kubernetes.io/inbound-cidrs: 8.8.8.8/32 
    # Use internet facing
    alb.ingress.kubernetes.io/scheme: internet-facing
    # Use most current (as of Dec 2021) encryption ciphers
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-Ext-2018-06
    # Use target type IP which is the case if the service type is ClusterIP
    alb.ingress.kubernetes.io/target-type: ip
    # Let AWS LB Controller handle the ingress
    kubernetes.io/ingress.class: alb

config:
  rpcAddress: "rpcAddress_value"
  smartContractAddress: "smartContractAddress_value"
  smartContractAbi: "smartContractAbi_value"
```

### Additional helm options

Run `helm upgrade --helm` for full list of options.

1. Install to other namespace

    You can install into other namespace than `default` by setting the `--namespace` parameter, e.g.

    ```bash
    helm upgrade my-release-name ./ethadapter \
        --install \
        --namespace=my-namespace \
        --values my-config.yaml \
        --set-string secrets.orgAccountJson="\{ \"key\": \"value\" \}"
    ```

2. Wait until installation has finished successfully and the deployment is up and running.

    Provide the `--wait` argument and time to wait (default is 5 minutes) via `--timeout`

    ```bash
    helm upgrade my-release-name ./ethadapter \
        --install \
        --wait --timeout=600s \
        --values my-config.yaml \
        --set-string secrets.orgAccountJson="\{ \"key\": \"value\" \}"
    ```

## Test

See [Helm Debugging Templates](https://helm.sh/docs/chart_template_guide/debugging/)

```bash
mkdir -p ./testresults
rm -rf ./testresults/*
# https://github.com/helm/helm/issues/5618
echo "Test 1 with default values and secret passed as String"
helm template test-ethadapter ./ethadapter --values ./tests/data/default.yaml --set-string secrets.orgAccountJson="\{ \"key\": \"value\" \}" > ./tests/results/test_1.yaml

echo "Test 2 with default values and secret passed as base64 encoded String"
helm template test-ethadapter ./ethadapter --values ./tests/data/default.yaml --set-string secrets.orgAccountJsonBase64="eyAia2V5IjogInZhbHVlIiB9" > ./tests/results/test_2.yaml

echo "Test 3 with LoadBalancer"
helm template test-ethadapter ./ethadapter --values ./tests/data/loadbalancer.yaml --set-string secrets.orgAccountJsonBase64="eyAia2V5IjogInZhbHVlIiB9" > ./tests/results/test_3.yaml

echo "Test 4 with Ingress via AWS LB Controller"
helm template test-ethadapter ./ethadapter --values ./tests/data/aws_lb_controller_ingress.yaml --set-string secrets.orgAccountJsonBase64="eyAia2V5IjogInZhbHVlIiB9" > ./tests/results/test_4.yaml
```
