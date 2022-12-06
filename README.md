# zkSync2 client sdk

## Contents
- [Getting started](#getting-started)
- [Provider](#provider-zksyncbuilder)
- [Account](#account)
- [Signer](#signer)
- [Contract interfaces](#contract-interfaces)
- [Examples](#examples)


### Getting started

#### Requirements
| Tool  | Required  |
|-------|-----------|
| python| >= 3.8    |
| package manager| pip |

### how to install

```console
pip install zksync2
```


### Provider (zkSyncBuilder)


#### Design
ZkSync 2.0 is designed with the same styling as web3.<br>
It defines the zksync module based on Etherium and extends it with zkSync-specific methods.<br>

#### How to construct
For usage, there is `ZkSyncBuilder` that returns a Web3 object with an instance of zksync module.<br>
Construction only needs the URL to the zkSync blockchain.

Example:
```python
from zksync2.module.module_builder import ZkSyncBuilder
...
web3 = ZkSyncBuilder.build("ZKSYNC_NET_URL")
```

#### Module parameters and methods

ZkSync module attributes:

|  Attribute | Description |
|------------|-------------|
|chain_id    | Returns an integer value for the currently configured "ChainId" |
|gas_price   | Returns the current gas price in Wei |


ZkSync module methods:

|  Method | Parameters | Return value |Description |
|---------|------------|--------------|------------|
|zks_estimate_fee | zkSync Transaction | Fee structure | Gets Fee for ZkSync transaction|
|zks_main_contract | - | Address of main contract | Return address of main contract |
|zks_get_confirmed_tokens | from, limit | List[Token]| Returns all tokens in the set range by global index|
|zks_l1_chain_id | - | ChainID | Return etherium chain ID|
|zks_get_all_account_balances| Address | Dict[str, int] | Return dictionary of token address and its value |
|zks_get_bridge_contracts | - | BridgeAddresses | Returns addresses of all bridge contracts that are interacting with L1 layer|
| eth_estimate_gas | Transaction | estimated gas | Overloaded method of eth_estimate_gas for ZkSync transaction gas estimation |
| wait_for_transaction_receipt | Tx Hash and optional timeout and tries | TxReceipt| Waits for the transaction to be included into block by its hash and returns its reciept. Optional arguments are `timeout` and `poll_latency` in seconds|


### Account

#### what is this?
Account incapsulate private key and, frequently based on it, the unique user identifier in the network.<br> This unique identifier also mean by wallet address.

#### Account contruction

ZkSync2 Python SDK account is compatible with `eth_account` package
In most cases user has its private key and gets account instance by using it.

Example:
```python
from eth_account import Account
from eth_account.signers.local import LocalAccount
...
account: LocalAccount = Account.from_key(self.PRIVATE_KEY)

```

The base property that is used directly of account is: `Account.address`


### Signer






### Examples

#### Deposit funds
This is example how to deposit from Ethereum account to ZkSync account:

```
from web3 import Web3
from web3.middleware import geth_poa_middleware
from eth_account import Account
from eth_account.signers.local import LocalAccount
from zksync2.manage_contracts.gas_provider import StaticGasProvider
from zksync2.module.module_builder import ZkSyncBuilder
from zksync2.core.types import Token
from zksync2.provider.eth_provider import EthereumProvider


def deposit():
    URL_TO_ETH_NETWORK = "https://goerli.infura.io/v3/25be7ab42c414680a5f89297f8a11a4d"
    ZKSYNC_NETWORK_URL = "https://zksync2-testnet.zksync.dev"

    eth_web3 = Web3(Web3.HTTPProvider(URL_TO_ETH_NETWORK))
    eth_web3.middleware_onion.inject(geth_poa_middleware, layer=0)
    zksync_web3 = ZkSyncBuilder.build(ZKSYNC_NETWORK_URL)
    account: LocalAccount = Account.from_key('YOUR_PRIVATE_KEY')
    gas_provider = StaticGasProvider(Web3.toWei(1, "gwei"), 555000)
    eth_provider = EthereumProvider.build_ethereum_provider(zksync=zksync_web3,
                                                            eth=eth_web3,
                                                            account=account,
                                                            gas_provider=gas_provider)
    tx_receipt = eth_provider.deposit(Token.create_eth(),
                                      eth_web3.toWei("YOUR_AMOUNT_OF_ETH", "ether"),
                                      account.address)
    print(f"tx status: {tx_receipt['status']}")


if __name__ == "__main__":
    deposit()

```


#### Check balance

After depositing there could be needed to check the account balance under ZkSync network:

```
from eth_account import Account
from eth_account.signers.local import LocalAccount
from zksync2.module.module_builder import ZkSyncBuilder
from zksync2.core.types import EthBlockParams


def get_account_balance():
    ZKSYNC_NETWORK_URL: str = 'https://'
    account: LocalAccount = Account.from_key('YOUR_PRIVATE_KEY')
    zksync_web3 = ZkSyncBuilder.build(ZKSYNC_NETWORK_URL)
    zk_balance = zksync_web3.zksync.get_balance(account.address, EthBlockParams.LATEST.value)
    print(f"ZkSync balance: {zk_balance}")


if __name__ == "__main__":
    get_account_balance()

```

#### Transfer

Here is example how to transfer funds in ZkSync network

```
from eth_typing import HexStr
from web3 import Web3
from zksync2.module.request_types import create_function_call_transaction
from zksync2.module.module_builder import ZkSyncBuilder
from zksync2.core.types import ZkBlockParams
from eth_account import Account
from eth_account.signers.local import LocalAccount

from zksync2.signer.eth_signer import PrivateKeyEthSigner
from zksync2.transaction.transaction712 import Transaction712


def transfer_to_self():
    ZKSYNC_NETWORK_URL: str = 'https://'
    account: LocalAccount = Account.from_key('YOUR_PRIVATE_KEY')
    zksync_web3 = ZkSyncBuilder.build(ZKSYNC_NETWORK_URL)
    chain_id = zksync_web3.zksync.chain_id
    signer = PrivateKeyEthSigner(account, chain_id)

    nonce = zksync_web3.zksync.get_transaction_count(account.address, ZkBlockParams.COMMITTED.value)
    tx = create_function_call_transaction(from_=account.address,
                                          to=account.address,
                                          ergs_price=0,
                                          ergs_limit=0,
                                          data=HexStr("0x"))
    estimate_gas = zksync_web3.zksync.eth_estimate_gas(tx)
    gas_price = zksync_web3.zksync.gas_price

    print(f"Fee for transaction is: {estimate_gas * gas_price}")

    tx_712 = Transaction712(chain_id=chain_id,
                            nonce=nonce,
                            gas_limit=estimate_gas,
                            to=tx["to"],
                            value=Web3.toWei(0.01, 'ether'),
                            data=tx["data"],
                            maxPriorityFeePerGas=100000000,
                            maxFeePerGas=gas_price,
                            from_=account.address,
                            meta=tx["eip712Meta"])

    singed_message = signer.sign_typed_data(tx_712.to_eip712_struct())
    msg = tx_712.encode(singed_message)
    tx_hash = zksync_web3.zksync.send_raw_transaction(msg)
    tx_receipt = zksync_web3.zksync.wait_for_transaction_receipt(tx_hash, timeout=240, poll_latency=0.5)
    print(f"tx status: {tx_receipt['status']}")


if __name__ == "__main__":
    transfer_to_self()

```

#### Transfer funds (ERC20 tokens)

Example of transferring ERC20 tokens

```
from zksync2.module.request_types import create_function_call_transaction
from zksync2.manage_contracts.erc20_contract import ERC20FunctionEncoder
from zksync2.module.module_builder import ZkSyncBuilder
from zksync2.core.types import ZkBlockParams
from eth_account import Account
from eth_account.signers.local import LocalAccount
from zksync2.signer.eth_signer import PrivateKeyEthSigner
from zksync2.transaction.transaction712 import Transaction712


def transfer_erc20_token():
    ZKSYNC_NETWORK_URL: str = 'https://'
    account: LocalAccount = Account.from_key('YOUR_PRIVATE_KEY')
    zksync_web3 = ZkSyncBuilder.build(ZKSYNC_NETWORK_URL)
    chain_id = zksync_web3.zksync.chain_id
    signer = PrivateKeyEthSigner(account, chain_id)

    nonce = zksync_web3.zksync.get_transaction_count(account.address, ZkBlockParams.COMMITTED.value)
    tokens = zksync_web3.zksync.zks_get_confirmed_tokens(0, 100)
    not_eth_tokens = [x for x in tokens if not x.is_eth()]
    token_address = not_eth_tokens[0].l2_address

    erc20_encoder = ERC20FunctionEncoder(zksync_web3)
    transfer_params = [account.address, 0]
    call_data = erc20_encoder.encode_method("transfer", args=transfer_params)

    tx = create_function_call_transaction(from_=account.address,
                                          to=token_address,
                                          ergs_price=0,
                                          ergs_limit=0,
                                          data=call_data)
    estimate_gas = zksync_web3.zksync.eth_estimate_gas(tx)
    gas_price = zksync_web3.zksync.gas_price

    print(f"Fee for transaction is: {estimate_gas * gas_price}")

    tx_712 = Transaction712(chain_id=chain_id,
                            nonce=nonce,
                            gas_limit=estimate_gas,
                            to=tx["to"],
                            value=tx["value"],
                            data=tx["data"],
                            maxPriorityFeePerGas=100000000,
                            maxFeePerGas=gas_price,
                            from_=account.address,
                            meta=tx["eip712Meta"])
    singed_message = signer.sign_typed_data(tx_712.to_eip712_struct())
    msg = tx_712.encode(singed_message)
    tx_hash = zksync_web3.zksync.send_raw_transaction(msg)
    tx_receipt = zksync_web3.zksync.wait_for_transaction_receipt(tx_hash, timeout=240, poll_latency=0.5)
    print(f"tx status: {tx_receipt['status']}")


if __name__ == "__main__":
    transfer_erc20_token()

```

#### Withdraw funds (Native coins)

```
from decimal import Decimal
from eth_typing import HexStr
from zksync2.module.request_types import create_function_call_transaction
from zksync2.manage_contracts.l2_bridge import L2BridgeEncoder
from zksync2.module.module_builder import ZkSyncBuilder
from zksync2.core.types import Token, ZkBlockParams, BridgeAddresses
from eth_account import Account
from eth_account.signers.local import LocalAccount

from zksync2.signer.eth_signer import PrivateKeyEthSigner
from zksync2.transaction.transaction712 import Transaction712


def withdraw():
    ZKSYNC_NETWORK_URL: str = 'https://'
    account: LocalAccount = Account.from_key('YOUR_PRIVATE_KEY')
    zksync_web3 = ZkSyncBuilder.build(ZKSYNC_NETWORK_URL)
    chain_id = zksync_web3.zksync.chain_id
    signer = PrivateKeyEthSigner(account, chain_id)
    ETH_TOKEN = Token.create_eth()

    nonce = zksync_web3.zksync.get_transaction_count(account.address, ZkBlockParams.COMMITTED.value)
    bridges: BridgeAddresses = zksync_web3.zksync.zks_get_bridge_contracts()

    l2_func_encoder = L2BridgeEncoder(zksync_web3)
    call_data = l2_func_encoder.encode_function(fn_name="withdraw", args=[
        account.address,
        ETH_TOKEN.l2_address,
        ETH_TOKEN.to_int(Decimal("0.001"))
    ])

    tx = create_function_call_transaction(from_=account.address,
                                          to=bridges.l2_eth_default_bridge,
                                          ergs_limit=0,
                                          ergs_price=0,
                                          data=HexStr(call_data))
    estimate_gas = zksync_web3.zksync.eth_estimate_gas(tx)
    gas_price = zksync_web3.zksync.gas_price

    print(f"Fee for transaction is: {estimate_gas * gas_price}")

    tx_712 = Transaction712(chain_id=chain_id,
                            nonce=nonce,
                            gas_limit=estimate_gas,
                            to=tx["to"],
                            value=tx["value"],
                            data=tx["data"],
                            maxPriorityFeePerGas=100000000,
                            maxFeePerGas=gas_price,
                            from_=account.address,
                            meta=tx["eip712Meta"])

    singed_message = signer.sign_typed_data(tx_712.to_eip712_struct())
    msg = tx_712.encode(singed_message)
    tx_hash = zksync_web3.zksync.send_raw_transaction(msg)
    tx_receipt = zksync_web3.zksync.wait_for_transaction_receipt(tx_hash, timeout=240, poll_latency=0.5)
    print(f"tx status: {tx_receipt['status']}")


if __name__ == "__main__":
    withdraw()

```


#### Deploy contract with method create

ZkSync tools allows to build the contract into binary format. Then it can be deployed to the network<br>
Here is the code of simple contract:

```
pragma solidity ^0.8.0;

contract Counter {
    uint256 value;

    function increment(uint256 x) public {
        value += x;
    }

    function get() public view returns (uint256) {
        return value;
    }
}
```


> INFO: It must be compiled by ZkSync compiler only !

After compilation there must be 2 files with:

- contract binary representation
- contract abi in json format


Contract ABI needs for calling its methods in standard web3 way<br>

> INFO: in some cases you would need to get contract address before deploying it<br>
> INFO: This case is also introduced in this example

```
import json
from pathlib import Path
from eth_typing import HexStr
from web3 import Web3
from web3.types import TxParams
from zksync2.module.request_types import create_contract_transaction
from zksync2.manage_contracts.contract_deployer import ContractDeployer
from zksync2.manage_contracts.nonce_holder import NonceHolder
from zksync2.module.module_builder import ZkSyncBuilder
from zksync2.core.types import ZkBlockParams, EthBlockParams
from eth_account import Account
from eth_account.signers.local import LocalAccount
from zksync2.signer.eth_signer import PrivateKeyEthSigner
from zksync2.transaction.transaction712 import Transaction712


def read_binary(p: Path) -> bytes:
    with p.open(mode='rb') as contact_file:
        data = contact_file.read()
        return data


def get_abi(p: Path):
    with p.open(mode='r') as json_f:
        return json.load(json_f)


class CounterContractEncoder:
    def __init__(self, web3: Web3, bin_path: Path, abi_path: Path):
        self.web3 = web3
        self.counter_contract = self.web3.eth.contract(abi=get_abi(abi_path),
                                                       bytecode=read_binary(bin_path))

    def encode_method(self, fn_name, args: list) -> HexStr:
        return self.counter_contract.encodeABI(fn_name, args)


def deploy_contract_create():
    ZKSYNC_NETWORK_URL: str = 'https://'
    account: LocalAccount = Account.from_key('YOUR_PRIVATE_KEY')
    zksync_web3 = ZkSyncBuilder.build(ZKSYNC_NETWORK_URL)
    chain_id = zksync_web3.zksync.chain_id
    signer = PrivateKeyEthSigner(account, chain_id)

    counter_contract_bin = read_binary("PATH_TO_BINARY_COMPILED_CONTRACT")

    nonce = zksync_web3.zksync.get_transaction_count(account.address, EthBlockParams.PENDING.value)
    nonce_holder = NonceHolder(zksync_web3, account)
    deployment_nonce = nonce_holder.get_deployment_nonce(account.address)
    deployer = ContractDeployer(zksync_web3)
    precomputed_address = deployer.compute_l2_create_address(account.address, deployment_nonce)

    print(f"precomputed address: {precomputed_address}")

    tx = create_contract_transaction(web3=zksync_web3,
                                     from_=account.address,
                                     ergs_limit=0,
                                     ergs_price=0,
                                     bytecode=counter_contract_bin)

    estimate_gas = zksync_web3.zksync.eth_estimate_gas(tx)
    gas_price = zksync_web3.zksync.gas_price
    print(f"Fee for transaction is: {estimate_gas * gas_price}")

    tx_712 = Transaction712(chain_id=chain_id,
                            nonce=nonce,
                            gas_limit=estimate_gas,
                            to=tx["to"],
                            value=tx["value"],
                            data=tx["data"],
                            maxPriorityFeePerGas=100000000,
                            maxFeePerGas=gas_price,
                            from_=account.address,
                            meta=tx["eip712Meta"])

    singed_message = signer.sign_typed_data(tx_712.to_eip712_struct())
    msg = tx_712.encode(singed_message)
    tx_hash = zksync_web3.zksync.send_raw_transaction(msg)
    tx_receipt = zksync_web3.zksync.wait_for_transaction_receipt(tx_hash, timeout=240, poll_latency=0.5)
    print(f"tx status: {tx_receipt['status']}")

    contract_address = tx_receipt["contractAddress"]
    print(f"contract address: {contract_address}")

    counter_contract_encoder = CounterContractEncoder(zksync_web3, "PATH_TO_BINARY_COMPILED_CONTRACT",
                                                      "PATH_TO_CONTRACT_ABI")

    call_data = counter_contract_encoder.encode_method(fn_name="get", args=[])
    eth_tx: TxParams = {
        "from": account.address,
        "to": contract_address,
        "data": call_data
    }
    # Value is type dependent so might need to be converted to corresponded type under Python
    eth_ret = zksync_web3.zksync.call(eth_tx, ZkBlockParams.COMMITTED.value)
    converted_result = int.from_bytes(eth_ret, "big", signed=True)
    print(f"Call method for deployed contract, address: {contract_address}, value: {converted_result}")


if __name__ == "__main__":
    deploy_contract_create()

```

#### Deploy contract with method create2


```
import os
import json
from pathlib import Path
from eth_typing import HexStr
from web3 import Web3
from web3.types import TxParams
from zksync2.module.request_types import create2_contract_transaction
from zksync2.manage_contracts.contract_deployer import ContractDeployer
from zksync2.module.module_builder import ZkSyncBuilder
from zksync2.core.types import ZkBlockParams, EthBlockParams
from eth_account import Account
from eth_account.signers.local import LocalAccount
from zksync2.signer.eth_signer import PrivateKeyEthSigner
from zksync2.transaction.transaction712 import Transaction712


def generate_random_salt() -> bytes:
    return os.urandom(32)


def read_binary(p: Path) -> bytes:
    with p.open(mode='rb') as contact_file:
        data = contact_file.read()
        return data


def get_abi(p: Path):
    with p.open(mode='r') as json_f:
        return json.load(json_f)


class CounterContractEncoder:
    def __init__(self, web3: Web3, bin_path: Path, abi_path: Path):
        self.web3 = web3
        self.counter_contract = self.web3.eth.contract(abi=get_abi(abi_path),
                                                       bytecode=read_binary(bin_path))

    def encode_method(self, fn_name, args: list) -> HexStr:
        return self.counter_contract.encodeABI(fn_name, args)


def deploy_contract_create2():
    ZKSYNC_NETWORK_URL: str = 'https://'
    account: LocalAccount = Account.from_key('YOUR_PRIVATE_KEY')
    zksync_web3 = ZkSyncBuilder.build(ZKSYNC_NETWORK_URL)
    chain_id = zksync_web3.zksync.chain_id
    signer = PrivateKeyEthSigner(account, chain_id)

    counter_contract_bin = read_binary("PATH_TO_BINARY_COMPILED_CONTRACT")

    nonce = zksync_web3.zksync.get_transaction_count(account.address, EthBlockParams.PENDING.value)
    deployer = ContractDeployer(zksync_web3)
    random_salt = generate_random_salt()
    precomputed_address = deployer.compute_l2_create2_address(sender=account.address,
                                                              bytecode=counter_contract_bin,
                                                              constructor=b'',
                                                              salt=random_salt)
    print(f"precomputed address: {precomputed_address}")

    tx = create2_contract_transaction(web3=zksync_web3,
                                      from_=account.address,
                                      ergs_price=0,
                                      ergs_limit=0,
                                      bytecode=counter_contract_bin,
                                      salt=random_salt)
    estimate_gas = zksync_web3.zksync.eth_estimate_gas(tx)
    gas_price = zksync_web3.zksync.gas_price
    print(f"Fee for transaction is: {estimate_gas * gas_price}")

    tx_712 = Transaction712(chain_id=chain_id,
                            nonce=nonce,
                            gas_limit=estimate_gas,
                            to=tx["to"],
                            value=tx["value"],
                            data=tx["data"],
                            maxPriorityFeePerGas=100000000,
                            maxFeePerGas=gas_price,
                            from_=account.address,
                            meta=tx["eip712Meta"])
    singed_message = signer.sign_typed_data(tx_712.to_eip712_struct())
    msg = tx_712.encode(singed_message)
    tx_hash = zksync_web3.zksync.send_raw_transaction(msg)
    tx_receipt = zksync_web3.zksync.wait_for_transaction_receipt(tx_hash, timeout=240, poll_latency=0.5)
    print(f"tx status: {tx_receipt['status']}")

    contract_address = tx_receipt["contractAddress"]
    print(f"contract address: {contract_address}")

    counter_contract_encoder = CounterContractEncoder(zksync_web3, "CONTRACT_BIN_PATH", "CONTRACT_ABI_PATH")
    call_data = counter_contract_encoder.encode_method(fn_name="get", args=[])
    eth_tx: TxParams = {
        "from": account.address,
        "to": contract_address,
        "data": call_data
    }
    eth_ret = zksync_web3.zksync.call(eth_tx, ZkBlockParams.COMMITTED.value)
    result = int.from_bytes(eth_ret, "big", signed=True)
    print(f"Call method for deployed contract, address: {contract_address}, value: {result}")


if __name__ == "__main__":
    deploy_contract_create2()

```