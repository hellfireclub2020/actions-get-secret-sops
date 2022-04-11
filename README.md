# actions-get-secret-sops
GitHub Action get secret with specific key from encrypted SOPS yaml file

## Features
- Only support Azure Key Vault

## Usage

### Azure Key Vault

```json
{
  "appId": "<some-uuid>",
  "displayName": "my-service-principal-name",
  "password": "<some-uuid>",
  "tenant": "<tenant-id>"
}
```

```yaml
steps:
  - uses: mildronize/actions-get-secret-sops/azure@main
    id: sops
    with:
      path: "azure.enc.yaml"                    # Encrypted SOPS yaml path
      property-path: ".property"                # jq/yq expression syntax for getting a particular value
      decrypting-key: ${{ secrets.credential }} # A credential using to decrypt a Encrypted SOPS yaml file
      sops-version: '3.7.2'

  - run: echo "${{ steps.sops.outputs.secret }}"
```


# SOPS 101


## Encrypt using Age

```
sops --encrypt --age [AGE KEY] test.yaml > test.enc.yaml
```

## Encrypt with Azure Key Vault

```bash
az ad sp create-for-rbac -n "sp_sops_github_action" --role Contributor --scopes /subscriptions/[Subscription ID]/resourceGroups/[resource_Group_name]/providers/Microsoft.KeyVault/vaults/[vault_name]

{
  "appId": "<some-uuid>",
  "displayName": "my-keyvault-sp",
  "name": "http://my-keyvault-sp",
  "password": "<some-uuid>",
  "tenant": "<tenant-id>"
}

export AZURE_CLIENT_ID="appId"
export AZURE_TENANT_ID="tenant"
export AZURE_CLIENT_SECRET="password"
```

```bash
az account set --subscription "XXXX"
az group create --name rg-common --location "Central US"
# Create a Vault, a key, and give the service principal access:
az keyvault create --name "kv-github-action" --resource-group rg-common --location "Central US"

az keyvault key create --name "sops-key" --vault-name "kv-github-action" --protection software --ops encrypt decrypt

az keyvault set-policy --name "kv-github-action" --resource-group "rg-common" --spn $AZURE_CLIENT_ID \
        --key-permissions encrypt decrypt

# Read the key id:
az keyvault key show --name "sops-key" --vault-name "kv-github-action" --query key.kid

https://sops.vault.azure.net/keys/sops-key/some-string

# Encrypt
sops --encrypt --azure-kv https://sops.vault.azure.net/keys/sops-key/some-string test.yaml > test.enc.yaml
# Decrypt
sops --decrypt test.enc.yaml
```