<a name = "adding-new-org-to-existing-network-in-indy"></a>
# Adding a new validator organization in Indy

  - [Prerequisites](#prerequisites)
  - [Create Configuration File](#create-configuration-file)
  - [Run playbook](#run-playbook)

<a name = "prerequisites"></a>
## Prerequisites
To add a new organization in Indy, an existing Indy network should be running, pool and domain genesis files should be available. 

---
**NOTE**: Addition of a new organization has been tested on an existing network which is created by BAF. Networks created using other methods may be suitable but this has not been tested by BAF team.

---

**NOTE**: The guide is only for the addition of VALIDATOR Node in existing Indy network.

---

<a name = "create-configuration-file"></a>
## Create Configuration File

Refer [this guide](./indy_networkyaml.md) for details on editing the configuration file.

The `network.yaml` file should contain the specific `network.organization` details along with the genesis information and `genesis.add-org` as `true`.

---
**NOTE**: If you are adding node to the same cluster as of another node, make sure that you add the ambassador ports of the existing node present in the cluster to the network.yaml

---
For reference, sample `network.yaml` file looks like below (but always check the latest network-indy-newnode.yaml at `platforms/hyperledger-indy/configuration/samples`):

```
---
# This is a sample configuration file for hyperledger indy which can be reused for adding of new org with 2 validator nodes.
# It has 2 organizations:
# 1. organization "university" with 1 trustee, 2 stewards and 1 endorser
# 2. organization "bank" with 1 trustee, 2 stewards and 1 endorser
# It is MANDATORY to have once existing orgnization and once new organization as a Steward can add one and only one Validator Node

network:
  # Network level configuration specifies the attributes required for each organization
  # to join an existing network.
  type: indy
  version: 1.11.0                         # Supported versions 1.11.0 and 1.12.1

  #Environment section for Kubernetes setup
  env:
    type: indy             # tag for the environment. Important to run multiple flux on single cluster
    proxy: ambassador               # value has to be 'ambassador' as 'haproxy' has not been implemented for Indy
    # Must be different from all other ports specified in the rest of this network yaml
    ambassadorPorts:                # Any additional Ambassador ports can be given here, this is valid only if proxy='ambassador'
      ports: 15010,15023,15024,15025,15033,15034,15035,15043,15044,15045 # Each Client Agent uses 3 ports  # Indy does not use a port range as it creates an NLB, and only necessary ports should be opened 
    loadBalancerSourceRanges: # (Optional) Default value is '0.0.0.0/0', this value can be changed to any other IP adres or list (comma-separated without spaces) of IP adresses, this is valid only if proxy='ambassador'
    retry_count: 40                 # Retry count for the checks
    external_dns: enabled           # Should be enabled if using external-dns for automatic route configuration

  # Docker registry details where images are stored. This will be used to create k8s secrets
  # Please ensure all required images are built and stored in this registry.
  # Do not check-in docker_password.
  docker:
    url: "index.docker.io/hyperledgerlabs"
    username: "docker_username"
    password: "docker_password"

  # It's used as the Indy network name (has impact e.g. on paths where the Indy nodes look for crypto files on their local filesystem)
  name: baf 

  # Information about pool transaction genesis and domain transactions genesis
  # All the fields below in the genesis section are MANDATORY
  genesis:
    add_org: true               # Flag to denote that this will add new orgs to existing Indy network
    state: present              # must be present when add_org is true
    pool: /path/to/pool_transactions_genesis         # path where pool_transactions_genesis from existing network has been stored locally
    domain: /path/to/domain_transactions_genesis     # path where domain_transactions_genesis from existing has been stored locally

  # Allows specification of one or many organizations that will be connecting to a network.
  organizations:
  - organization:
    name: university
    type: peer
    org_status: existing # Status of the organization for the existing network, can be new / existing
    cloud_provider: aws
    external_url_suffix: indy.blockchaincloudpoc.com  # Provide the external dns suffix. Only used when Indy webserver/Clients are deployed.
    
    aws:
      access_key: "aws_access_key"            # AWS Access key
      secret_key: "aws_secret_key"            # AWS Secret key
      encryption_key: "encryption_key_id"     # AWS encryption key. If present, it's used as the KMS key id for K8S storage class encryption.
      zone: "availability_zone"               # AWS availability zone
      region: "region"                        # AWS region
    
    publicIps: ["3.221.78.194"]             # List of all public IP addresses of each availability zone from all organizations in the same k8s cluster

    # Kubernetes cluster deployment variables. The config file path has to be provided in case
    # the cluster has already been created.
    k8s:
      config_file: "/path/to/cluster_config"
      context: "kubernetes-admin@kubernetes"

    # Hashicorp Vault server address and root-token. Vault should be unsealed.
    # Do not check-in root_token
    vault:
      url: "vault_addr"
      root_token: "vault_root_token"
      
    # Git Repo details which will be used by GitOps/Flux.
    # Do not check-in git_access_token
    gitops:
      git_protocol: "https" # Option for git over https or ssh
      git_url: "https://github.com/<username>/blockchain-automation-framework.git"                 # Gitops https or ssh url for flux value files 
      branch: "develop"                   # Git branch where release is being made
      release_dir: "platforms/hyperledger-indy/releases/dev"         # Relative Path in the Git repo for flux sync per environment. 
      chart_source: "platforms/hyperledger-indy/charts"             # Relative Path where the Helm charts are stored in Git repo
      git_repo: "github.com/<username>/blockchain-automation-framework.git"           # Gitops git repository URL for git push 
      username: "git_username"                  # Git Service user who has rights to check-in in all branches
      password: "git_access_token"                  # Git Server user password
      email: "git_email"                        # Email to use in git config
      private_key: "path_to_private_key"        # Path to private key file which has write-access to the git repo (Optional for https; Required for ssh)

    # Services maps to the pods that will be deployed on the k8s cluster
    # This sample has trustee, 2 stewards and endoorser
    services:
      trustees:
      - trustee:
        name: university-trustee
        genesis: true
      stewards:
      - steward:
        name: university-steward-1
        type: VALIDATOR
        genesis: true
        publicIp: 3.221.78.194        # IP address of current organization in current availability zone
        node:
          port: 9731
          targetPort: 9731
          ambassador: 9731            # Port for ambassador service
        client:
          port: 9732
          targetPort: 9732
          ambassador: 9732            # Port for ambassador service
      - steward:
        name: university-steward-2
        type: VALIDATOR
        genesis: true
        publicIp: 3.221.78.194        # IP address of current organization in current availability zone
        node:
          port: 9741
          targetPort: 9741
          ambassador: 9741            # Port for ambassador service
        client:
          port: 9742
          targetPort: 9742
          ambassador: 9742            # Port for ambassador service
      endorsers:
      - endorser:
        name: university-endorser
        full_name: Some Decentralized Identity Mobile Services Partner
        avatar: http://university.com/avatar.png
        # public endpoint will be {{ endorser.name}}.{{ external_url_suffix}}:{{endorser.server.httpPort}}
        # Eg. In this sample http://university-endorser.indy.blockchaincloudpoc.com:15033/
        # For minikube: http://<minikubeip>>:15033
        server:
          httpPort: 15033
          apiPort: 15034
          webhookPort: 15035
  - organization:
    name: bank
    type: peer
    org_status: new # Status of the organization for the existing network, can be new / existing
    cloud_provider: aws
    external_url_suffix: indy.blockchaincloudpoc.com  # Provide the external dns suffix. Only used when Indy webserver/Clients are deployed.

    aws:
      access_key: "aws_access_key"            # AWS Access key
      secret_key: "aws_secret_key"            # AWS Secret key
      encryption_key: "encryption_key_id"     # AWS encryption key. If present, it's used as the KMS key id for K8S storage class encryption.
      zone: "availability_zone"               # AWS availability zone
      region: "region"                        # AWS region

    publicIps: ["3.221.78.194"]               # List of all public IP addresses of each availability zone from all organizations in the same k8s cluster                        # List of all public IP addresses of each availability zone

    # Kubernetes cluster deployment variables. The config file path has to be provided in case
    # the cluster has already been created.
    k8s:
      config_file: "/path/to/cluster_config"
      context: "kubernetes-admin@kubernetes"

    # Hashicorp Vault server address and root-token. Vault should be unsealed.
    # Do not check-in root_token
    vault:
      url: "vault_addr"
      root_token: "vault_root_token"

    # Git Repo details which will be used by GitOps/Flux.
    # Do not check-in git_access_token
    gitops:
      git_protocol: "https" # Option for git over https or ssh
      git_url: "https://github.com/<username>/blockchain-automation-framework.git"                   # Gitops https or ssh url for flux value files 
      branch: "develop"                     # Git branch where release is being made
      release_dir: "platforms/hyperledger-indy/releases/dev"           # Relative Path in the Git repo for flux sync per environment. 
      chart_source: "platforms/hyperledger-indy/charts"               # Relative Path where the Helm charts are stored in Git repo
      git_repo: "github.com/<username>/blockchain-automation-framework.git"             # Gitops git repository URL for git push 
      username: "git_username"                    # Git Service user who has rights to check-in in all branches
      password: "git_access_token"                    # Git Server user password
      email: "git_email"                          # Email to use in git config
      private_key: "path_to_private_key"          # Path to private key file which has write-access to the git repo (Optional for https; Required for ssh)

    # Services maps to the pods that will be deployed on the k8s cluster
    # This sample has trustee, 2 stewards and endoorser
    services:
      trustees:
      - trustee:
        name: bank-trustee
        genesis: true
      stewards:
      - steward:
        name: bank-steward-1
        type: VALIDATOR
        genesis: true
        publicIp: 3.221.78.194    # IP address of current organization in current availability zone
        node:
          port: 9711
          targetPort: 9711
          ambassador: 9711        # Port for ambassador service
        client:
          port: 9712
          targetPort: 9712
          ambassador: 9712        # Port for ambassador service
      - steward:
        name: bank-steward-2
        type: VALIDATOR
        genesis: true
        publicIp: 3.221.78.194    # IP address of current organization in current availability zone
        node:
          port: 9721
          targetPort: 9721
          ambassador: 9721        # Port for ambassador service
        client:
          port: 9722
          targetPort: 9722
          ambassador: 9722        # Port for ambassador service
      endorsers:
      - endorser:
        name: bank-endorser
        full_name: Some Decentralized Identity Mobile Services Provider
        avatar: http://bank.com/avatar.png


```
Following items must be added/updated to the network.yaml used to add new organizations


| Field       | Description                                              |
|-------------|----------------------------------------------------------|
| genesis.add_org | Flag to denote that this will add new orgs to existing Indy network. Must be `true` in this case |
| genesis.state  | Must be `present` for add org  |
| genesis.pool | Path to Pool Genesis file of the existing Indy network.|
| genesis.domain | Path to Domain Genesis file of the existing Indy network.|


Also, ensure that `organization.org_status` is set to `existing` for existing org and `new` for new org.

<a name = "run-playbook"></a>
## Run playbook

The [add-new-organization.yaml](https://github.com/hyperledger-labs/blockchain-automation-framework/tree/main/platforms/shared/configuration/add-new-organization.yaml) playbook is used to add a new organization to the existing network. This can be done using the following command

```
ansible-playbook platforms/shared/configuration/add-new-organization.yaml --extra-vars "@path-to-network.yaml" 
```
