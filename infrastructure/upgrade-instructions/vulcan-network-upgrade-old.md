# Vulcan Network Upgrade (OLD)

## Network Upgrade Instructions From `secret-1` To `secret-2` <a href="#network-upgrade-instructions-from-secret-1-to-secret-2" id="network-upgrade-instructions-from-secret-1-to-secret-2"></a>

⚠️ Please read carefully before you begin the upgrade.

* Network Upgrade Instructions from `secret-1` to `secret-2`
* Validators
  * Prepare your `secret-1` validator to halt after block #1,246,400
  * Install the new binaries on your SGX machine
  * Migrate your validator's signing key
  * Migrate your validator's wallet
  * Set up your SGX machine and become a `secret-2` validator
* In case of an upgrade failure

## Validators <a href="#validators" id="validators"></a>

All coordination efforts will be done in the [mainnet-validators](https://chat.scrt.network/channel/mainnet-validators) channel in the Secret Network Discord.

> ⚠️ Don't delete your `secret-1` machine, as we might have to relaunch it.

> ⚠️ If your SGX machine has a previous `secretnetwork` installation on it (e.g. from the testnet), you will need to remove it before you continue:

```bash
cd ~
sudo systemctl stop secret-node
secretd unsafe-reset-all
secretd reset-enclave
secretd init-enclave --reset
sudo apt purge -y secretnetwork
rm -rf ~/.secretcli/*
rm -rf ~/.secretd/*
rm -rf ~/.sgx_secrets/*
```

You're probably familiar with [SGX](../../overview-ecosystem-and-technology/techstack/privacy-technology/intel-sgx/why-sgx.md) by now:

* [Setup SGX](../node-runners/node-setup/install-sgx.md)
* [Verify SGX](broken-reference)

### Prepare Your `secret-1` Validator To Halt After Block #1,246,400 <a href="#_1-prepare-your-secret-1-validator-to-halt-after-block-1-246-400" id="_1-prepare-your-secret-1-validator-to-halt-after-block-1-246-400"></a>

On the old machine (`secret-1`):

```bash
perl -i -pe 's/^halt-height =.*/halt-height = 1246400/' ~/.secretd/config/app.toml

sudo systemctl restart secret-node
```

### 2. Install The New Binaries On Your SGX Machine <a href="#_2-install-the-new-binaries-on-your-sgx-machine" id="_2-install-the-new-binaries-on-your-sgx-machine"></a>

On the new SGX machine (`secret-2`):

```bash
cd ~

wget "https://github.com/scrtlabs/SecretNetwork/releases/download/v1.0.0/secretnetwork_1.0.0_amd64.deb"

echo "87aee80a112f429db0b8b6703b1eba33accea0f08af9e65339d14d92a5186b24 secretnetwork_1.0.0_amd64.deb" | sha256sum --check

sudo apt install -y ./secretnetwork_1.0.0_amd64.deb

secretd init "$MONIKER" --chain-id secret-2
```

### 3. Migrate Your Validator's Signing Key <a href="#_3-migrate-your-validator-s-signing-key" id="_3-migrate-your-validator-s-signing-key"></a>

Copy your `~/.secretd/config/priv_validator_key.json` from the old machine (`secret-1`) to the new SGX machine (`secret-2`) at the same location.

### 4. Migrate Your Validator's Wallet <a href="#_4-migrate-your-validator-s-wallet" id="_4-migrate-your-validator-s-wallet"></a>

Export the self-delegator wallet from the old machine (`secret-1`) and import to the new SGX machine (`secret-2`).

On the old machine (`secret-1`) use `secretcli keys export "$YOUR_KEY_NAME"`.\
On the new SGX machine (`secret-2`) use `secretcli keys import "$YOUR_KEY_NAME" "$FROM_FILE_NAME"`

> **Notes:**
>
> 1. If you're recovering the wallet using `secretcli keys add "$YOUR_KEY_NAME" --recover` you should also use `--hd-path "44'/118'/0'/0/0"`.
> 2. If the wallet is stored on a Ledger device, use `--legacy-hd-path` when importing it with `secretcli keys add`.

### 5. Set Up Your SGX Machine And Become A `secret-2` Validator <a href="#_5-set-up-your-sgx-machine-and-become-a-secret-2-validator" id="_5-set-up-your-sgx-machine-and-become-a-secret-2-validator"></a>

On the new SGX machine (`secret-2`):

```bash
cd ~

secretd init $MONIKER --chain-id secret-2

wget -O ~/.secretd/config/genesis.json "https://github.com/scrtlabs/SecretNetwork/releases/download/v1.0.0/genesis.json"

echo "4ca53e34afed034d16464d025291fe16a847c9aca0a259f9237413171b19b4cf .secretd/config/genesis.json" | sha256sum --check

secretd validate-genesis

secretd init-enclave

PUBLIC_KEY=$(secretd parse attestation_cert.der 2> /dev/null | cut -c 3-)
echo $PUBLIC_KEY

secretcli config chain-id secret-2
secretcli config node tcp://secret-2.node.enigma.co:26657
secretcli config trust-node true
secretcli config output json
secretcli config indent true

secretcli tx register auth ./attestation_cert.der --from "$YOUR_KEY_NAME" --gas-prices 0.0125uscrt

SEED=$(secretcli query register seed "$PUBLIC_KEY" | cut -c 3-)
echo $SEED

secretcli query register secret-network-params

mkdir -p ~/.secretd/.node

secretd configure-secret node-master-cert.der "$SEED"

perl -i -pe 's/persistent_peers =.*/persistent_peers = "bee0edb320d50c839349224b9be1575ca4e67948\@secret-2.node.enigma.co:26656"/' ~/.secretd/config/config.toml

sudo systemctl enable secret-node

sudo systemctl start secret-node # (Now your new node is live and catching up)

secretcli config node tcp://localhost:26657
```

Now wait until you're done catching up. This is fast.\
Once the following command outputs `true` you can continue:

```bash
watch 'secretcli status | jq ".sync_info.catching_up == false"'
```

Once your node is done catching up, you can unjail your validator:

```bash
secretcli tx slashing unjail --from "$YOUR_KEY_NAME" --gas-prices 0.0125uscrt
```

You’re now a validator in `secret-2`! 🎉

To make sure your validator is unjailed, look for it in here:

```bash
secretcli q staking validators | jq -r '.[] | select(.status == 2) | .description.moniker'
```

([Ref for testnet instructions](https://docs.scrt.network/testnet/run-full-node-testnet.html))

## In Case Of An Upgrade Failure <a href="#in-case-of-an-upgrade-failure" id="in-case-of-an-upgrade-failure"></a>

If after a few hours the Enigma team announces on the chat that the upgrade failed, we will relaunch `secret-1`.

1.  On the old machine (`secret-1`):

    ```
    perl -i -pe 's/^halt-height =.*/halt-height = 0/' ~/.secretd/config/app.toml
    as
    sudo systemctl restart secret-node
    ```
2. Wait for 67% of voting power to come back online.
