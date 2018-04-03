# Provisioning a CA and Generating TLS Certificates

In this lab you will provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) using CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), then use it to bootstrap a Certificate Authority, and generate TLS certificates for the following components: etcd, kube-apiserver, kubelet, and kube-proxy.

## Certificate Authority

In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates.

Create the CA configuration file:

```
mkdir ../$ENV-ca
cd ../$ENV-ca

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

Create the CA certificate signing request:

```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "AU",
      "L": "Sydney",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "NSW"
    }
  ]
}
EOF
```

Generate the CA certificate and private key:

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Results:

```
ca-key.pem
ca.pem
```

## Client and Server Certificates

In this section you will generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes `admin` user.

### The Admin Client Certificate

Create the `admin` client certificate signing request:

```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "AU",
      "L": "Sydney",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "NSW"
    }
  ]
}
EOF
```

Generate the `admin` client certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

Results:

```
admin-key.pem
admin.pem
```

### The Kubelet and kube-proxy Client Certificates

Kubernetes uses a [special-purpose authorization mode](https://kubernetes.io/docs/admin/authorization/node/) called Node Authorizer, that specifically authorizes API requests made by [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet). Instead of using certificate based authorization we are going to use [webhook mode](https://kubernetes.io/docs/admin/authorization/webhook/), so we don't need worker nodes certificates.


### The Kubernetes API Server Certificate

The Load Balancer URL will be included in the list of subject alternative names for the Kubernetes API Server certificate. This will ensure the certificate can be validated by remote clients. We had it stored in LB_URL environment variable in [Provisioning Network Resources](03-network-resources.md) section.

Create the Kubernetes API Server certificate signing request:

```
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "AU",
      "L": "Sydney",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "NSW"
    }
  ]
}
EOF
```

Generate the Kubernetes API Server certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${LB_URL},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

Results:

```
kubernetes-key.pem
kubernetes.pem
```

## Distribute the Client and Server Certificates

We will use [SSM parameter store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html) to save and get certificates and private keys on instances. Certificates will be stored as a plain text values and private keys will be encrypted with KMS. For simplicity we are going to use account default KMS key ant restrict access by IAM policies, however, in production environment it would be preferable to use [a custom KMS key](https://docs.aws.amazon.com/kms/latest/developerguide/services-parameter-store.html).

Create plain text parameters for CA and master's certificates

```bash
aws ssm put-parameter --name "/$ENV/ca/ca" --value "$(cat ca.pem)" --type String
aws ssm put-parameter --name "/$ENV/ca/master" --value "$(cat kubernetes.pem)" --type String
```

Create encrypted parameter for master's private key

```bash
aws ssm put-parameter --name "/$ENV/ca/private/master" --value "$(cat kubernetes-key.pem)" --type SecureString
```

In order to get and restrict access from instances we use the following IAM policy for master nodes (it is created in the section [Provisioning compute resources](04-compute-resources.md)):
```json
{
  "Version" : "2012-10-17",
  "Statement":
    [
      {
          "Effect": "Allow",
          "Action": [
              "ssm:GetParameters"
          ],
          "Resource": [
              "arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:parameter/${EnvironmentName}/*"
          ]
      },
      {
          "Effect":"Allow",
          "Action":[
            "kms:Decrypt"
          ],
          "Resource":[
            "${CMK}"
          ]
      }
    ]
  }
```

Next: [Generating Kubernetes Configuration Files for Authentication](06-kubernetes-configuration-files.md)
