# cosmovisor-poa-test

```shell
git clone https://github.com/liftedinit/manifest-ledger.git
cd manifest-ledger
git checkout v0.0.1-alpha.5
make build
./build/manifestd version
v0.0.1-alpha.5
```

Cleanup `~/.manifest` (never do this in a production environment!)

```shell
rm -rf ~/.manifest
```

Initialize and configure the chain (never use `--overwrite` in a production environment!)

```shell
./build/manifestd init test --chain-id test --default-denom upoa --overwrite

./build/manifestd config set client chain-id test
./build/manifestd config set client keyring-backend test
./build/manifestd config set client broadcast-mode sync
```

Create the validation and self-delegate
```shell
./build/manifestd keys add val

# Set the POA Admin to the Cosmos Governance module address
# This is required in order to execute governance proposal (without chain modification, default POA module behavior)
# Full control to the POA Admin will require modifying the modules authority string as described in 
# https://github.com/strangelove-ventures/poa/blob/da5cc7991f27eafd0917d1af006ea8f1a3e3c46e/INTEGRATION.md?plain=1#L237
cat <<< $(jq '.app_state.poa.params.admins = ["manifest10d07y265gmmuvt4z0w9aw880jnsr700jmq3jzm"]' $HOME/.manifest/config/genesis.json) > $HOME/.manifest/config/genesis.json

# Set the voting period to 20s
cat <<< $(jq '.app_state.gov.params.voting_period = "20s"' $HOME/.manifest/config/genesis.json) > $HOME/.manifest/config/genesis.json

# Set the deposit period to 30s
cat <<< $(jq '.app_state.gov.params.max_deposit_period = "30s"' $HOME/.manifest/config/genesis.json) > $HOME/.manifest/config/genesis.json

# Set the expedited voting period to 10s (required to be less than the voting period)
cat <<< $(jq '.app_state.gov.params.expedited_voting_period = "10s"' $HOME/.manifest/config/genesis.json) > $HOME/.manifest/config/genesis.json

# Lower the minimum deposit to 10000upoa
cat <<< $(jq '.app_state.gov.params.min_deposit = [{"denom": "upoa", "amount": "100000"}]' $HOME/.manifest/config/genesis.json) > $HOME/.manifest/config/genesis.json

./build/manifestd genesis add-genesis-account val 10000000upoa --keyring-backend test
./build/manifestd genesis gentx val 1000000upoa --chain-id test
./build/manifestd genesis collect-gentxs
```

Prepare Cosmovisor

```shell
export DAEMON_NAME=manifestd
export DAEMON_HOME=$HOME/.manifest
export DAEMON_ALLOW_DOWNLOAD_BINARIES=false
export DAEMON_RESTART_AFTER_UPGRADE=true
export POA_ADMIN_ADDRESS=manifest10d07y265gmmuvt4z0w9aw880jnsr700jmq3jzm
```

Initialize and start Cosmovisor

```shell
cosmovisor init ./build/manifestd
cosmovisor run start
```

We now have a single validator node running! Let's now perform an upgrade. 

Open a new terminal and change directory to the `manifest-ledger` repository. 

Re-export the environment variables

```shell
export DAEMON_NAME=manifestd
export DAEMON_HOME=$HOME/.manifest
export DAEMON_ALLOW_DOWNLOAD_BINARIES=false
export DAEMON_RESTART_AFTER_UPGRADE=true
export POA_ADMIN_ADDRESS=manifest10d07y265gmmuvt4z0w9aw880jnsr700jmq3jzm
```

Build the new version of the binary

```shell
git remote add fmorency https://github.com/fmorency/manifest-ledger.git
git fetch fmorency
git checkout fake-upgrade
make build
./build/manifestd version
v0.0.1-fu.1
```

Prepare the upgrade (with governance proposal)

```shell
cosmovisor add-upgrade supercalifragilisticexpialidocious ./build/manifestd
./build/manifestd tx upgrade software-upgrade supercalifragilisticexpialidocious --title upgrade --summary upgrade --upgrade-height [SOME_HEIGHT] --upgrade-info "{}" --deposit 100000upoa --no-validate --from val --yes
./build/manifestd tx gov vote 1 yes --from val --yes

# Wait for the voting period to pass

./build/manifestd q upgrade plan
plan:
  height: "[SOME_HEIGHT]"
  info: '{}'
  name: supercalifragilisticexpialidocious
  time: "0001-01-01T00:00:00Z"
```

One can also skip the governance proposal altogether and force an upgrade

NOTE: There is a bug in `cosmovisor` v1.5.0 which applies the upgrade immediately instead of at the specified height.
See https://github.com/cosmos/cosmos-sdk/issues/19227

```shell
cosmovisor add-upgrade supercalifragilisticexpialidocious ./build/manifestd --upgrade-height [SOME_HEIGHT]
```

Finally, one can auto-download the binary by setting `DAEMON_ALLOW_DOWNLOAD_BINARIES=true`, removing the `cosmovisor add-upgrade` command and adjusting the `--upgrade-info` field to include the URL and hash of the binary.

```shell
./build/manifestd tx upgrade software-upgrade supercalifragilisticexpialidocious --title upgrade --summary upgrade --upgrade-height [SOME_HEIGHT] --upgrade-info '{"binaries": {"linux/amd64": "https://github.com/fmorency/test-migration-auto-dl/releases/download/v0.0.1/manifestd.zip?checksum=sha256:c79168cb6dadad8c2d292ca02054ce6354c4d16e085f868659021c4a2ee58ca0"}}' --deposit 100000upoa --no-validate --from val --yes
./build/manifestd tx gov vote 1 yes --from val --yes

# Wait for the voting period to pass

./build/manifestd q upgrade plan
plan:
  height: "[SOME_HEIGHT]"
  info: '{}'
  name: supercalifragilisticexpialidocious
  time: "0001-01-01T00:00:00Z"
```
