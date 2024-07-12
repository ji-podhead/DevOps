# DevOps

| [Guides](https://github.com/ji-podhead/DevOps/tree/main/docs/guides)  |  [Projects](https://github.com/ji-podhead/DevOps/)  |

---

## Guides 

| [TerrForm & Vault]() | [Terraform & Proxmox]() | 


## Terraform & Vault

### set up vault secrets and auth

***create a policy called `terraform`***
```bash
path "*" {
  capabilities = ["list", "read"]
}
path "auth/token/create" {
capabilities = ["create", "read", "update", "list"]
}# Permit creating a new token that is a child of the one given
path "auth/token/create"
{
  capabilities = ["update"]
}

# Permit managing the lifecycle of the gcp secrets engine mount
path "sys/mounts/gcp"
{
  capabilities = ["read", "create", "update", "delete"]
}

# Permit reading tune metadata of the gcp secrets engine
path "sys/mounts/gcp/tune"
{
  capabilities = ["read"]
}

# Permit managing the lifecycle of the gcp secrets engine configuration
path "gcp/config"
{
  capabilities = ["create", "update", "read"]
}
```

***enable approle auth***

```bash
$ vault auth enable approle
```
***create an approle***
```bash
$ vault write auth/approle/role/terraform \
>   secret_id_ttl=0 \
>   token_num_uses=0 \
>   token_ttl=0 \
>   token_max_ttl=0 \
>   secret_id_num_uses=0
Success! Data written to: auth/approle/role/terraform
```
***assignt the policy to our approle***

```bash
$ vault write auth/approle/role/approle policies=terraform-policy
```
***get role id***

```bash
$ vault read auth/approle/role/terraform/role-id
Key        Value
---        -----
role_id    <role_id>
```
***get secret id***

```bash
$ vault write -f auth/approle/role/terraform/secret-id
Key                   Value
---                   -----
secret_id             <secret id>
secret_id_accessor    b18...
secret_id_num_uses    0
secret_id_ttl         0s
```

***create a token***

```bash
$ vault token create -policy=terraform-policy
Key                  Value
---                  -----
token                <your token>
token_accessor       Rd0...
token_duration       768h
token_renewable      true
token_policies       ["default" "terraform-policy"]
identity_policies    []
policies             ["default" "terraform-policy"]
````
***create a secret***

![secret](https://github.com/ji-podhead/DevOps/blob/main/docs/terraform&vault/vaul_secret.png?raw=true)
> ***this image shows a wrong path, the engine uses v2, so dont be confused***

***login (test)***

```bash
$ vault login <your token>
WARNING! The VAULT_TOKEN environment variable is set! The value of this
variable will take precedence; if this is unwanted please unset VAULT_TOKEN or
update its value accordingly.

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                <your token>
token_accessor       rzN...
token_duration       767h59m28s
token_renewable      true
token_policies       ["default" "terraform-policy"]
identity_policies    []
policies             ["default" "terraform-policy"]
```

***get a secret (test)***

```bash
§ vault kv get -mount="keyvalue" "terraform/github"
========= Secret Path =========
keyvalue/data/terraform/github

======= Metadata =======
Key                Value
---                -----
created_time       2024-07-11T14:37:20.156254741Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

======== Data ========
Key             Value
---             -----
github_token    <your secret>
```

***use the secret in terraform***

```hcl
provider "vault" {
  address = "http://127.0.0.1:8200"
token = var.vault_token                 # the token we used before via manual login
  skip_child_token = true 
#  NOT WORKING!!!
#   auth_login {
#    method = "approle"
#    path = "auth/approle/login"
#    parameters = {
#      role_id   = var.vault_role_id
#      secret_id = var.vault_role_secret_id
#    }
#    
#  }
#
}
# NOT WORKING BECAUSE OF V2
#  data "vault_kv_secret" "proxmox_api_token" {
#    path = "/keyvalue/data/terraform/github"
#  }
  data "vault_kv_secret_v2" "proxmox" {
  mount = "keyvalue"
  name  = "terraform/proxmox"
}
  output "test" {
    value = data.vault_kv_secret_v2.proxmox.data[<your secret key>]
      sensitive = true
  }
```

***test and apply it***


```bash
...
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

test = <sensitive>
```
> - this way  we can test our vault and terraform configs
