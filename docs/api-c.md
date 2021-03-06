# API Reference C


## Overview

The C implementation of the Incubed client is prepared and optimized to run on small embedded devices. Because each device is different, we prepare different modules that should be combined. This allows us to only generate the code needed and reduce requirements for flash and memory.

### Why C?

We have been asked a lot, why we implemented Incubed in C and not in Rust. When we started Incubed we began with a feasibility test and wrote the client in TypeScript. Once we confirmed it was working, we wanted to provide a minimal verifaction client for embedded devices. And yes, we actually wanted to do it in Rust, since Rust offers a lot of safety-features (like the memory-management at compiletime, thread-safety, ...), but after considering a lot of different aspects we made a pragmatic desicion to use C.

These are the reasons why:

#### Support for embedded devices.

As of today almost all toolchain used in the embedded world are build for C. Even though Rust may be able to still use some, there are a lot of issues. Quote from [rust-embedded.org](https://docs.rust-embedded.org/book/interoperability/#interoperability-with-rtoss):

*Integrating Rust with an RTOS such as FreeRTOS or ChibiOS is still a work in progress; especially calling RTOS functions from Rust can be tricky.*

This may change in the future, but C is so dominant, that chances of Rust taking over the embedded development completly is low.

#### Portability

C is the most portable programming language. Rust actually has a pretty admirable selection of supported targets for a new language (thanks mostly to LLVM), but it pales in comparison to C, which runs on almost everything. A new CPU architecture or operating system can barely be considered to exist until it has a C compiler. And once it does, it unlocks access to a vast repository of software written in C. Many other programming languages, such as Ruby and Python, are implemented in C and you get those for free too.

Most programing language have very good support for calling c-function in a shared library (like ctypes in python or cgo in golang) or even support integration of C code directly like [android studio](https://developer.android.com/studio/projects/add-native-code) does.

#### Integration in existing projects

Since especially embedded systems are usually written in C/C++, offering a pure C-Implementation makes it easy for these projects to use Incubed, since they do not have to change their toolchain.

Even though we may not be able to use a lot of great features Rust offers by going with C, it allows to reach the goal to easily integrate with a lot of projects. For the future we might port the incubed to Rust if we see a demand or chance for the same support as C has today.

### Modules

Incubed consists of different modules. While the core module is always required, additional functions will be prepared by different modules.

```eval_rst
.. graphviz::

    digraph "GG" {
        graph [ rankdir = "RL" ]
        node [
          fontsize = "12"
          fontname="Helvetica"
          shape="ellipse"
        ];
    
        subgraph cluster_transport {
            label="Transports"  color=lightblue  style=filled
            transport_http;
            transport_curl;
    
        }
    
    
        evm;
        tommath;
    
        subgraph cluster_verifier {
            label="Verifiers"  color=lightblue  style=filled
            eth_basic;
            eth_full;
            eth_nano;
            btc;
        }
        subgraph cluster_bindings {
            label="Bindings"  color=lightblue  style=filled
            wasm;
            java;
            python;
    
        }
        subgraph cluster_api {
            label="APIs"  color=lightblue  style=filled
            eth_api;
            usn_api;
    
        }
    
        core;
        segger_rtt;
        crypto;
        core -> segger_rtt;
        core -> crypto // core -> crypto
        eth_api -> eth_nano // eth_api -> eth_nano
        eth_nano -> core // eth_nano -> core
        btc -> core // eth_nano -> core
        eth_basic -> eth_nano // eth_basic -> eth_nano
        eth_full -> evm // eth_full -> evm
        evm -> eth_basic // evm -> eth_basic
        evm -> tommath // evm -> tommath
        transport_http -> core // transport_http -> core
        transport_curl -> core // transport_http -> core
        usn_api -> core // usn_api -> core
    
        java -> core // usn_api -> core
        python -> core // usn_api -> core
        wasm -> core // usn_api -> core
    }
    
```
#### Verifier

Incubed is a minimal verification client, which means that each response needs to be verifiable. Depending on the expected requests and responses, you need to carefully choose which verifier you may need to register. For Ethereum, we have developed three modules:

1. [eth_nano](#module-eth-nano): a minimal module only able to verify transaction receipts (`eth_getTransactionReceipt`).
2. [eth_basic](#module-eth-basic): module able to verify almost all other standard RPC functions (except `eth_call`).
3. [eth_full](#module-eth-full): module able to verify standard RPC functions. It also implements a full EVM to handle `eth_call`.

1. [btc](#module-btc): module able to verify bitcoin or bitcoin based chains.

Depending on the module, you need to register the verifier before using it. This is done by calling the `in3_register...` function like [in3_register_eth_full()](#in3-register-eth-full).

#### Transport

To verify responses, you need to be able to send requests. The way to handle them depends heavily on your hardware capabilities. For example, if your device only supports Bluetooth, you may use this connection to deliver the request to a device with an existing internet connection and get the response in the same way, but if your device is able to use a direct internet connection, you may use a curl-library to execute them. This is why the core client only defines function pointer [in3_transport_send](#in3-transport-send), which must handle the requests.

At the moment we offer these modules; other implementations are supported by different hardware modules.

1. [transport_curl](#module-transport-curl): module with a dependency on curl, which executes these requests and supports HTTPS. This module runs a standard OS with curl installed.
2. [transport_http](#module-transport-http): module with no dependency, but a very basic http-implementation (no https-support)

#### API

While Incubed operates on JSON-RPC level, as a developer, you might want to use a better-structured API to prepare these requests for you. These APIs are optional but make life easier:

1. [**eth**](#module-eth-api): This module offers all standard RPC functions as described in the [Ethereum JSON-RPC Specification](https://github.com/ethereum/wiki/wiki/JSON-RPC). In addition, it allows you to sign and encode/decode calls and transactions.
2. [**usn**](#module-usn-api): This module offers basic USN functions like renting, event handling, and message verification.




## Building

While we provide binaries, you can also build from source:

### requirements

- cmake
- curl : curl is used as transport for command-line tools.
- optional: libsycrypt, which would be used for unlocking keystore files using `scrypt` as kdf method. if it does not exist you can still build, but not decrypt such keys. 


 for osx `brew install libscrypt` and for debian `sudo apt-get install libscrypt-dev`

Incubed uses cmake for configuring:

```c
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release .. && make
make install
```
### CMake options

When configuring cmake, you can set a lot of different incubed specific like `cmake -DEVM_GAS=false ..`.

#### ASMJS

compiles the code as asm.js.

Default-Value: `-DASMJS=OFF`

#### BUILD_DOC

generates the documenation with doxygen.

Default-Value: `-DBUILD_DOC=OFF`

#### CMD

build the comandline utils

Default-Value: `-DCMD=ON`

#### ERR_MSG

if set human readable error messages will be inculded in th executable, otherwise only the error code is used. (saves about 19kB)

Default-Value: `-DERR_MSG=ON`

#### ETH_BASIC

build basic eth verification.(all rpc-calls except eth_call)

Default-Value: `-DETH_BASIC=ON`

#### ETH_FULL

build full eth verification.(including eth_call)

Default-Value: `-DETH_FULL=ON`

#### ETH_NANO

build minimal eth verification.(eth_getTransactionReceipt)

Default-Value: `-DETH_NANO=ON`

#### EVM_GAS

if true the gas costs are verified when validating a eth_call. This is a optimization since most calls are only interessted in the result. EVM_GAS would be required if the contract uses gas-dependend op-codes.

Default-Value: `-DEVM_GAS=ON`

#### FAST_MATH

Math optimizations used in the EVM. This will also increase the filesize.

Default-Value: `-DFAST_MATH=OFF`

#### FILTER_NODES

if true the nodelist is filtered against config node properties

Default-Value: `-DFILTER_NODES=OFF`

#### IN3API

build the USN-API which offer better interfaces and additional functions on top of the pure verification

Default-Value: `-DIN3API=ON`

#### IN3_LIB

if true a shared anmd static library with all in3-modules will be build.

Default-Value: `-DIN3_LIB=ON`

#### IN3_SERVER

support for proxy server as part of the cmd-tool, which allows to start the cmd-tool with the -p option and listens to the given port for rpc-requests

Default-Value: `-DIN3_SERVER=OFF`

#### IN3_STAGING

if true, the client will use the staging-network instead of the live ones

Default-Value: `-DIN3_STAGING=OFF`

#### JAVA

build the java-binding (shared-lib and jar-file)

Default-Value: `-DJAVA=OFF`

#### POA

support POA verification including validatorlist updates

Default-Value: `-DPOA=OFF`

#### SEGGER_RTT

Use the segger real time transfer terminal as the logging mechanism

Default-Value: `-DSEGGER_RTT=OFF`

#### TAG_VERSION

the tagged version, which should be used

Default-Value: `-DTAG_VERSION=OFF`

#### TEST

builds the tests and also adds special memory-management, which detects memory leaks, but will cause slower performance

Default-Value: `-DTEST=OFF`

#### TRANSPORTS

builds transports, which may require extra libraries.

Default-Value: `-DTRANSPORTS=ON`

#### USE_CURL

if true the curl transport will be build (with a dependency to libcurl)

Default-Value: `-DUSE_CURL=ON`

#### USE_PRECOMPUTED_EC

if true the secp256k1 curve uses precompiled tables to boost performance. turning this off makes ecrecover slower, but saves about 37kb.

Default-Value: `-DUSE_PRECOMPUTED_EC=ON`

#### USE_SCRYPT

if scrypt is installed, it will link dynamicly to the shared scrypt lib.

Default-Value: `-DUSE_SCRYPT=OFF`

#### WASM

Includes the WASM-Build. In order to build it you need emscripten as toolchain. Usually you also want to turn off other builds in this case.

Default-Value: `-DWASM=OFF`

#### WASM_EMBED

embedds the wasm as base64-encoded into the js-file

Default-Value: `-DWASM_EMBED=ON`

#### WASM_EMMALLOC

use ther smaller EMSCRIPTEN Malloc, which reduces the size about 10k, but may be a bit slower

Default-Value: `-DWASM_EMMALLOC=ON`

#### WASM_SYNC

intiaializes the WASM synchronisly, which allows to require and use it the same function, but this will not be supported by chrome (4k limit)

Default-Value: `-DWASM_SYNC=OFF` 




## Examples

### call_a_function

source : [in3-c/examples/c/call_a_function.c](https://github.com/slockit/in3-c/blob/master/examples/c/call_a_function.c)

This example shows how to call functions on a smart contract eiither directly or using the api to encode the arguments

```c

#include <in3/client.h>   // the core client
#include <in3/eth_api.h>  // wrapper for easier use
#include <in3/eth_full.h> // the full ethereum verifier containing the EVM
#include <in3/in3_curl.h> // transport implementation
#include <in3/log.h>
#include <inttypes.h>
#include <stdio.h>

static in3_ret_t call_func_rpc(in3_t* c);
static in3_ret_t call_func_api(in3_t* c, address_t contract);

int main() {
  in3_ret_t ret = IN3_OK;

  // register a chain-verifier for full Ethereum-Support in order to verify eth_call
  // this needs to be called only once.
  in3_register_eth_full();

  // use curl as the default for sending out requests
  // this needs to be called only once.
  in3_register_curl();

  // Remove log prefix for readability
  in3_log_set_prefix("");

  // create new incubed client
  in3_t* c = in3_for_chain(ETH_CHAIN_ID_MAINNET);

  // define a address (20byte)
  address_t contract;

  // copy the hexcoded string into this address
  hex_to_bytes("0x2736D225f85740f42D17987100dc8d58e9e16252", -1, contract, 20);

  // call function using RPC
  ret = call_func_rpc(c);
  if (ret != IN3_OK) goto END;

  // call function using API
  ret = call_func_api(c, contract);
  if (ret != IN3_OK) goto END;

END:
  // clean up
  in3_free(c);
  return 0;
}

in3_ret_t call_func_rpc(in3_t* c) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      c,                                                                                                //  the configured client
      "eth_call",                                                                                       // the rpc-method you want to call.
      "[{\"to\":\"0x2736d225f85740f42d17987100dc8d58e9e16252\", \"data\":\"0x15625c5e\"}, \"latest\"]", // the signed raw txn, same as the one used in the API example
      &result,                                                                                          // the reference to a pointer which will hold the result
      &error);                                                                                          // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Result: \n%s\n", result);
    free(result);
    return 0;
  } else {
    printf("Error sending tx: \n%s\n", error);
    free(error);
    return IN3_EUNKNOWN;
  }
}

in3_ret_t call_func_api(in3_t* c, address_t contract) {
  // ask for the number of servers registered
  json_ctx_t* response = eth_call_fn(c, contract, BLKNUM_LATEST(), "totalServers():uint256");
  if (!response) {
    printf("Could not get the response: %s", eth_last_error());
    return IN3_EUNKNOWN;
  }

  // convert the response to a uint32_t,
  uint32_t number_of_servers = d_int(response->result);

  // clean up resources
  json_free(response);

  // output
  printf("Found %u servers registered : \n", number_of_servers);

  // read all structs ...
  for (uint32_t i = 0; i < number_of_servers; i++) {
    response = eth_call_fn(c, contract, BLKNUM_LATEST(), "servers(uint256):(string,address,uint,uint,uint,address)", to_uint256(i));
    if (!response) {
      printf("Could not get the response: %s", eth_last_error());
      return IN3_EUNKNOWN;
    }

    char*    url     = d_get_string_at(response->result, 0); // get the first item of the result (the url)
    bytes_t* owner   = d_get_bytes_at(response->result, 1);  // get the second item of the result (the owner)
    uint64_t deposit = d_get_long_at(response->result, 2);   // get the third item of the result (the deposit)

    printf("Server %i : %s owner = %02x%02x...", i, url, owner->data[0], owner->data[1]);
    printf(", deposit = %" PRIu64 "\n", deposit);

    // free memory
    json_free(response);
  }
  return 0;
}
```
### get_balance

source : [in3-c/examples/c/get_balance.c](https://github.com/slockit/in3-c/blob/master/examples/c/get_balance.c)

get the Balance with the API and also as direct RPC-call

```c

#include <in3/client.h>    // the core client
#include <in3/eth_api.h>   // wrapper for easier use
#include <in3/eth_basic.h> // use the basic module
#include <in3/in3_curl.h>  // transport implementation

#include <stdio.h>

static void get_balance_rpc(in3_t* in3);
static void get_balance_api(in3_t* in3);

int main() {

  // register a chain-verifier for basic Ethereum-Support, which is enough to verify accounts
  // this needs to be called only once
  in3_register_eth_basic();

  // use curl as the default for sending out requests
  // this needs to be called only once.
  in3_register_curl();

  // create new incubed client
  in3_t* in3 = in3_for_chain(ETH_CHAIN_ID_MAINNET);

  // get balance using raw RPC call
  get_balance_rpc(in3);

  // get balance using API
  get_balance_api(in3);

  // cleanup client after usage
  in3_free(in3);
}

void get_balance_rpc(in3_t* in3) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      in3,                                                            //  the configured client
      "eth_getBalance",                                               // the rpc-method you want to call.
      "[\"0xc94770007dda54cF92009BFF0dE90c06F603a09f\", \"latest\"]", // the arguments as json-string
      &result,                                                        // the reference to a pointer whill hold the result
      &error);                                                        // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Balance: \n%s\n", result);
    free(result);
  } else {
    printf("Error getting balance: \n%s\n", error);
    free(error);
  }
}

void get_balance_api(in3_t* in3) {
  // the address of account whose balance we want to get
  address_t account;
  hex_to_bytes("0xc94770007dda54cF92009BFF0dE90c06F603a09f", -1, account, 20);

  // get balance of account
  long double balance = as_double(eth_getBalance(in3, account, BLKNUM_EARLIEST()));

  // if the result is null there was an error an we can get the latest error message from eth_lat_error()
  balance ? printf("Balance: %Lf\n", balance) : printf("error getting the balance : %s\n", eth_last_error());
}
```
### get_block

source : [in3-c/examples/c/get_block.c](https://github.com/slockit/in3-c/blob/master/examples/c/get_block.c)

using the basic-module to get and verify a Block with the API and also as direct RPC-call

```c

#include <in3/client.h>    // the core client
#include <in3/eth_api.h>   // wrapper for easier use
#include <in3/eth_basic.h> // use the basic module
#include <in3/in3_curl.h>  // transport implementation

#include <inttypes.h>
#include <stdio.h>

static void get_block_rpc(in3_t* in3);
static void get_block_api(in3_t* in3);

int main() {

  // register a chain-verifier for basic Ethereum-Support, which is enough to verify blocks
  // this needs to be called only once
  in3_register_eth_basic();

  // use curl as the default for sending out requests
  // this needs to be called only once.
  in3_register_curl();

  // create new incubed client
  in3_t* in3 = in3_for_chain(ETH_CHAIN_ID_MAINNET);

  // get block using raw RPC call
  get_block_rpc(in3);

  // get block using API
  get_block_api(in3);

  // cleanup client after usage
  in3_free(in3);
}

void get_block_rpc(in3_t* in3) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      in3,                    //  the configured client
      "eth_getBlockByNumber", // the rpc-method you want to call.
      "[\"latest\",true]",    // the arguments as json-string
      &result,                // the reference to a pointer whill hold the result
      &error);                // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Latest block : \n%s\n", result);
    free(result);
  } else {
    printf("Error verifing the Latest block : \n%s\n", error);
    free(error);
  }
}

void get_block_api(in3_t* in3) {
  // get the block without the transaction details
  eth_block_t* block = eth_getBlockByNumber(in3, BLKNUM(8432424), false);

  // if the result is null there was an error an we can get the latest error message from eth_lat_error()
  if (!block)
    printf("error getting the block : %s\n", eth_last_error());
  else {
    printf("Number of transactions in Block #%llu: %d\n", block->number, block->tx_count);
    free(block);
  }
}
```
### get_logs

source : [in3-c/examples/c/get_logs.c](https://github.com/slockit/in3-c/blob/master/examples/c/get_logs.c)

fetching events and verify them with eth_getLogs

```c

#include <in3/client.h>    // the core client
#include <in3/eth_api.h>   // wrapper for easier use
#include <in3/eth_basic.h> // use the basic module
#include <in3/in3_curl.h>  // transport implementation

#include <inttypes.h>
#include <stdio.h>

static void get_logs_rpc(in3_t* in3);
static void get_logs_api(in3_t* in3);

int main() {

  // register a chain-verifier for basic Ethereum-Support, which is enough to verify logs
  // this needs to be called only once
  in3_register_eth_basic();

  // use curl as the default for sending out requests
  // this needs to be called only once.
  in3_register_curl();

  // create new incubed client
  in3_t* in3    = in3_for_chain(ETH_CHAIN_ID_MAINNET);
  in3->chain_id = ETH_CHAIN_ID_KOVAN;

  // get logs using raw RPC call
  get_logs_rpc(in3);

  // get logs using API
  get_logs_api(in3);

  // cleanup client after usage
  in3_free(in3);
}

void get_logs_rpc(in3_t* in3) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      in3,           //  the configured client
      "eth_getLogs", // the rpc-method you want to call.
      "[{}]",        // the arguments as json-string
      &result,       // the reference to a pointer whill hold the result
      &error);       // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Logs : \n%s\n", result);
    free(result);
  } else {
    printf("Error getting logs : \n%s\n", error);
    free(error);
  }
}

void get_logs_api(in3_t* in3) {
  // Create filter options
  char b[30];
  sprintf(b, "{\"fromBlock\":\"0x%" PRIx64 "\"}", eth_blockNumber(in3) - 2);
  json_ctx_t* jopt = parse_json(b);

  // Create new filter with options
  size_t fid = eth_newFilter(in3, jopt);

  // Get logs
  eth_log_t* logs = NULL;
  in3_ret_t  ret  = eth_getFilterLogs(in3, fid, &logs);
  if (ret != IN3_OK) {
    printf("eth_getFilterLogs() failed [%d]\n", ret);
    return;
  }

  // print result
  while (logs) {
    eth_log_t* l = logs;
    printf("--------------------------------------------------------------------------------\n");
    printf("\tremoved: %s\n", l->removed ? "true" : "false");
    printf("\tlogId: %lu\n", l->log_index);
    printf("\tTxId: %lu\n", l->transaction_index);
    printf("\thash: ");
    ba_print(l->block_hash, 32);
    printf("\n\tnum: %" PRIu64 "\n", l->block_number);
    printf("\taddress: ");
    ba_print(l->address, 20);
    printf("\n\tdata: ");
    b_print(&l->data);
    printf("\ttopics[%lu]: ", l->topic_count);
    for (size_t i = 0; i < l->topic_count; i++) {
      printf("\n\t");
      ba_print(l->topics[i], 32);
    }
    printf("\n");
    logs = logs->next;
    free(l->data.data);
    free(l->topics);
    free(l);
  }
  eth_uninstallFilter(in3, fid);
  json_free(jopt);
}
```
### get_transaction

source : [in3-c/examples/c/get_transaction.c](https://github.com/slockit/in3-c/blob/master/examples/c/get_transaction.c)

checking the transaction data

```c

#include <in3/client.h>    // the core client
#include <in3/eth_api.h>   // wrapper for easier use
#include <in3/eth_basic.h> // use the basic module
#include <in3/in3_curl.h>  // transport implementation

#include <stdio.h>

static void get_tx_rpc(in3_t* in3);
static void get_tx_api(in3_t* in3);

int main() {

  // register a chain-verifier for basic Ethereum-Support, which is enough to verify txs
  // this needs to be called only once
  in3_register_eth_basic();

  // use curl as the default for sending out requests
  // this needs to be called only once.
  in3_register_curl();

  // create new incubed client
  in3_t* in3 = in3_for_chain(ETH_CHAIN_ID_MAINNET);

  // get tx using raw RPC call
  get_tx_rpc(in3);

  // get tx using API
  get_tx_api(in3);

  // cleanup client after usage
  in3_free(in3);
}

void get_tx_rpc(in3_t* in3) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      in3,                                                                        //  the configured client
      "eth_getTransactionByHash",                                                 // the rpc-method you want to call.
      "[\"0xdd80249a0631cf0f1593c7a9c9f9b8545e6c88ab5252287c34bc5d12457eab0e\"]", // the arguments as json-string
      &result,                                                                    // the reference to a pointer which will hold the result
      &error);                                                                    // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Latest tx : \n%s\n", result);
    free(result);
  } else {
    printf("Error verifing the Latest tx : \n%s\n", error);
    free(error);
  }
}

void get_tx_api(in3_t* in3) {
  // the hash of transaction that we want to get
  bytes32_t tx_hash;
  hex_to_bytes("0xdd80249a0631cf0f1593c7a9c9f9b8545e6c88ab5252287c34bc5d12457eab0e", -1, tx_hash, 32);

  // get the tx by hash
  eth_tx_t* tx = eth_getTransactionByHash(in3, tx_hash);

  // if the result is null there was an error an we can get the latest error message from eth_last_error()
  if (!tx)
    printf("error getting the tx : %s\n", eth_last_error());
  else {
    printf("Transaction #%d of block #%llx", tx->transaction_index, tx->block_number);
    free(tx);
  }
}
```
### get_transaction_receipt

source : [in3-c/examples/c/get_transaction_receipt.c](https://github.com/slockit/in3-c/blob/master/examples/c/get_transaction_receipt.c)

validating the result or receipt of an transaction

```c

#include <in3/client.h>    // the core client
#include <in3/eth_api.h>   // wrapper for easier use
#include <in3/eth_basic.h> // use the basic module
#include <in3/in3_curl.h>  // transport implementation

#include <inttypes.h>
#include <stdio.h>

static void get_tx_receipt_rpc(in3_t* in3);
static void get_tx_receipt_api(in3_t* in3);

int main() {

  // register a chain-verifier for basic Ethereum-Support, which is enough to verify tx receipts
  // this needs to be called only once
  in3_register_eth_basic();

  // use curl as the default for sending out requests
  // this needs to be called only once.
  in3_register_curl();

  // create new incubed client
  in3_t* in3 = in3_for_chain(ETH_CHAIN_ID_MAINNET);

  // get tx receipt using raw RPC call
  get_tx_receipt_rpc(in3);

  // get tx receipt using API
  get_tx_receipt_api(in3);

  // cleanup client after usage
  in3_free(in3);
}

void get_tx_receipt_rpc(in3_t* in3) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      in3,                                                                        //  the configured client
      "eth_getTransactionReceipt",                                                // the rpc-method you want to call.
      "[\"0xdd80249a0631cf0f1593c7a9c9f9b8545e6c88ab5252287c34bc5d12457eab0e\"]", // the arguments as json-string
      &result,                                                                    // the reference to a pointer which will hold the result
      &error);                                                                    // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Transaction receipt: \n%s\n", result);
    free(result);
  } else {
    printf("Error verifing the tx receipt: \n%s\n", error);
    free(error);
  }
}

void get_tx_receipt_api(in3_t* in3) {
  // the hash of transaction whose receipt we want to get
  bytes32_t tx_hash;
  hex_to_bytes("0xdd80249a0631cf0f1593c7a9c9f9b8545e6c88ab5252287c34bc5d12457eab0e", -1, tx_hash, 32);

  // get the tx receipt by hash
  eth_tx_receipt_t* txr = eth_getTransactionReceipt(in3, tx_hash);

  // if the result is null there was an error an we can get the latest error message from eth_last_error()
  if (!txr)
    printf("error getting the tx : %s\n", eth_last_error());
  else {
    printf("Transaction #%d of block #%llx, gas used = %" PRIu64 ", status = %s\n", txr->transaction_index, txr->block_number, txr->gas_used, txr->status ? "success" : "failed");
    eth_tx_receipt_free(txr);
  }
}
```
### send_transaction

source : [in3-c/examples/c/send_transaction.c](https://github.com/slockit/in3-c/blob/master/examples/c/send_transaction.c)

sending a transaction including signing it with a private key

```c

#include <in3/client.h>    // the core client
#include <in3/eth_api.h>   // wrapper for easier use
#include <in3/eth_basic.h> // use the basic module
#include <in3/in3_curl.h>  // transport implementation
#include <in3/signer.h>    // default signer implementation

#include <stdio.h>

// fixme: This is only for the sake of demo. Do NOT store private keys as plaintext.
#define ETH_PRIVATE_KEY "0x8da4ef21b864d2cc526dbdb2a120bd2874c36c9d0a1fb7f8c63d7f7a8b41de8f"

static void send_tx_rpc(in3_t* in3);
static void send_tx_api(in3_t* in3);

int main() {

  // register a chain-verifier for basic Ethereum-Support, which is enough to verify txs
  // this needs to be called only once
  in3_register_eth_basic();

  // use curl as the default for sending out requests
  // this needs to be called only once.
  in3_register_curl();

  // create new incubed client
  in3_t* in3 = in3_for_chain(ETH_CHAIN_ID_MAINNET);

  // convert the hexstring to bytes
  bytes32_t pk;
  hex_to_bytes(ETH_PRIVATE_KEY, -1, pk, 32);

  // create a simple signer with this key
  eth_set_pk_signer(in3, pk);

  // send tx using raw RPC call
  send_tx_rpc(in3);

  // send tx using API
  send_tx_api(in3);

  // cleanup client after usage
  in3_free(in3);
}

void send_tx_rpc(in3_t* in3) {
  // prepare 2 pointers for the result.
  char *result, *error;

  // send raw rpc-request, which is then verified
  in3_ret_t res = in3_client_rpc(
      in3,                      //  the configured client
      "eth_sendRawTransaction", // the rpc-method you want to call.
      "[\"0xf892808609184e72a0008296c094d46e8dd67c5d32be8058bb8eb970870f0724456"
      "7849184e72aa9d46e8dd67c5d32be8d46e8dd67c5d32be8058bb8eb970870f072445675058bb8eb9"
      "70870f07244567526a06f0103fccdcae0d6b265f8c38ee42f4a722c1cb36230fe8da40315acc3051"
      "9a8a06252a68b26a5575f76a65ac08a7f684bc37b0c98d9e715d73ddce696b58f2c72\"]", // the signed raw txn, same as the one used in the API example
      &result,                                                                    // the reference to a pointer which will hold the result
      &error);                                                                    // the pointer which may hold a error message

  // check and print the result or error
  if (res == IN3_OK) {
    printf("Result: \n%s\n", result);
    free(result);
  } else {
    printf("Error sending tx: \n%s\n", error);
    free(error);
  }
}

void send_tx_api(in3_t* in3) {
  // prepare parameters
  address_t to, from;
  hex_to_bytes("0x63FaC9201494f0bd17B9892B9fae4d52fe3BD377", -1, from, 20);
  hex_to_bytes("0xd46e8dd67c5d32be8058bb8eb970870f07244567", -1, to, 20);

  bytes_t* data = hex_to_new_bytes("d46e8dd67c5d32be8d46e8dd67c5d32be8058bb8eb970870f072445675058bb8eb970870f072445675", 82);

  // send the tx
  bytes_t* tx_hash = eth_sendTransaction(in3, from, to, OPTIONAL_T_VALUE(uint64_t, 0x96c0), OPTIONAL_T_VALUE(uint64_t, 0x9184e72a000), OPTIONAL_T_VALUE(uint256_t, to_uint256(0x9184e72a)), OPTIONAL_T_VALUE(bytes_t, *data), OPTIONAL_T_UNDEFINED(uint64_t));

  // if the result is null there was an error and we can get the latest error message from eth_last_error()
  if (!tx_hash)
    printf("error sending the tx : %s\n", eth_last_error());
  else {
    printf("Transaction hash: ");
    b_print(tx_hash);
    b_free(tx_hash);
  }
  b_free(data);
}
```
### usn_device

source : [in3-c/examples/c/usn_device.c](https://github.com/slockit/in3-c/blob/master/examples/c/usn_device.c)

a example how to watch usn events and act upon it.

```c

#include <in3/client.h>   // the core client
#include <in3/eth_api.h>  // wrapper for easier use
#include <in3/eth_full.h> // the full ethereum verifier containing the EVM
#include <in3/in3_curl.h> // transport implementation
#include <in3/signer.h>   // signer-api
#include <in3/usn_api.h>  // api for renting
#include <inttypes.h>
#include <stdio.h>
#include <time.h>
#if defined(_WIN32) || defined(WIN32)
#include <windows.h>
#else
#include <unistd.h>
#endif

static int handle_booking(usn_event_t* ev) {
  printf("\n%s Booking timestamp=%" PRIu64 "\n", ev->type == BOOKING_START ? "START" : "STOP", ev->ts);
  return 0;
}

int main(int argc, char* argv[]) {

  // register a chain-verifier for full Ethereum-Support in order to verify eth_call
  // this needs to be called only once.
  in3_register_eth_full();

  // use curl as the default for sending out requests
  // this needs to be called only once.
  in3_register_curl();

  // create new incubed client
  in3_t* c = in3_for_chain(ETH_CHAIN_ID_MAINNET);

  // switch to goerli
  c->chain_id = 0x5;

  // setting up a usn-device-config
  usn_device_conf_t usn;
  usn.booking_handler    = handle_booking;                                          // this is the handler, which is called for each rent/return or start/stop
  usn.c                  = c;                                                       // the incubed client
  usn.chain_id           = c->chain_id;                                             // the chain_id
  usn.devices            = NULL;                                                    // this will contain the list of devices supported
  usn.len_devices        = 0;                                                       // and length of this list
  usn.now                = 0;                                                       // the current timestamp
  unsigned int wait_time = 5;                                                       // the time to wait between the internval
  hex_to_bytes("0x85Ec283a3Ed4b66dF4da23656d4BF8A507383bca", -1, usn.contract, 20); // address of the usn-contract, which we copy from hex

  // register a usn-device
  usn_register_device(&usn, "office@slockit");

  // now we run en endless loop which simply wait for events on the chain.
  printf("\n start watching...\n");
  while (true) {
    usn.now              = time(NULL);                               // update the timestamp, since this is running on embedded devices, this may be depend on the hardware.
    unsigned int timeout = usn_update_state(&usn, wait_time) * 1000; // this will now check for new events and trigger the handle_booking if so.

    // sleep
#if defined(_WIN32) || defined(WIN32)
    Sleep(timeout);
#else
    nanosleep((const struct timespec[]){{0, timeout * 1000000L}}, NULL);
#endif
  }

  // clean up
  in3_free(c);
  return 0;
}
```
### usn_rent

source : [in3-c/examples/c/usn_rent.c](https://github.com/slockit/in3-c/blob/master/examples/c/usn_rent.c)

how to send a rent transaction to a usn contract usinig the usn-api.

```c

#include <in3/client.h>   // the core client
#include <in3/eth_api.h>  // wrapper for easier use
#include <in3/eth_full.h> // the full ethereum verifier containing the EVM
#include <in3/in3_curl.h> // transport implementation
#include <in3/signer.h>   // signer-api
#include <in3/usn_api.h>  // api for renting
#include <inttypes.h>
#include <stdio.h>

void unlock_key(in3_t* c, char* json_data, char* passwd) {
  // parse the json
  json_ctx_t* key_data = parse_json(json_data);
  if (!key_data) {
    perror("key is not parseable!\n");
    exit(EXIT_FAILURE);
  }

  // decrypt the key
  uint8_t* pk = malloc(32);
  if (decrypt_key(key_data->result, passwd, pk) != IN3_OK) {
    perror("wrong password!\n");
    exit(EXIT_FAILURE);
  }

  // free json
  json_free(key_data);

  // create a signer with this key
  eth_set_pk_signer(c, pk);
}

int main(int argc, char* argv[]) {

  // register a chain-verifier for full Ethereum-Support in order to verify eth_call
  // this needs to be called only once.
  in3_register_eth_full();

  // use curl as the default for sending out requests
  // this needs to be called only once.
  in3_register_curl();

  // create new incubed client
  in3_t* c = in3_for_chain(ETH_CHAIN_ID_GOERLI);

  // address of the usn-contract, which we copy from hex
  address_t contract;
  hex_to_bytes("0x85Ec283a3Ed4b66dF4da23656d4BF8A507383bca", -1, contract, 20);

  // read the key from args - I know this is not safe, but this is just a example.
  if (argc < 3) {
    perror("you need to provide a json-key and password to rent it");
    exit(EXIT_FAILURE);
  }
  char* key_data = argv[1];
  char* passwd   = argv[2];
  unlock_key(c, key_data, passwd);

  // rent it for one hour.
  uint32_t renting_seconds = 3600;

  // allocate 32 bytes for the resulting tx hash
  bytes32_t tx_hash;

  // start charging
  if (usn_rent(c, contract, NULL, "office@slockit", renting_seconds, tx_hash))
    printf("Could not start charging\n");
  else {
    printf("Charging tx successfully sent... tx_hash=0x");
    for (int i = 0; i < 32; i++) printf("%02x", tx_hash[i]);
    printf("\n");

    if (argc == 4) // just to include it : if you want to stop earlier, you can call
      usn_return(c, contract, "office@slockit", tx_hash);
  }

  // clean up
  in3_free(c);
  return 0;
}
```
### Building

In order to run those examples, you only need a c-compiler (gcc or clang) and curl installed.

```c
./build.sh
```
will build all examples in this directory. You can build them individually by executing:

```c
gcc -o get_block_api get_block_api.c -lin3 -lcurl
```



## RPC

The core of incubed is to execute rpc-requests which will be send to the incubed nodes and verified. This means the available RPC-Requests are defined by the clients itself.

- For Ethereum : [https://github.com/ethereum/wiki/wiki/JSON-RPC](https://github.com/ethereum/wiki/wiki/JSON-RPC)
- For Bitcoin : [https://bitcoincore.org/en/doc/0.18.0/](https://bitcoincore.org/en/doc/0.18.0/)

The Incbed nodes already add a few special RPC-methods, which are specified in the [RPC-Specification](https://in3.readthedocs.io/en/develop/spec.html#incubed) Section of the Protocol.

In addition the incubed client itself offers special RPC-Methods, which are mostly handled directly inside the client:

### in3_config

changes the configuration of a client. The configuration is passed as the first param and may contain only the values to change.

Parameters:

1. `config`: config-object - a Object with config-params.

The config params support the following properties :

- **[autoUpdateList](https://github.com/slockit/in3/blob/master/src/types/types.ts#L255)** :`boolean` *(optional)* - if true the nodelist will be automaticly updated if the lastBlock is newer example: true
- **[chainId](https://github.com/slockit/in3/blob/master/src/types/types.ts#L240)** :`string` - servers to filter for the given chain. The chain-id based on EIP-155. example: 0x1
- **[finality](https://github.com/slockit/in3/blob/master/src/types/types.ts#L230)** :`number` *(optional)* - the number in percent needed in order reach finality (% of signature of the validators) example: 50
- **[includeCode](https://github.com/slockit/in3/blob/master/src/types/types.ts#L187)** :`boolean` *(optional)* - if true, the request should include the codes of all accounts. otherwise only the the codeHash is returned. In this case the client may ask by calling eth_getCode() afterwards example: true
- **[keepIn3](https://github.com/slockit/in3/blob/master/src/types/types.ts#L187)** :`boolean` *(optional)* - if true, requests sent to the input sream of the comandline util will be send theor responses in the same form as the server did. example: false
- **[key](https://github.com/slockit/in3/blob/master/src/types/types.ts#L169)** :`any` *(optional)* - the client key to sign requests example: 0x387a8233c96e1fc0ad5e284353276177af2186e7afa85296f106336e376669f7
- **[maxAttempts](https://github.com/slockit/in3/blob/master/src/types/types.ts#L182)** :`number` *(optional)* - max number of attempts in case a response is rejected example: 10
- **[maxBlockCache](https://github.com/slockit/in3/blob/master/src/types/types.ts#L197)** :`number` *(optional)* - number of number of blocks cached in memory example: 100
- **[maxCodeCache](https://github.com/slockit/in3/blob/master/src/types/types.ts#L192)** :`number` *(optional)* - number of max bytes used to cache the code in memory example: 100000
- **[minDeposit](https://github.com/slockit/in3/blob/master/src/types/types.ts#L215)** :`number` - min stake of the server. Only nodes owning at least this amount will be chosen.
- **[nodeLimit](https://github.com/slockit/in3/blob/master/src/types/types.ts#L155)** :`number` *(optional)* - the limit of nodes to store in the client. example: 150
- **[proof](https://github.com/slockit/in3/blob/master/src/types/types.ts#L206)** :,'none,`|`'standard'`|`'full'` *(optional)* - if true the nodes should send a proof of the response example: true
- **[replaceLatestBlock](https://github.com/slockit/in3/blob/master/src/types/types.ts#L220)** :`number` *(optional)* - if specified, the blocknumber *latest* will be replaced by blockNumber- specified value example: 6
- **[requestCount](https://github.com/slockit/in3/blob/master/src/types/types.ts#L225)** :`number` - the number of request send when getting a first answer example: 3
- **[rpc](https://github.com/slockit/in3/blob/master/src/types/types.ts#L267)** :`string` *(optional)* - url of one or more rpc-endpoints to use. (list can be comma seperated)
- **[servers](https://github.com/slockit/in3/blob/master/src/types/types.ts#L271)** *(optional)* - the nodelist per chain
- **[signatureCount](https://github.com/slockit/in3/blob/master/src/types/types.ts#L211)** :`number` *(optional)* - number of signatures requested example: 2
- **[verifiedHashes](https://github.com/slockit/in3/blob/master/src/types/types.ts#L201)** :`string`[] *(optional)* - if the client sends a array of blockhashes the server will not deliver any signatures or blockheaders for these blocks, but only return a string with a number. This is automaticly updated by the cache, but can be overriden per request.

Returns:

an boolean confirming that the config has changed.

Example:

Request: 

```c
{
  "method":"in3_config",
  "params":[{
      "chainId":"0x5",
      "maxAttempts":4,
      "nodeLimit":10
      "servers":{
          "0x1": [
              "nodeList": [
                  {
                    "address":"0x1234567890123456789012345678901234567890",
                    "url":"https://mybootnode-A.com",
                    "props":"0xFFFF",
                  },
                  {
                    "address":"0x1234567890123456789012345678901234567890",
                    "url":"https://mybootnode-B.com",
                    "props":"0xFFFF",
                  }
              ]
          ]
      }

   }]
}
```
Response:

```c
{
  "id": 1,
  "result": true,
}
```
### in3_abiEncode

based on the [ABI-encoding](https://solidity.readthedocs.io/en/v0.5.3/abi-spec.html) used by solidity, this function encodes the values and returns it as hex-string.

Parameters:

1. `signature`: string - the signature of the function. e.g. `getBalance(uint256)`. The format is the same as used by solidity to create the functionhash. optional you can also add the return type, which in this case is ignored.
2. `params`: array - a array of arguments. the number of arguments must match the arguments in the signature.

Returns:

the ABI-encoded data as hex including the 4 byte function-signature. These data can be used for `eth_call` or to send a transaction.

Request:

```c
{
    "method":"in3_abiEncode",
    "params":[
        "getBalance(address)",
        ["0x1234567890123456789012345678901234567890"]
    ]
}
```
Response:

```c
{
  "id": 1,
  "result": "0xf8b2cb4f0000000000000000000000001234567890123456789012345678901234567890",
}
```
### in3_abiDecode

based on the [ABI-encoding](https://solidity.readthedocs.io/en/v0.5.3/abi-spec.html) used by solidity, this function decodes the bytes given and returns it as array of values.

Parameters:

1. `signature`: string - the signature of the function. e.g. `uint256`, `(address,string,uint256)` or `getBalance(address):uint256`. If the complete functionhash is given, only the return-part will be used.
2. `data`: hex - the data to decode (usually the result of a eth_call)

Returns:

a array (if more then one arguments in the result-type) or the the value after decodeing.

Request:

```c
{
    "method":"in3_abiDecode",
    "params":[
        "(address,uint256)",
        "0x00000000000000000000000012345678901234567890123456789012345678900000000000000000000000000000000000000000000000000000000000000005"
    ]
}
```
Response:

```c
{
  "id": 1,
  "result": ["0x1234567890123456789012345678901234567890","0x05"],
}
```
### in3_checksumAddress

Will convert an upper or lowercase Ethereum address to a checksum address. (See [EIP55](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-55.md) )

Parameters:

1. `address`: address - the address to convert.
2. `useChainId`: boolean - if true, the chainId is integrated as well (See [EIP1191](https://github.com/ethereum/EIPs/issues/1121) )

Returns:

the address-string using the upper/lowercase hex characters.

Request:

```c
{
    "method":"in3_checksumAddress",
    "params":[
        "0x1fe2e9bf29aa1938859af64c413361227d04059a",
        false
    ]
}
```
Response:

```c
{
  "id": 1,
  "result": "0x1Fe2E9bf29aa1938859Af64C413361227d04059a"
}
```



## Module api/eth1 




### eth_api.h

Ethereum API.

This header-file defines easy to use function, which are preparing the JSON-RPC-Request, which is then executed and verified by the incubed-client. 

File: [src/api/eth1/eth_api.h](https://github.com/slockit/in3-c/blob/master/src/api/eth1/eth_api.h)

#### BLKNUM (blk)

Initializer macros for eth_blknum_t. 

```c
#define BLKNUM (blk) ((eth_blknum_t){.u64 = blk, .is_u64 = true})
```


#### BLKNUM_LATEST ()

```c
#define BLKNUM_LATEST () ((eth_blknum_t){.def = BLK_LATEST, .is_u64 = false})
```


#### BLKNUM_EARLIEST ()

```c
#define BLKNUM_EARLIEST () ((eth_blknum_t){.def = BLK_EARLIEST, .is_u64 = false})
```


#### BLKNUM_PENDING ()

```c
#define BLKNUM_PENDING () ((eth_blknum_t){.def = BLK_PENDING, .is_u64 = false})
```


#### eth_tx_t

A transaction. 


The stuct contains following fields:

```eval_rst
========================= ======================= =========================================
`bytes32_t <#bytes32-t>`_  **hash**               the blockhash
`bytes32_t <#bytes32-t>`_  **block_hash**         hash of ther containnig block
``uint64_t``               **block_number**       number of the containing block
`address_t <#address-t>`_  **from**               sender of the tx
``uint64_t``               **gas**                gas send along
``uint64_t``               **gas_price**          gas price used
`bytes_t <#bytes-t>`_      **data**               data send along with the transaction
``uint64_t``               **nonce**              nonce of the transaction
`address_t <#address-t>`_  **to**                 receiver of the address 0x0000. 
                                                  
                                                  . -Address is used for contract creation.
`uint256_t <#uint256-t>`_  **value**              the value in wei send
``int``                    **transaction_index**  the transaction index
``uint8_t``                **signature**          signature of the transaction
========================= ======================= =========================================
```

#### eth_block_t

An Ethereum Block. 


The stuct contains following fields:

```eval_rst
=========================== ======================= ====================================================
``uint64_t``                 **number**             the blockNumber
`bytes32_t <#bytes32-t>`_    **hash**               the blockhash
``uint64_t``                 **gasUsed**            gas used by all the transactions
``uint64_t``                 **gasLimit**           gasLimit
`address_t <#address-t>`_    **author**             the author of the block.
`uint256_t <#uint256-t>`_    **difficulty**         the difficulty of the block.
`bytes_t <#bytes-t>`_        **extra_data**         the extra_data of the block.
``uint8_t``                  **logsBloom**          the logsBloom-data
`bytes32_t <#bytes32-t>`_    **parent_hash**        the hash of the parent-block
`bytes32_t <#bytes32-t>`_    **sha3_uncles**        root hash of the uncle-trie
`bytes32_t <#bytes32-t>`_    **state_root**         root hash of the state-trie
`bytes32_t <#bytes32-t>`_    **receipts_root**      root of the receipts trie
`bytes32_t <#bytes32-t>`_    **transaction_root**   root of the transaction trie
``int``                      **tx_count**           number of transactions in the block
`eth_tx_t * <#eth-tx-t>`_    **tx_data**            array of transaction data or NULL if not requested
`bytes32_t * <#bytes32-t>`_  **tx_hashes**          array of transaction hashes or NULL if not requested
``uint64_t``                 **timestamp**          the unix timestamp of the block
`bytes_t * <#bytes-t>`_      **seal_fields**        sealed fields
``int``                      **seal_fields_count**  number of seal fields
=========================== ======================= ====================================================
```

#### eth_log_t

A linked list of Ethereum Logs 


 


The stuct contains following fields:

```eval_rst
=============================== ======================= ==============================================================
``bool``                         **removed**            true when the log was removed, due to a chain reorganization. 
                                                        
                                                        false if its a valid log
``size_t``                       **log_index**          log index position in the block
``size_t``                       **transaction_index**  transactions index position log was created from
`bytes32_t <#bytes32-t>`_        **transaction_hash**   hash of the transactions this log was created from
`bytes32_t <#bytes32-t>`_        **block_hash**         hash of the block where this log was in
``uint64_t``                     **block_number**       the block number where this log was in
`address_t <#address-t>`_        **address**            address from which this log originated
`bytes_t <#bytes-t>`_            **data**               non-indexed arguments of the log
`bytes32_t * <#bytes32-t>`_      **topics**             array of 0 to 4 32 Bytes DATA of indexed log arguments
``size_t``                       **topic_count**        counter for topics
`eth_logstruct , * <#eth-log>`_  **next**               pointer to next log in list or NULL
=============================== ======================= ==============================================================
```

#### eth_tx_receipt_t

A transaction receipt. 


The stuct contains following fields:

```eval_rst
=========================== ========================= =============================================================================
`bytes32_t <#bytes32-t>`_    **transaction_hash**     the transaction hash
``int``                      **transaction_index**    the transaction index
`bytes32_t <#bytes32-t>`_    **block_hash**           hash of ther containnig block
``uint64_t``                 **block_number**         number of the containing block
``uint64_t``                 **cumulative_gas_used**  total amount of gas used by block
``uint64_t``                 **gas_used**             amount of gas used by this specific transaction
`bytes_t * <#bytes-t>`_      **contract_address**     contract address created (if the transaction was a contract creation) or NULL
``bool``                     **status**               1 if transaction succeeded, 0 otherwise.
`eth_log_t * <#eth-log-t>`_  **logs**                 array of log objects, which this transaction generated
=========================== ========================= =============================================================================
```

#### DEFINE_OPTIONAL_T

```c
DEFINE_OPTIONAL_T(uint64_t);
```

Optional types. 

arguments:
```eval_rst
============  
``uint64_t``  
============  
```
returns: ``


#### DEFINE_OPTIONAL_T

```c
DEFINE_OPTIONAL_T(bytes_t);
```

arguments:
```eval_rst
=====================  
`bytes_t <#bytes-t>`_  
=====================  
```
returns: ``


#### DEFINE_OPTIONAL_T

```c
DEFINE_OPTIONAL_T(address_t);
```

arguments:
```eval_rst
=========================  
`address_t <#address-t>`_  
=========================  
```
returns: ``


#### DEFINE_OPTIONAL_T

```c
DEFINE_OPTIONAL_T(uint256_t);
```

arguments:
```eval_rst
=========================  
`uint256_t <#uint256-t>`_  
=========================  
```
returns: ``


#### eth_getStorageAt

```c
uint256_t eth_getStorageAt(in3_t *in3, address_t account, bytes32_t key, eth_blknum_t block);
```

Returns the storage value of a given address. 

arguments:
```eval_rst
=============================== ============= 
`in3_t * <#in3-t>`_              **in3**      
`address_t <#address-t>`_        **account**  
`bytes32_t <#bytes32-t>`_        **key**      
`eth_blknum_t <#eth-blknum-t>`_  **block**    
=============================== ============= 
```
returns: [`uint256_t`](#uint256-t)


#### eth_getCode

```c
bytes_t eth_getCode(in3_t *in3, address_t account, eth_blknum_t block);
```

Returns the code of the account of given address. 

(Make sure you free the data-point of the result after use.) 

arguments:
```eval_rst
=============================== ============= 
`in3_t * <#in3-t>`_              **in3**      
`address_t <#address-t>`_        **account**  
`eth_blknum_t <#eth-blknum-t>`_  **block**    
=============================== ============= 
```
returns: [`bytes_t`](#bytes-t)


#### eth_getBalance

```c
uint256_t eth_getBalance(in3_t *in3, address_t account, eth_blknum_t block);
```

Returns the balance of the account of given address. 

arguments:
```eval_rst
=============================== ============= 
`in3_t * <#in3-t>`_              **in3**      
`address_t <#address-t>`_        **account**  
`eth_blknum_t <#eth-blknum-t>`_  **block**    
=============================== ============= 
```
returns: [`uint256_t`](#uint256-t)


#### eth_blockNumber

```c
uint64_t eth_blockNumber(in3_t *in3);
```

Returns the current price per gas in wei. 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
=================== ========= 
```
returns: `uint64_t`


#### eth_gasPrice

```c
uint64_t eth_gasPrice(in3_t *in3);
```

Returns the current blockNumber, if bn==0 an error occured and you should check eth_last_error() 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
=================== ========= 
```
returns: `uint64_t`


#### eth_getBlockByNumber

```c
eth_block_t* eth_getBlockByNumber(in3_t *in3, eth_blknum_t number, bool include_tx);
```

Returns the block for the given number (if number==0, the latest will be returned). 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it! 

arguments:
```eval_rst
=============================== ================ 
`in3_t * <#in3-t>`_              **in3**         
`eth_blknum_t <#eth-blknum-t>`_  **number**      
``bool``                         **include_tx**  
=============================== ================ 
```
returns: [`eth_block_t *`](#eth-block-t)


#### eth_getBlockByHash

```c
eth_block_t* eth_getBlockByHash(in3_t *in3, bytes32_t hash, bool include_tx);
```

Returns the block for the given hash. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it! 

arguments:
```eval_rst
========================= ================ 
`in3_t * <#in3-t>`_        **in3**         
`bytes32_t <#bytes32-t>`_  **hash**        
``bool``                   **include_tx**  
========================= ================ 
```
returns: [`eth_block_t *`](#eth-block-t)


#### eth_getLogs

```c
eth_log_t* eth_getLogs(in3_t *in3, char *fopt);
```

Returns a linked list of logs. 

If result is null, check eth_last_error()! otherwise make sure to free the log, its topics and data after using it! 

arguments:
```eval_rst
=================== ========== 
`in3_t * <#in3-t>`_  **in3**   
``char *``           **fopt**  
=================== ========== 
```
returns: [`eth_log_t *`](#eth-log-t)


#### eth_newFilter

```c
in3_ret_t eth_newFilter(in3_t *in3, json_ctx_t *options);
```

Creates a new event filter with specified options and returns its id (>0) on success or 0 on failure. 

arguments:
```eval_rst
============================= ============= 
`in3_t * <#in3-t>`_            **in3**      
`json_ctx_t * <#json-ctx-t>`_  **options**  
============================= ============= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_newBlockFilter

```c
in3_ret_t eth_newBlockFilter(in3_t *in3);
```

Creates a new block filter with specified options and returns its id (>0) on success or 0 on failure. 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
=================== ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_newPendingTransactionFilter

```c
in3_ret_t eth_newPendingTransactionFilter(in3_t *in3);
```

Creates a new pending txn filter with specified options and returns its id on success or 0 on failure. 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
=================== ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_uninstallFilter

```c
bool eth_uninstallFilter(in3_t *in3, size_t id);
```

Uninstalls a filter and returns true on success or false on failure. 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
``size_t``           **id**   
=================== ========= 
```
returns: `bool`


#### eth_getFilterChanges

```c
in3_ret_t eth_getFilterChanges(in3_t *in3, size_t id, bytes32_t **block_hashes, eth_log_t **logs);
```

Sets the logs (for event filter) or blockhashes (for block filter) that match a filter; returns <0 on error, otherwise no. 

of block hashes matched (for block filter) or 0 (for log filter) 

arguments:
```eval_rst
============================ ================== 
`in3_t * <#in3-t>`_           **in3**           
``size_t``                    **id**            
`bytes32_t ** <#bytes32-t>`_  **block_hashes**  
`eth_log_t ** <#eth-log-t>`_  **logs**          
============================ ================== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_getFilterLogs

```c
in3_ret_t eth_getFilterLogs(in3_t *in3, size_t id, eth_log_t **logs);
```

Sets the logs (for event filter) or blockhashes (for block filter) that match a filter; returns <0 on error, otherwise no. 

of block hashes matched (for block filter) or 0 (for log filter) 

arguments:
```eval_rst
============================ ========== 
`in3_t * <#in3-t>`_           **in3**   
``size_t``                    **id**    
`eth_log_t ** <#eth-log-t>`_  **logs**  
============================ ========== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_chainId

```c
uint64_t eth_chainId(in3_t *in3);
```

Returns the currently configured chain id. 

arguments:
```eval_rst
=================== ========= 
`in3_t * <#in3-t>`_  **in3**  
=================== ========= 
```
returns: `uint64_t`


#### eth_getBlockTransactionCountByHash

```c
uint64_t eth_getBlockTransactionCountByHash(in3_t *in3, bytes32_t hash);
```

Returns the number of transactions in a block from a block matching the given block hash. 

arguments:
```eval_rst
========================= ========== 
`in3_t * <#in3-t>`_        **in3**   
`bytes32_t <#bytes32-t>`_  **hash**  
========================= ========== 
```
returns: `uint64_t`


#### eth_getBlockTransactionCountByNumber

```c
uint64_t eth_getBlockTransactionCountByNumber(in3_t *in3, eth_blknum_t block);
```

Returns the number of transactions in a block from a block matching the given block number. 

arguments:
```eval_rst
=============================== =========== 
`in3_t * <#in3-t>`_              **in3**    
`eth_blknum_t <#eth-blknum-t>`_  **block**  
=============================== =========== 
```
returns: `uint64_t`


#### eth_call_fn

```c
json_ctx_t* eth_call_fn(in3_t *in3, address_t contract, eth_blknum_t block, char *fn_sig,...);
```

Returns the result of a function_call. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it with json_free()! 

arguments:
```eval_rst
=============================== ============== 
`in3_t * <#in3-t>`_              **in3**       
`address_t <#address-t>`_        **contract**  
`eth_blknum_t <#eth-blknum-t>`_  **block**     
``char *``                       **fn_sig**    
``...``                                        
=============================== ============== 
```
returns: [`json_ctx_t *`](#json-ctx-t)


#### eth_estimate_fn

```c
uint64_t eth_estimate_fn(in3_t *in3, address_t contract, eth_blknum_t block, char *fn_sig,...);
```

Returns the result of a function_call. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it with json_free()! 

arguments:
```eval_rst
=============================== ============== 
`in3_t * <#in3-t>`_              **in3**       
`address_t <#address-t>`_        **contract**  
`eth_blknum_t <#eth-blknum-t>`_  **block**     
``char *``                       **fn_sig**    
``...``                                        
=============================== ============== 
```
returns: `uint64_t`


#### eth_getTransactionByHash

```c
eth_tx_t* eth_getTransactionByHash(in3_t *in3, bytes32_t tx_hash);
```

Returns the information about a transaction requested by transaction hash. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it! 

arguments:
```eval_rst
========================= ============= 
`in3_t * <#in3-t>`_        **in3**      
`bytes32_t <#bytes32-t>`_  **tx_hash**  
========================= ============= 
```
returns: [`eth_tx_t *`](#eth-tx-t)


#### eth_getTransactionByBlockHashAndIndex

```c
eth_tx_t* eth_getTransactionByBlockHashAndIndex(in3_t *in3, bytes32_t block_hash, size_t index);
```

Returns the information about a transaction by block hash and transaction index position. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it! 

arguments:
```eval_rst
========================= ================ 
`in3_t * <#in3-t>`_        **in3**         
`bytes32_t <#bytes32-t>`_  **block_hash**  
``size_t``                 **index**       
========================= ================ 
```
returns: [`eth_tx_t *`](#eth-tx-t)


#### eth_getTransactionByBlockNumberAndIndex

```c
eth_tx_t* eth_getTransactionByBlockNumberAndIndex(in3_t *in3, eth_blknum_t block, size_t index);
```

Returns the information about a transaction by block number and transaction index position. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it! 

arguments:
```eval_rst
=============================== =========== 
`in3_t * <#in3-t>`_              **in3**    
`eth_blknum_t <#eth-blknum-t>`_  **block**  
``size_t``                       **index**  
=============================== =========== 
```
returns: [`eth_tx_t *`](#eth-tx-t)


#### eth_getTransactionCount

```c
uint64_t eth_getTransactionCount(in3_t *in3, address_t address, eth_blknum_t block);
```

Returns the number of transactions sent from an address. 

arguments:
```eval_rst
=============================== ============= 
`in3_t * <#in3-t>`_              **in3**      
`address_t <#address-t>`_        **address**  
`eth_blknum_t <#eth-blknum-t>`_  **block**    
=============================== ============= 
```
returns: `uint64_t`


#### eth_getUncleByBlockNumberAndIndex

```c
eth_block_t* eth_getUncleByBlockNumberAndIndex(in3_t *in3, eth_blknum_t block, size_t index);
```

Returns information about a uncle of a block by number and uncle index position. 

If result is null, check eth_last_error()! otherwise make sure to free the result after using it! 

arguments:
```eval_rst
=============================== =========== 
`in3_t * <#in3-t>`_              **in3**    
`eth_blknum_t <#eth-blknum-t>`_  **block**  
``size_t``                       **index**  
=============================== =========== 
```
returns: [`eth_block_t *`](#eth-block-t)


#### eth_getUncleCountByBlockHash

```c
uint64_t eth_getUncleCountByBlockHash(in3_t *in3, bytes32_t hash);
```

Returns the number of uncles in a block from a block matching the given block hash. 

arguments:
```eval_rst
========================= ========== 
`in3_t * <#in3-t>`_        **in3**   
`bytes32_t <#bytes32-t>`_  **hash**  
========================= ========== 
```
returns: `uint64_t`


#### eth_getUncleCountByBlockNumber

```c
uint64_t eth_getUncleCountByBlockNumber(in3_t *in3, eth_blknum_t block);
```

Returns the number of uncles in a block from a block matching the given block number. 

arguments:
```eval_rst
=============================== =========== 
`in3_t * <#in3-t>`_              **in3**    
`eth_blknum_t <#eth-blknum-t>`_  **block**  
=============================== =========== 
```
returns: `uint64_t`


#### eth_sendTransaction

```c
bytes_t* eth_sendTransaction(in3_t *in3, address_t from, address_t to, OPTIONAL_T(uint64_t) gas, OPTIONAL_T(uint64_t) gas_price, OPTIONAL_T(uint256_t) value, OPTIONAL_T(bytes_t) data, OPTIONAL_T(uint64_t) nonce);
```

Creates new message call transaction or a contract creation. 

Returns (32 Bytes) - the transaction hash, or the zero hash if the transaction is not yet available. Free result after use with b_free(). 

arguments:
```eval_rst
===================================== =============== 
`in3_t * <#in3-t>`_                    **in3**        
`address_t <#address-t>`_              **from**       
`address_t <#address-t>`_              **to**         
`OPTIONAL_T(uint64_t) <#optional-t>`_  **gas**        
`OPTIONAL_T(uint64_t) <#optional-t>`_  **gas_price**  
``(,)``                                **value**      
``(,)``                                **data**       
`OPTIONAL_T(uint64_t) <#optional-t>`_  **nonce**      
===================================== =============== 
```
returns: [`bytes_t *`](#bytes-t)


#### eth_sendRawTransaction

```c
bytes_t* eth_sendRawTransaction(in3_t *in3, bytes_t data);
```

Creates new message call transaction or a contract creation for signed transactions. 

Returns (32 Bytes) - the transaction hash, or the zero hash if the transaction is not yet available. Free after use with b_free(). 

arguments:
```eval_rst
===================== ========== 
`in3_t * <#in3-t>`_    **in3**   
`bytes_t <#bytes-t>`_  **data**  
===================== ========== 
```
returns: [`bytes_t *`](#bytes-t)


#### eth_getTransactionReceipt

```c
eth_tx_receipt_t* eth_getTransactionReceipt(in3_t *in3, bytes32_t tx_hash);
```

Returns the receipt of a transaction by transaction hash. 

Free result after use with eth_tx_receipt_free() 

arguments:
```eval_rst
========================= ============= 
`in3_t * <#in3-t>`_        **in3**      
`bytes32_t <#bytes32-t>`_  **tx_hash**  
========================= ============= 
```
returns: [`eth_tx_receipt_t *`](#eth-tx-receipt-t)


#### eth_wait_for_receipt

```c
char* eth_wait_for_receipt(in3_t *in3, bytes32_t tx_hash);
```

Waits for receipt of a transaction requested by transaction hash. 

arguments:
```eval_rst
========================= ============= 
`in3_t * <#in3-t>`_        **in3**      
`bytes32_t <#bytes32-t>`_  **tx_hash**  
========================= ============= 
```
returns: `char *`


#### eth_last_error

```c
char* eth_last_error();
```

The current error or null if all is ok. 

returns: `char *`


#### as_double

```c
long double as_double(uint256_t d);
```

Converts a uint256_t in a long double. 

Important: since a long double stores max 16 byte, there is no guarantee to have the full precision.

Converts a uint256_t in a long double. 

arguments:
```eval_rst
========================= ======= 
`uint256_t <#uint256-t>`_  **d**  
========================= ======= 
```
returns: `long double`


#### as_long

```c
uint64_t as_long(uint256_t d);
```

Converts a uint256_t in a long . 

Important: since a long double stores 8 byte, this will only use the last 8 byte of the value.

Converts a uint256_t in a long . 

arguments:
```eval_rst
========================= ======= 
`uint256_t <#uint256-t>`_  **d**  
========================= ======= 
```
returns: `uint64_t`


#### to_uint256

```c
uint256_t to_uint256(uint64_t value);
```

Converts a uint64_t into its uint256_t representation. 

arguments:
```eval_rst
============ =========== 
``uint64_t``  **value**  
============ =========== 
```
returns: [`uint256_t`](#uint256-t)


#### decrypt_key

```c
in3_ret_t decrypt_key(d_token_t *key_data, char *password, bytes32_t dst);
```

Decrypts the private key from a json keystore file using PBKDF2 or SCRYPT (if enabled) 

arguments:
```eval_rst
=========================== ============== 
`d_token_t * <#d-token-t>`_  **key_data**  
``char *``                   **password**  
`bytes32_t <#bytes32-t>`_    **dst**       
=========================== ============== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### log_free

```c
void log_free(eth_log_t *log);
```

Frees a eth_log_t object. 

arguments:
```eval_rst
=========================== ========= 
`eth_log_t * <#eth-log-t>`_  **log**  
=========================== ========= 
```

#### eth_tx_receipt_free

```c
void eth_tx_receipt_free(eth_tx_receipt_t *txr);
```

Frees a eth_tx_receipt_t object. 

arguments:
```eval_rst
========================================= ========= 
`eth_tx_receipt_t * <#eth-tx-receipt-t>`_  **txr**  
========================================= ========= 
```

#### to_checksum

```c
in3_ret_t to_checksum(address_t adr, chain_id_t chain_id, char out[43]);
```

converts the given address to a checksum address. 

If chain_id is passed, it will use the EIP1191 to include it as well. 

arguments:
```eval_rst
=========================== ============== 
`address_t <#address-t>`_    **adr**       
`chain_id_t <#chain-id-t>`_  **chain_id**  
``char``                     **out**       
=========================== ============== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_register_eth_api

```c
void in3_register_eth_api();
```


## Module api/usn 




### usn_api.h

USN API.

This header-file defines easy to use function, which are verifying USN-Messages. 

File: [src/api/usn/usn_api.h](https://github.com/slockit/in3-c/blob/master/src/api/usn/usn_api.h)

#### usn_booking_handler


```c
typedef int(* usn_booking_handler) (usn_event_t *)
```

returns: `int(*`


#### usn_verify_message

```c
usn_msg_result_t usn_verify_message(usn_device_conf_t *conf, char *message);
```

arguments:
```eval_rst
=========================================== ============= 
`usn_device_conf_t * <#usn-device-conf-t>`_  **conf**     
``char *``                                   **message**  
=========================================== ============= 
```
returns: [`usn_msg_result_t`](#usn-msg-result-t)


#### usn_register_device

```c
in3_ret_t usn_register_device(usn_device_conf_t *conf, char *url);
```

arguments:
```eval_rst
=========================================== ========== 
`usn_device_conf_t * <#usn-device-conf-t>`_  **conf**  
``char *``                                   **url**   
=========================================== ========== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### usn_parse_url

```c
usn_url_t usn_parse_url(char *url);
```

arguments:
```eval_rst
========== ========= 
``char *``  **url**  
========== ========= 
```
returns: [`usn_url_t`](#usn-url-t)


#### usn_update_state

```c
unsigned int usn_update_state(usn_device_conf_t *conf, unsigned int wait_time);
```

arguments:
```eval_rst
=========================================== =============== 
`usn_device_conf_t * <#usn-device-conf-t>`_  **conf**       
``unsigned int``                             **wait_time**  
=========================================== =============== 
```
returns: `unsigned int`


#### usn_update_bookings

```c
in3_ret_t usn_update_bookings(usn_device_conf_t *conf);
```

arguments:
```eval_rst
=========================================== ========== 
`usn_device_conf_t * <#usn-device-conf-t>`_  **conf**  
=========================================== ========== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### usn_remove_old_bookings

```c
void usn_remove_old_bookings(usn_device_conf_t *conf);
```

arguments:
```eval_rst
=========================================== ========== 
`usn_device_conf_t * <#usn-device-conf-t>`_  **conf**  
=========================================== ========== 
```

#### usn_get_next_event

```c
usn_event_t usn_get_next_event(usn_device_conf_t *conf);
```

arguments:
```eval_rst
=========================================== ========== 
`usn_device_conf_t * <#usn-device-conf-t>`_  **conf**  
=========================================== ========== 
```
returns: [`usn_event_t`](#usn-event-t)


#### usn_rent

```c
in3_ret_t usn_rent(in3_t *c, address_t contract, address_t token, char *url, uint32_t seconds, bytes32_t tx_hash);
```

arguments:
```eval_rst
========================= ============== 
`in3_t * <#in3-t>`_        **c**         
`address_t <#address-t>`_  **contract**  
`address_t <#address-t>`_  **token**     
``char *``                 **url**       
``uint32_t``               **seconds**   
`bytes32_t <#bytes32-t>`_  **tx_hash**   
========================= ============== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### usn_return

```c
in3_ret_t usn_return(in3_t *c, address_t contract, char *url, bytes32_t tx_hash);
```

arguments:
```eval_rst
========================= ============== 
`in3_t * <#in3-t>`_        **c**         
`address_t <#address-t>`_  **contract**  
``char *``                 **url**       
`bytes32_t <#bytes32-t>`_  **tx_hash**   
========================= ============== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### usn_price

```c
in3_ret_t usn_price(in3_t *c, address_t contract, address_t token, char *url, uint32_t seconds, address_t controller, bytes32_t price);
```

arguments:
```eval_rst
========================= ================ 
`in3_t * <#in3-t>`_        **c**           
`address_t <#address-t>`_  **contract**    
`address_t <#address-t>`_  **token**       
``char *``                 **url**         
``uint32_t``               **seconds**     
`address_t <#address-t>`_  **controller**  
`bytes32_t <#bytes32-t>`_  **price**       
========================= ================ 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


## Module core 




### client.h

this file defines the incubed configuration struct and it registration. 

File: [src/core/client/client.h](https://github.com/slockit/in3-c/blob/master/src/core/client/client.h)

#### IN3_PROTO_VER

the protocol version used when sending requests from the this client 

```c
#define IN3_PROTO_VER "2.1.0"
```


#### ETH_CHAIN_ID_MULTICHAIN

chain_id working with all known chains 

```c
#define ETH_CHAIN_ID_MULTICHAIN 0x0
```


#### ETH_CHAIN_ID_MAINNET

chain_id for mainnet 

```c
#define ETH_CHAIN_ID_MAINNET 0x01
```


#### ETH_CHAIN_ID_KOVAN

chain_id for kovan 

```c
#define ETH_CHAIN_ID_KOVAN 0x2a
```


#### ETH_CHAIN_ID_TOBALABA

chain_id for tobalaba 

```c
#define ETH_CHAIN_ID_TOBALABA 0x44d
```


#### ETH_CHAIN_ID_GOERLI

chain_id for goerlii 

```c
#define ETH_CHAIN_ID_GOERLI 0x5
```


#### ETH_CHAIN_ID_EVAN

chain_id for evan 

```c
#define ETH_CHAIN_ID_EVAN 0x4b1
```


#### ETH_CHAIN_ID_IPFS

chain_id for ipfs 

```c
#define ETH_CHAIN_ID_IPFS 0x7d0
```


#### ETH_CHAIN_ID_VOLTA

chain_id for volta 

```c
#define ETH_CHAIN_ID_VOLTA 0x12046
```


#### ETH_CHAIN_ID_LOCAL

chain_id for local chain 

```c
#define ETH_CHAIN_ID_LOCAL 0xFFFF
```


#### in3_node_props_init (np)

Initializer for in3_node_props_t. 

```c
#define in3_node_props_init (np) *(np) = 0
```


#### IN3_SIGN_ERR_REJECTED

return value used by the signer if the the signature-request was rejected. 

```c
#define IN3_SIGN_ERR_REJECTED -1
```


#### IN3_SIGN_ERR_ACCOUNT_NOT_FOUND

return value used by the signer if the requested account was not found. 

```c
#define IN3_SIGN_ERR_ACCOUNT_NOT_FOUND -2
```


#### IN3_SIGN_ERR_INVALID_MESSAGE

return value used by the signer if the message was invalid. 

```c
#define IN3_SIGN_ERR_INVALID_MESSAGE -3
```


#### IN3_SIGN_ERR_GENERAL_ERROR

return value used by the signer for unspecified errors. 

```c
#define IN3_SIGN_ERR_GENERAL_ERROR -4
```


#### chain_id_t

type for a chain_id. 


```c
typedef uint32_t chain_id_t
```

#### in3_request_config_t

the configuration as part of each incubed request. 

This will be generated for each request based on the client-configuration. the verifier may access this during verification in order to check against the request. 


The stuct contains following fields:

```eval_rst
=========================================== ============================ ====================================================================
`chain_id_t <#chain-id-t>`_                  **chain_id**                the chain to be used. 
                                                                         
                                                                         this is holding the integer-value of the hexstring.
``uint8_t``                                  **include_code**            if true the code needed will always be devlivered.
``uint8_t``                                  **use_full_proof**          this flaqg is set, if the proof is set to "PROOF_FULL"
``uint8_t``                                  **use_binary**              this flaqg is set, the client should use binary-format
`bytes_t * <#bytes-t>`_                      **verified_hashes**         a list of blockhashes already verified. 
                                                                         
                                                                         The Server will not send any proof for them again .
``uint16_t``                                 **verified_hashes_length**  number of verified blockhashes
``uint16_t``                                 **latest_block**            the last blocknumber the nodelistz changed
``uint16_t``                                 **finality**                number of signatures( in percent) needed in order to reach finality.
`in3_verification_t <#in3-verification-t>`_  **verification**            Verification-type.
`bytes_t * <#bytes-t>`_                      **client_signature**        the signature of the client with the client key
`bytes_t * <#bytes-t>`_                      **signers**                 the addresses of servers requested to sign the blockhash
``uint8_t``                                  **signers_length**          number or addresses
=========================================== ============================ ====================================================================
```

#### in3_node_props_t

Node capabilities. 


```c
typedef uint64_t in3_node_props_t
```

#### in3_node_t

incubed node-configuration. 

These information are read from the Registry contract and stored in this struct representing a server or node. 


The stuct contains following fields:

```eval_rst
======================================= ================= ================================================================================================
`bytes_t * <#bytes-t>`_                  **address**      address of the server
``uint64_t``                             **deposit**      the deposit stored in the registry contract, which this would lose if it sends a wrong blockhash
``uint32_t``                             **index**        index within the nodelist, also used in the contract as key
``uint32_t``                             **capacity**     the maximal capacity able to handle
`in3_node_props_t <#in3-node-props-t>`_  **props**        used to identify the capabilities of the node. 
                                                          
                                                          See in3_node_props_type_t in nodelist.h
``char *``                               **url**          the url of the node
``bool``                                 **whitelisted**  boolean indicating if node exists in whiteList
======================================= ================= ================================================================================================
```

#### in3_node_weight_t

Weight or reputation of a node. 

Based on the past performance of the node a weight is calculated given faster nodes a higher weight and chance when selecting the next node from the nodelist. These weights will also be stored in the cache (if available) 


The stuct contains following fields:

```eval_rst
============ ========================= ========================================
``uint32_t``  **response_count**       counter for responses
``uint32_t``  **total_response_time**  total of all response times
``uint64_t``  **blacklisted_until**    if >0 this node is blacklisted until k. 
                                       
                                       k is a unix timestamp
``float``     **weight**               current weight
============ ========================= ========================================
```

#### in3_whitelist_t

defines a whitelist structure used for the nodelist. 


The stuct contains following fields:

```eval_rst
========================= ================== =================================================================================================================
`address_t <#address-t>`_  **contract**      address of whiteList contract. 
                                             
                                             If specified, whiteList is always auto-updated and manual whiteList is overridden
`bytes_t <#bytes-t>`_      **addresses**     serialized list of node addresses that constitute the whiteList
``uint64_t``               **last_block**    last blocknumber the whiteList was updated, which is used to detect changed in the whitelist
``bool``                   **needs_update**  if true the nodelist should be updated and will trigger a `in3_nodeList`-request before the next request is send.
========================= ================== =================================================================================================================
```

#### in3_chain_t

Chain definition inside incubed. 

for incubed a chain can be any distributed network or database with incubed support. 


The stuct contains following fields:

```eval_rst
=========================================== ===================== =================================================================================================================
`chain_id_t <#chain-id-t>`_                  **chain_id**         chain_id, which could be a free or based on the public ethereum networkId
`in3_chain_type_t <#in3-chain-type-t>`_      **type**             chaintype
``uint64_t``                                 **last_block**       last blocknumber the nodeList was updated, which is used to detect changed in the nodelist
``bool``                                     **needs_update**     if true the nodelist should be updated and will trigger a `in3_nodeList`-request before the next request is send.
``int``                                      **nodelist_length**  number of nodes in the nodeList
`in3_node_t * <#in3-node-t>`_                **nodelist**         array of nodes
`in3_node_weight_t * <#in3-node-weight-t>`_  **weights**          stats and weights recorded for each node
`bytes_t ** <#bytes-t>`_                     **init_addresses**   array of addresses of nodes that should always part of the nodeList
`bytes_t * <#bytes-t>`_                      **contract**         the address of the registry contract
`bytes32_t <#bytes32-t>`_                    **registry_id**      the identifier of the registry
``uint8_t``                                  **version**          version of the chain
`in3_whitelist_t * <#in3-whitelist-t>`_      **whitelist**        if set the whitelist of the addresses.
=========================================== ===================== =================================================================================================================
```

#### in3_storage_get_item

storage handler function for reading from cache. 


```c
typedef bytes_t*(* in3_storage_get_item) (void *cptr, char *key)
```

returns: [`bytes_t *(*`](#bytes-t) : the found result. if the key is found this function should return the values as bytes otherwise `NULL`. 




#### in3_storage_set_item

storage handler function for writing to the cache. 


```c
typedef void(* in3_storage_set_item) (void *cptr, char *key, bytes_t *value)
```


#### in3_storage_handler_t

storage handler to handle cache. 


The stuct contains following fields:

```eval_rst
=============================================== ============== ============================================================
`in3_storage_get_item <#in3-storage-get-item>`_  **get_item**  function pointer returning a stored value for the given key.
`in3_storage_set_item <#in3-storage-set-item>`_  **set_item**  function pointer setting a stored value for the given key.
``void *``                                       **cptr**      custom pointer which will will be passed to functions
=============================================== ============== ============================================================
```

#### in3_sign

signing function. 

signs the given data and write the signature to dst. the return value must be the number of bytes written to dst. In case of an error a negativ value must be returned. It should be one of the IN3_SIGN_ERR... values. 


```c
typedef in3_ret_t(* in3_sign) (void *ctx, d_signature_type_t type, bytes_t message, bytes_t account, uint8_t *dst)
```

returns: [`in3_ret_t(*`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_prepare_tx

transform transaction function. 

for multisigs, we need to change the transaction to gro through the ms. if the new_tx is not set within the function, it will use the old_tx. 


```c
typedef in3_ret_t(* in3_prepare_tx) (void *ctx, d_token_t *old_tx, json_ctx_t **new_tx)
```

returns: [`in3_ret_t(*`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_signer_t


The stuct contains following fields:

```eval_rst
=================================== ================ 
`in3_sign <#in3-sign>`_              **sign**        
`in3_prepare_tx <#in3-prepare-tx>`_  **prepare_tx**  
``void *``                           **wallet**      
=================================== ================ 
```

#### in3_response_t

response-object. 

if the error has a length>0 the response will be rejected 


The stuct contains following fields:

```eval_rst
=============== ============ ==================================
`sb_t <#sb-t>`_  **error**   a stringbuilder to add any errors!
`sb_t <#sb-t>`_  **result**  a stringbuilder to add the result
=============== ============ ==================================
```

#### in3_request_t

request-object. 

represents a RPC-request 


The stuct contains following fields:

```eval_rst
===================================== ============== ===================
``char *``                             **payload**   the payload to send
``char **``                            **urls**      array of urls
``int``                                **urls_len**  number of urls
`in3_response_t * <#in3-response-t>`_  **results**   
===================================== ============== ===================
```

#### in3_transport_send

the transport function to be implemented by the transport provider. 


```c
typedef in3_ret_t(* in3_transport_send) (in3_request_t *request)
```

returns: [`in3_ret_t(*`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_filter_t


The stuct contains following fields:

```eval_rst
========================================= ================ ==========================================================
`in3_filter_type_t <#in3-filter-type-t>`_  **type**        filter type: (event, block or pending)
``char *``                                 **options**     associated filter options
``uint64_t``                               **last_block**  block no. 
                                                           
                                                           when filter was created OR eth_getFilterChanges was called
``void(*``                                 **release**     method to release owned resources
========================================= ================ ==========================================================
```

#### in3_filter_handler_t

Handler which is added to client config in order to handle filter. 


The stuct contains following fields:

```eval_rst
================================== =========== ================
`in3_filter_t ** <#in3-filter-t>`_  **array**  
``size_t``                          **count**  array of filters
================================== =========== ================
```

#### in3_t

Incubed Configuration. 

This struct holds the configuration and also point to internal resources such as filters or chain configs. 


The stuct contains following fields:

```eval_rst
=================================================== ========================== =======================================================================================
``uint32_t``                                         **cache_timeout**         number of seconds requests can be cached.
``uint16_t``                                         **node_limit**            the limit of nodes to store in the client.
`bytes_t * <#bytes-t>`_                              **key**                   the client key to sign requests
``uint32_t``                                         **max_code_cache**        number of max bytes used to cache the code in memory
``uint32_t``                                         **max_block_cache**       number of number of blocks cached in memory
`in3_proof_t <#in3-proof-t>`_                        **proof**                 the type of proof used
``uint8_t``                                          **request_count**         the number of request send when getting a first answer
``uint8_t``                                          **signature_count**       the number of signatures used to proof the blockhash.
``uint64_t``                                         **min_deposit**           min stake of the server. 
                                                                               
                                                                               Only nodes owning at least this amount will be chosen.
``uint16_t``                                         **replace_latest_block**  if specified, the blocknumber *latest* will be replaced by blockNumber- specified value
``uint16_t``                                         **finality**              the number of signatures in percent required for the request
``uint16_t``                                         **max_attempts**          the max number of attempts before giving up
``uint32_t``                                         **timeout**               specifies the number of milliseconds before the request times out. 
                                                                               
                                                                               increasing may be helpful if the device uses a slow connection.
`chain_id_t <#chain-id-t>`_                          **chain_id**              servers to filter for the given chain. 
                                                                               
                                                                               The chain-id based on EIP-155.
``uint8_t``                                          **auto_update_list**      if true the nodelist will be automaticly updated if the last_block is newer
`in3_storage_handler_t * <#in3-storage-handler-t>`_  **cache**                 a cache handler offering 2 functions ( setItem(string,string), getItem(string) )
`in3_signer_t * <#in3-signer-t>`_                    **signer**                signer-struct managing a wallet
`in3_transport_send <#in3-transport-send>`_          **transport**             the transporthandler sending requests
``uint8_t``                                          **include_code**          includes the code when sending eth_call-requests
``uint8_t``                                          **use_binary**            if true the client will use binary format
``uint8_t``                                          **use_http**              if true the client will try to use http instead of https
``uint8_t``                                          **keep_in3**              if true the in3-section with the proof will also returned
`in3_chain_t * <#in3-chain-t>`_                      **chains**                chain spec and nodeList definitions
``uint16_t``                                         **chains_length**         number of configured chains
`in3_filter_handler_t * <#in3-filter-handler-t>`_    **filters**               filter handler
`in3_node_props_t <#in3-node-props-t>`_              **node_props**            used to identify the capabilities of the node.
=================================================== ========================== =======================================================================================
```

#### in3_node_props_set

```c
void in3_node_props_set(in3_node_props_t *node_props, in3_node_props_type_t type, uint8_t value);
```

setter method for interacting with in3_node_props_t. 

arguments:
```eval_rst
========================================= ================ 
`in3_node_props_t * <#in3-node-props-t>`_  **node_props**  
``in3_node_props_type_t``                  **type**        
``uint8_t``                                **value**       
========================================= ================ 
```

#### in3_node_props_get

```c
static uint32_t in3_node_props_get(in3_node_props_t np, in3_node_props_type_t t);
```

returns the value of the specified propertytype. 

arguments:
```eval_rst
======================================= ======== 
`in3_node_props_t <#in3-node-props-t>`_  **np**  
``in3_node_props_type_t``                **t**   
======================================= ======== 
```
returns: `uint32_t` : value as a number 




#### in3_node_props_matches

```c
static bool in3_node_props_matches(in3_node_props_t np, in3_node_props_type_t t);
```

checkes if the given type is set in the properties 

arguments:
```eval_rst
======================================= ======== 
`in3_node_props_t <#in3-node-props-t>`_  **np**  
``in3_node_props_type_t``                **t**   
======================================= ======== 
```
returns: `bool` : true if set 




#### in3_new

```c
in3_t* in3_new() __attribute__((deprecated("use in3_for_chain(ETH_CHAIN_ID_MULTICHAIN)")));
```

creates a new Incubes configuration and returns the pointer. 

This Method is depricated. you should use `in3_for_chain(ETH_CHAIN_ID_MULTICHAIN)` instead.

you need to free this instance with `in3_free` after use!

Before using the client you still need to set the tramsport and optional the storage handlers:

- example of initialization: 

```c
// register verifiers
in3_register_eth_full();

// create new client
in3_t* client = in3_new();

// configure storage...
in3_storage_handler_t storage_handler;
storage_handler.get_item = storage_get_item;
storage_handler.set_item = storage_set_item;

// configure transport
client->transport    = send_curl;

// configure storage
client->cache = &storage_handler;

// init cache
in3_cache_init(client);

// ready to use ...
```

returns: [`in3_t *`](#in3-t) : the incubed instance. 




#### in3_for_chain

```c
in3_t* in3_for_chain(chain_id_t chain_id);
```

creates a new Incubes configuration for a specified chain and returns the pointer. 

when creating the client only the one chain will be configured. (saves memory). but if you pass `ETH_CHAIN_ID_MULTICHAIN` as argument all known chains will be configured allowing you to switch between chains within the same client or configuring your own chain.

you need to free this instance with `in3_free` after use!

Before using the client you still need to set the tramsport and optional the storage handlers:

- example of initialization: , ** This Method is depricated. you should use `in3_for_chain` instead.**

```c
// register verifiers
in3_register_eth_full();

// create new client
in3_t* client = in3_for_chain(ETH_CHAIN_ID_MAINNET);

// configure storage...
in3_storage_handler_t storage_handler;
storage_handler.get_item = storage_get_item;
storage_handler.set_item = storage_set_item;

// configure transport
client->transport    = send_curl;

// configure storage
client->cache = &storage_handler;

// init cache
in3_cache_init(client);

// ready to use ...
```

arguments:
```eval_rst
=========================== ============== ==============================================
`chain_id_t <#chain-id-t>`_  **chain_id**  the chain_id (see ETH_CHAIN_ID_... constants).
=========================== ============== ==============================================
```
returns: [`in3_t *`](#in3-t) : the incubed instance. 




#### in3_client_rpc

```c
in3_ret_t in3_client_rpc(in3_t *c, char *method, char *params, char **result, char **error);
```

sends a request and stores the result in the provided buffer 

arguments:
```eval_rst
=================== ============ ======================================================================================================================================================
`in3_t * <#in3-t>`_  **c**       the pointer to the incubed client config.
``char *``           **method**  the name of the rpc-funcgtion to call.
``char *``           **params**  docs for input parameter v.
``char **``          **result**  pointer to string which will be set if the request was successfull. This will hold the result as json-rpc-string. (make sure you free this after use!)
``char **``          **error**   pointer to a string containg the error-message. (make sure you free it after use!)
=================== ============ ======================================================================================================================================================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_client_exec_req

```c
char* in3_client_exec_req(in3_t *c, char *req);
```

executes a request and returns result as string. 

in case of an error, the error-property of the result will be set. The resulting string must be free by the the caller of this function! 

arguments:
```eval_rst
=================== ========= =========================================
`in3_t * <#in3-t>`_  **c**    the pointer to the incubed client config.
``char *``           **req**  the request as rpc.
=================== ========= =========================================
```
returns: `char *`


#### in3_req_add_response

```c
void in3_req_add_response(in3_response_t *res, int index, bool is_error, void *data, int data_len);
```

adds a response for a request-object. 

This function should be used in the transport-function to set the response. 

arguments:
```eval_rst
===================================== ============== =====================================================================================
`in3_response_t * <#in3-response-t>`_  **res**       the response-pointer
``int``                                **index**     the index of the url, since this request could go out to many urls
``bool``                               **is_error**  if true this will be reported as error. the message should then be the error-message
``void *``                             **data**      the data or the the string
``int``                                **data_len**  the length of the data or the the string (use -1 if data is a null terminated string)
===================================== ============== =====================================================================================
```

#### in3_client_register_chain

```c
in3_ret_t in3_client_register_chain(in3_t *client, chain_id_t chain_id, in3_chain_type_t type, address_t contract, bytes32_t registry_id, uint8_t version, address_t wl_contract);
```

registers a new chain or replaces a existing (but keeps the nodelist) 

arguments:
```eval_rst
======================================= ================= =========================================
`in3_t * <#in3-t>`_                      **client**       the pointer to the incubed client config.
`chain_id_t <#chain-id-t>`_              **chain_id**     the chain id.
`in3_chain_type_t <#in3-chain-type-t>`_  **type**         the verification type of the chain.
`address_t <#address-t>`_                **contract**     contract of the registry.
`bytes32_t <#bytes32-t>`_                **registry_id**  the identifier of the registry.
``uint8_t``                              **version**      the chain version.
`address_t <#address-t>`_                **wl_contract**  contract of whiteList.
======================================= ================= =========================================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_client_add_node

```c
in3_ret_t in3_client_add_node(in3_t *client, chain_id_t chain_id, char *url, in3_node_props_t props, address_t address);
```

adds a node to a chain ore updates a existing node 

[in] public address of the signer. 

arguments:
```eval_rst
======================================= ============== =========================================
`in3_t * <#in3-t>`_                      **client**    the pointer to the incubed client config.
`chain_id_t <#chain-id-t>`_              **chain_id**  the chain id.
``char *``                               **url**       url of the nodes.
`in3_node_props_t <#in3-node-props-t>`_  **props**     properties of the node.
`address_t <#address-t>`_                **address**   
======================================= ============== =========================================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_client_remove_node

```c
in3_ret_t in3_client_remove_node(in3_t *client, chain_id_t chain_id, address_t address);
```

removes a node from a nodelist 

[in] public address of the signer. 

arguments:
```eval_rst
=========================== ============== =========================================
`in3_t * <#in3-t>`_          **client**    the pointer to the incubed client config.
`chain_id_t <#chain-id-t>`_  **chain_id**  the chain id.
`address_t <#address-t>`_    **address**   
=========================== ============== =========================================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_client_clear_nodes

```c
in3_ret_t in3_client_clear_nodes(in3_t *client, chain_id_t chain_id);
```

removes all nodes from the nodelist 

[in] the chain id. 

arguments:
```eval_rst
=========================== ============== =========================================
`in3_t * <#in3-t>`_          **client**    the pointer to the incubed client config.
`chain_id_t <#chain-id-t>`_  **chain_id**  
=========================== ============== =========================================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_free

```c
void in3_free(in3_t *a);
```

frees the references of the client 

arguments:
```eval_rst
=================== ======= =================================================
`in3_t * <#in3-t>`_  **a**  the pointer to the incubed client config to free.
=================== ======= =================================================
```

#### in3_cache_init

```c
in3_ret_t in3_cache_init(in3_t *c);
```

inits the cache. 

this will try to read the nodelist from cache. 

arguments:
```eval_rst
=================== ======= ==================
`in3_t * <#in3-t>`_  **c**  the incubed client
=================== ======= ==================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_find_chain

```c
in3_chain_t* in3_find_chain(in3_t *c, chain_id_t chain_id);
```

finds the chain-config for the given chain_id. 

My return NULL if not found. 

arguments:
```eval_rst
=========================== ============== ==================
`in3_t * <#in3-t>`_          **c**         the incubed client
`chain_id_t <#chain-id-t>`_  **chain_id**  chain_id
=========================== ============== ==================
```
returns: [`in3_chain_t *`](#in3-chain-t)


#### in3_configure

```c
in3_ret_t in3_configure(in3_t *c, char *config);
```

configures the clent based on a json-config. 

For details about the structure of ther config see [https://in3.readthedocs.io/en/develop/api-ts.html#type-in3config](https://in3.readthedocs.io/en/develop/api-ts.html#type-in3config) 

arguments:
```eval_rst
=================== ============ ==========================================
`in3_t * <#in3-t>`_  **c**       the incubed client
``char *``           **config**  JSON-string with the configuration to set.
=================== ============ ==========================================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_set_default_transport

```c
void in3_set_default_transport(in3_transport_send transport);
```

defines a default transport which is used when creating a new client. 

arguments:
```eval_rst
=========================================== =============== ===============================
`in3_transport_send <#in3-transport-send>`_  **transport**  the default transport-function.
=========================================== =============== ===============================
```

#### in3_set_default_storage

```c
void in3_set_default_storage(in3_storage_handler_t *cacheStorage);
```

defines a default storage handler which is used when creating a new client. 

arguments:
```eval_rst
=================================================== ================== =============================
`in3_storage_handler_t * <#in3-storage-handler-t>`_  **cacheStorage**  pointer to the handler-struct
=================================================== ================== =============================
```

#### in3_set_default_signer

```c
void in3_set_default_signer(in3_signer_t *signer);
```

defines a default signer which is used when creating a new client. 

arguments:
```eval_rst
================================= ============ ========================
`in3_signer_t * <#in3-signer-t>`_  **signer**  default signer-function.
================================= ============ ========================
```

#### in3_create_signer

```c
in3_signer_t* in3_create_signer(in3_sign sign, in3_prepare_tx prepare_tx, void *wallet);
```

create a new signer-object to be set on the client. 

the caller will need to free this pointer after usage. 

arguments:
```eval_rst
=================================== ================ ===================================================================================================================================
`in3_sign <#in3-sign>`_              **sign**        function pointer returning a stored value for the given key.
`in3_prepare_tx <#in3-prepare-tx>`_  **prepare_tx**  function pointer returning capable of manipulating the transaction before signing it. This is needed in order to support multisigs.
``void *``                           **wallet**      custom object whill will be passed to functions
=================================== ================ ===================================================================================================================================
```
returns: [`in3_signer_t *`](#in3-signer-t)


#### in3_create_storeage_handler

```c
in3_storage_handler_t* in3_create_storeage_handler(in3_storage_get_item get_item, in3_storage_set_item set_item, void *cptr);
```

create a new storage handler-object to be set on the client. 

the caller will need to free this pointer after usage. 

arguments:
```eval_rst
=============================================== ============== ============================================================
`in3_storage_get_item <#in3-storage-get-item>`_  **get_item**  function pointer returning a stored value for the given key.
`in3_storage_set_item <#in3-storage-set-item>`_  **set_item**  function pointer setting a stored value for the given key.
``void *``                                       **cptr**      custom pointer which will will be passed to functions
=============================================== ============== ============================================================
```
returns: [`in3_storage_handler_t *`](#in3-storage-handler-t)


### context.h

Request Context. This is used for each request holding request and response-pointers but also controls the execution process. 

File: [src/core/client/context.h](https://github.com/slockit/in3-c/blob/master/src/core/client/context.h)

#### ctx_set_error (c,msg,err)

```c
#define ctx_set_error (c,msg,err) ctx_set_error_intern(c, NULL, err)
```


#### ctx_type_t

type of the request context, 


The enum type contains the following values:

```eval_rst
============= = ============================================================
 **CT_RPC**   0 a json-rpc request, which needs to be send to a incubed node
 **CT_SIGN**  1 a sign request
============= = ============================================================
```

#### node_weight_t

the weight of a certain node as linked list. 

This will be used when picking the nodes to send the request to. A linked list of these structs desribe the result. 


The stuct contains following fields:

```eval_rst
=========================================== ============ ==========================================================
`in3_node_t * <#in3-node-t>`_                **node**    the node definition including the url
`in3_node_weight_t * <#in3-node-weight-t>`_  **weight**  the current weight and blacklisting-stats
``float``                                    **s**       The starting value.
``float``                                    **w**       weight value
`weightstruct , * <#weight>`_                **next**    next in the linkedlist or NULL if this is the last element
=========================================== ============ ==========================================================
```

#### in3_ctx_t

The Request config. 

This is generated for each request and represents the current state. it holds the state until the request is finished and must be freed afterwards. 


The stuct contains following fields:

```eval_rst
================================================= ======================== =========================================================================================================
`ctx_type_t <#ctx-type-t>`_                        **type**                the type of the request
`in3_t * <#in3-t>`_                                **client**              reference to the client
`json_ctx_t * <#json-ctx-t>`_                      **request_context**     the result of the json-parser for the request.
`json_ctx_t * <#json-ctx-t>`_                      **response_context**    the result of the json-parser for the response.
``char *``                                         **error**               in case of an error this will hold the message, if not it points to `NULL`
``int``                                            **len**                 the number of requests
``int``                                            **attempt**             the number of attempts
`d_token_t ** <#d-token-t>`_                       **responses**           references to the tokens representring the parsed responses
`d_token_t ** <#d-token-t>`_                       **requests**            references to the tokens representring the requests
`in3_request_config_t * <#in3-request-config-t>`_  **requests_configs**    array of configs adjusted for each request.
`node_weight_t * <#node-weight-t>`_                **nodes**               
`cache_entry_t * <#cache-entry-t>`_                **cache**               optional cache-entries. 
                                                                           
                                                                           These entries will be freed when cleaning up the context.
`in3_response_t * <#in3-response-t>`_              **raw_response**        the raw response-data, which should be verified.
`in3_ctxstruct , * <#in3-ctx>`_                    **required**            pointer to the next required context. 
                                                                           
                                                                           if not NULL the data from this context need get finished first, before being able to resume this context.
`in3_ret_t <#in3-ret-t>`_                          **verification_state**  state of the verification
================================================= ======================== =========================================================================================================
```

#### in3_ctx_state_t

The current state of the context. 

you can check this state after each execute-call. 


The enum type contains the following values:

```eval_rst
================================== == ============================================================
 **CTX_SUCCESS**                   0  The ctx has a verified result.
 **CTX_WAITING_FOR_REQUIRED_CTX**  1  there are required contexts, which need to be resolved first
 **CTX_WAITING_FOR_RESPONSE**      2  the response is not set yet
 **CTX_ERROR**                     -1 the request has a error
================================== == ============================================================
```

#### ctx_new

```c
in3_ctx_t* ctx_new(in3_t *client, char *req_data);
```

creates a new context. 

the request data will be parsed and represented in the context. calling this function will only parse the request data, but not send anything yet.

*Important*: the req_data will not be cloned but used during the execution. The caller of the this function is also responsible for freeing this string afterwards. 

arguments:
```eval_rst
=================== ============== ===============================
`in3_t * <#in3-t>`_  **client**    the client-config.
``char *``           **req_data**  the rpc-request as json string.
=================== ============== ===============================
```
returns: [`in3_ctx_t *`](#in3-ctx-t)


#### in3_send_ctx

```c
in3_ret_t in3_send_ctx(in3_ctx_t *ctx);
```

sends a previously created context to nodes and verifies it. 

The execution happens within the same thread, thich mean it will be blocked until the response ha beedn received and verified. In order to handle calls asynchronously, you need to call the `in3_ctx_execute` function and provide the data as needed. 

arguments:
```eval_rst
=========================== ========= ====================
`in3_ctx_t * <#in3-ctx-t>`_  **ctx**  the request context.
=========================== ========= ====================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_ctx_execute

```c
in3_ret_t in3_ctx_execute(in3_ctx_t *ctx);
```

tries to execute the context, but stops whenever data are required. 

This function should be used in order to call data in a asyncronous way, since this function will not use the transport-function to actually send it.

The caller is responsible for delivering the required responses. After calling you need to check the return-value:

- IN3_WAITING : provide the required data and then call in3_ctx_execute again.
- IN3_OK : success, we have a result.
- any other status = error

Here is a example how to use this function:

```c
 in3_ret_t in3_send_ctx(in3_ctx_t* ctx) {
  in3_ret_t ret;
  // execute the context and store the return value.
  // if the return value is 0 == IN3_OK, it was successful and we return,
  // if not, we keep on executing
  while ((ret = in3_ctx_execute(ctx))) {
    // error we stop here, because this means we got an error
    if (ret != IN3_WAITING) return ret;

    // handle subcontexts first, if they have not been finished
    while (ctx->required && in3_ctx_state(ctx->required) != CTX_SUCCESS) {
      // exxecute them, and return the status if still waiting or error
      if ((ret = in3_send_ctx(ctx->required))) return ret;

      // recheck in order to prepare the request.
      // if it is not waiting, then it we cannot do much, becaus it will an error or successfull.
      if ((ret = in3_ctx_execute(ctx)) != IN3_WAITING) return ret;
    }

    // only if there is no response yet...
    if (!ctx->raw_response) {

      // what kind of request do we need to provide?
      switch (ctx->type) {

        // RPC-request to send to the nodes
        case CT_RPC: {

            // build the request
            in3_request_t* request = in3_create_request(ctx);

            // here we use the transport, but you can also try to fetch the data in any other way.
            ctx->client->transport(request);

            // clean up
            request_free(request, ctx, false);
            break;
        }

        // this is a request to sign a transaction
        case CT_SIGN: {
            // read the data to sign from the request
            d_token_t* params = d_get(ctx->requests[0], K_PARAMS);
            // the data to sign
            bytes_t    data   = d_to_bytes(d_get_at(params, 0));
            // the account to sign with
            bytes_t    from   = d_to_bytes(d_get_at(params, 1));

            // prepare the response
            ctx->raw_response = _malloc(sizeof(in3_response_t));
            sb_init(&ctx->raw_response[0].error);
            sb_init(&ctx->raw_response[0].result);

            // data for the signature 
            uint8_t sig[65];
            // use the signer to create the signature
            ret = ctx->client->signer->sign(ctx, SIGN_EC_HASH, data, from, sig);
            // if it fails we report this as error
            if (ret < 0) return ctx_set_error(ctx, ctx->raw_response->error.data, ret);
            // otherwise we simply add the raw 65 bytes to the response.
            sb_add_range(&ctx->raw_response->result, (char*) sig, 0, 65);
        }
      }
    }
  }
  // done...
  return ret;
}
```
arguments:
```eval_rst
=========================== ========= ====================
`in3_ctx_t * <#in3-ctx-t>`_  **ctx**  the request context.
=========================== ========= ====================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_ctx_state

```c
in3_ctx_state_t in3_ctx_state(in3_ctx_t *ctx);
```

returns the current state of the context. 

arguments:
```eval_rst
=========================== ========= ====================
`in3_ctx_t * <#in3-ctx-t>`_  **ctx**  the request context.
=========================== ========= ====================
```
returns: [`in3_ctx_state_t`](#in3-ctx-state-t)


#### in3_create_request

```c
in3_request_t* in3_create_request(in3_ctx_t *ctx);
```

creates a request-object, which then need to be filled with the responses. 

each request object contains a array of reponse-objects. In order to set the response, you need to call

```c
// set a succesfull response
sb_add_chars(&request->results[0].result, my_response);
// set a error response
sb_add_chars(&request->results[0].error, my_error);
```
arguments:
```eval_rst
=========================== ========= ====================
`in3_ctx_t * <#in3-ctx-t>`_  **ctx**  the request context.
=========================== ========= ====================
```
returns: [`in3_request_t *`](#in3-request-t)


#### request_free

```c
void request_free(in3_request_t *req, const in3_ctx_t *ctx, bool response_free);
```

frees a previuosly allocated request. 

arguments:
```eval_rst
=================================== =================== ======================================================================================
`in3_request_t * <#in3-request-t>`_  **req**            the request.
`in3_ctx_tconst , * <#in3-ctx-t>`_   **ctx**            the request context.
``bool``                             **response_free**  if true the responses will freed also, but usually this is done when the ctx is freed.
=================================== =================== ======================================================================================
```

#### ctx_free

```c
void ctx_free(in3_ctx_t *ctx);
```

frees all resources allocated during the request. 

But this will not free the request string passed when creating the context! 

arguments:
```eval_rst
=========================== ========= ====================
`in3_ctx_t * <#in3-ctx-t>`_  **ctx**  the request context.
=========================== ========= ====================
```

#### ctx_add_required

```c
in3_ret_t ctx_add_required(in3_ctx_t *parent, in3_ctx_t *ctx);
```

adds a new context as a requirment. 

Whenever a verifier needs more data and wants to send a request, we should create the request and add it as dependency and stop.

If the function is called again, we need to search and see if the required status is now useable.

Here is an example of how to use it:

```c
in3_ret_t get_from_nodes(in3_ctx_t* parent, char* method, char* params, bytes_t* dst) {
  // check if the method is already existing
  in3_ctx_t* ctx = ctx_find_required(parent, method);
  if (ctx) {
    // found one - so we check if it is useable.
    switch (in3_ctx_state(ctx)) {
      // in case of an error, we report it back to the parent context
      case CTX_ERROR:
        return ctx_set_error(parent, ctx->error, IN3_EUNKNOWN);
      // if we are still waiting, we stop here and report it.
      case CTX_WAITING_FOR_REQUIRED_CTX:
      case CTX_WAITING_FOR_RESPONSE:
        return IN3_WAITING;

      // if it is useable, we can now handle the result.
      case CTX_SUCCESS: {
        d_token_t* r = d_get(ctx->responses[0], K_RESULT);
        if (r) {
          // we have a result, so write it back to the dst
          *dst = d_to_bytes(r);
          return IN3_OK;
        } else
          // or check the error and report it
          return ctx_check_response_error(parent, 0);
      }
    }
  }

  // no required context found yet, so we create one:

  // since this is a subrequest it will be freed when the parent is freed.
  // allocate memory for the request-string
  char* req = _malloc(strlen(method) + strlen(params) + 200);
  // create it
  sprintf(req, "{\"method\":\"%s\",\"jsonrpc\":\"2.0\",\"id\":1,\"params\":%s}", method, params);
  // and add the request context to the parent.
  return ctx_add_required(parent, ctx_new(parent->client, req));
}
```
arguments:
```eval_rst
=========================== ============ ===============================
`in3_ctx_t * <#in3-ctx-t>`_  **parent**  the current request context.
`in3_ctx_t * <#in3-ctx-t>`_  **ctx**     the new request context to add.
=========================== ============ ===============================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### ctx_find_required

```c
in3_ctx_t* ctx_find_required(const in3_ctx_t *parent, const char *method);
```

searches within the required request contextes for one with the given method. 

This method is used internaly to find a previously added context. 

arguments:
```eval_rst
================================== ============ ==============================
`in3_ctx_tconst , * <#in3-ctx-t>`_  **parent**  the current request context.
``const char *``                    **method**  the method of the rpc-request.
================================== ============ ==============================
```
returns: [`in3_ctx_t *`](#in3-ctx-t)


#### ctx_remove_required

```c
in3_ret_t ctx_remove_required(in3_ctx_t *parent, in3_ctx_t *ctx);
```

removes a required context after usage. 

removing will also call free_ctx to free resources. 

arguments:
```eval_rst
=========================== ============ ==============================
`in3_ctx_t * <#in3-ctx-t>`_  **parent**  the current request context.
`in3_ctx_t * <#in3-ctx-t>`_  **ctx**     the request context to remove.
=========================== ============ ==============================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### ctx_check_response_error

```c
in3_ret_t ctx_check_response_error(in3_ctx_t *c, int i);
```

check if the response contains a error-property and reports this as error in the context. 

arguments:
```eval_rst
=========================== ======= ============================================================================
`in3_ctx_t * <#in3-ctx-t>`_  **c**  the current request context.
``int``                      **i**  the index of the request to check (if this is a batch-request, otherwise 0).
=========================== ======= ============================================================================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### ctx_set_error_intern

```c
in3_ret_t ctx_set_error_intern(in3_ctx_t *c, char *msg, in3_ret_t errnumber);
```

sets the error message in the context. 

If there is a previous error it will append it. the return value will simply be passed so you can use it like

```c
return ctx_set_error(ctx, "wrong number of arguments", IN3_EINVAL)
```
arguments:
```eval_rst
=========================== =============== ===============================================
`in3_ctx_t * <#in3-ctx-t>`_  **c**          the current request context.
``char *``                   **msg**        the error message. (This string will be copied)
`in3_ret_t <#in3-ret-t>`_    **errnumber**  the error code to return
=========================== =============== ===============================================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### ctx_get_error

```c
in3_ret_t ctx_get_error(in3_ctx_t *ctx, int id);
```

determins the errorcode for the given request. 

arguments:
```eval_rst
=========================== ========= ============================================================================
`in3_ctx_t * <#in3-ctx-t>`_  **ctx**  the current request context.
``int``                      **id**   the index of the request to check (if this is a batch-request, otherwise 0).
=========================== ========= ============================================================================
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_client_rpc_ctx

```c
in3_ctx_t* in3_client_rpc_ctx(in3_t *c, char *method, char *params);
```

sends a request and returns a context used to access the result or errors. 

This context *MUST* be freed with ctx_free(ctx) after usage to release the resources. 

arguments:
```eval_rst
=================== ============ ===================
`in3_t * <#in3-t>`_  **c**       the clientt config.
``char *``           **method**  rpc method.
``char *``           **params**  params as string.
=================== ============ ===================
```
returns: [`in3_ctx_t *`](#in3-ctx-t)


### verifier.h

Verification Context. This context is passed to the verifier. 

File: [src/core/client/verifier.h](https://github.com/slockit/in3-c/blob/master/src/core/client/verifier.h)

#### vc_err (vc,msg)

```c
#define vc_err (vc,msg) vc_set_error(vc, NULL)
```


#### in3_verify

function to verify the result. 


```c
typedef in3_ret_t(* in3_verify) (in3_vctx_t *c)
```

returns: [`in3_ret_t(*`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_pre_handle

function which is called to fill the response before a request is triggered. 

This can be used to handle requests which don't need a node to response. 


```c
typedef in3_ret_t(* in3_pre_handle) (in3_ctx_t *ctx, in3_response_t **response)
```

returns: [`in3_ret_t(*`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_verifier_t


The stuct contains following fields:

```eval_rst
======================================= ================ 
`in3_verify <#in3-verify>`_              **verify**      
`in3_pre_handle <#in3-pre-handle>`_      **pre_handle**  
`in3_chain_type_t <#in3-chain-type-t>`_  **type**        
`verifierstruct , * <#verifier>`_        **next**        
======================================= ================ 
```

#### in3_get_verifier

```c
in3_verifier_t* in3_get_verifier(in3_chain_type_t type);
```

returns the verifier for the given chainType 

arguments:
```eval_rst
======================================= ========== 
`in3_chain_type_t <#in3-chain-type-t>`_  **type**  
======================================= ========== 
```
returns: [`in3_verifier_t *`](#in3-verifier-t)


#### in3_register_verifier

```c
void in3_register_verifier(in3_verifier_t *verifier);
```

arguments:
```eval_rst
===================================== ============== 
`in3_verifier_t * <#in3-verifier-t>`_  **verifier**  
===================================== ============== 
```

#### vc_set_error

```c
in3_ret_t vc_set_error(in3_vctx_t *vc, char *msg);
```

arguments:
```eval_rst
============================= ========= 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**   
``char *``                     **msg**  
============================= ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### bytes.h

util helper on byte arrays. 

File: [src/core/util/bytes.h](https://github.com/slockit/in3-c/blob/master/src/core/util/bytes.h)

#### bb_new ()

creates a new bytes_builder with a initial size of 32 bytes 

```c
#define bb_new () bb_newl(32)
```


#### bb_read (_bb_,_i_,_vptr_)

```c
#define bb_read (_bb_,_i_,_vptr_) bb_readl((_bb_), (_i_), (_vptr_), sizeof(*_vptr_))
```


#### bb_read_next (_bb_,_iptr_,_vptr_)

```c
#define bb_read_next (_bb_,_iptr_,_vptr_) do {                                          \
    size_t _l_ = sizeof(*_vptr_);               \
    bb_readl((_bb_), *(_iptr_), (_vptr_), _l_); \
    *(_iptr_) += _l_;                           \
  } while (0)
```


#### bb_readl (_bb_,_i_,_vptr_,_l_)

```c
#define bb_readl (_bb_,_i_,_vptr_,_l_) memcpy((_vptr_), (_bb_)->b.data + (_i_), _l_)
```


#### b_read (_b_,_i_,_vptr_)

```c
#define b_read (_b_,_i_,_vptr_) b_readl((_b_), (_i_), _vptr_, sizeof(*_vptr_))
```


#### b_readl (_b_,_i_,_vptr_,_l_)

```c
#define b_readl (_b_,_i_,_vptr_,_l_) memcpy(_vptr_, (_b_)->data + (_i_), (_l_))
```


#### address_t

pointer to a 20byte address 


```c
typedef uint8_t address_t[20]
```

#### bytes32_t

pointer to a 32byte word 


```c
typedef uint8_t bytes32_t[32]
```

#### wlen_t

number of bytes within a word (min 1byte but usually a uint) 


```c
typedef uint_fast8_t wlen_t
```

#### bytes_t

a byte array 


The stuct contains following fields:

```eval_rst
============= ========== =================================
``uint8_t *``  **data**  the byte-data
``uint32_t``   **len**   the length of the array ion bytes
============= ========== =================================
```

#### b_new

```c
bytes_t* b_new(const char *data, int len);
```

allocates a new byte array with 0 filled 

arguments:
```eval_rst
================ ========== 
``const char *``  **data**  
``int``           **len**   
================ ========== 
```
returns: [`bytes_t *`](#bytes-t)


#### b_print

```c
void b_print(const bytes_t *a);
```

prints a the bytes as hex to stdout 

arguments:
```eval_rst
============================== ======= 
`bytes_tconst , * <#bytes-t>`_  **a**  
============================== ======= 
```

#### ba_print

```c
void ba_print(const uint8_t *a, size_t l);
```

prints a the bytes as hex to stdout 

arguments:
```eval_rst
=================== ======= 
``const uint8_t *``  **a**  
``size_t``           **l**  
=================== ======= 
```

#### b_cmp

```c
int b_cmp(const bytes_t *a, const bytes_t *b);
```

compares 2 byte arrays and returns 1 for equal and 0 for not equal 

arguments:
```eval_rst
============================== ======= 
`bytes_tconst , * <#bytes-t>`_  **a**  
`bytes_tconst , * <#bytes-t>`_  **b**  
============================== ======= 
```
returns: `int`


#### bytes_cmp

```c
int bytes_cmp(const bytes_t a, const bytes_t b);
```

compares 2 byte arrays and returns 1 for equal and 0 for not equal 

arguments:
```eval_rst
=========================== ======= 
`bytes_tconst  <#bytes-t>`_  **a**  
`bytes_tconst  <#bytes-t>`_  **b**  
=========================== ======= 
```
returns: `int`


#### b_free

```c
void b_free(bytes_t *a);
```

frees the data 

arguments:
```eval_rst
======================= ======= 
`bytes_t * <#bytes-t>`_  **a**  
======================= ======= 
```

#### b_dup

```c
bytes_t* b_dup(const bytes_t *a);
```

clones a byte array 

arguments:
```eval_rst
============================== ======= 
`bytes_tconst , * <#bytes-t>`_  **a**  
============================== ======= 
```
returns: [`bytes_t *`](#bytes-t)


#### b_read_byte

```c
uint8_t b_read_byte(bytes_t *b, size_t *pos);
```

reads a byte on the current position and updates the pos afterwards. 

arguments:
```eval_rst
======================= ========= 
`bytes_t * <#bytes-t>`_  **b**    
``size_t *``             **pos**  
======================= ========= 
```
returns: `uint8_t`


#### b_read_int

```c
uint32_t b_read_int(bytes_t *b, size_t *pos);
```

reads a integer on the current position and updates the pos afterwards. 

arguments:
```eval_rst
======================= ========= 
`bytes_t * <#bytes-t>`_  **b**    
``size_t *``             **pos**  
======================= ========= 
```
returns: `uint32_t`


#### b_read_long

```c
uint64_t b_read_long(bytes_t *b, size_t *pos);
```

reads a long on the current position and updates the pos afterwards. 

arguments:
```eval_rst
======================= ========= 
`bytes_t * <#bytes-t>`_  **b**    
``size_t *``             **pos**  
======================= ========= 
```
returns: `uint64_t`


#### b_new_chars

```c
char* b_new_chars(bytes_t *b, size_t *pos);
```

creates a new string (needs to be freed) on the current position and updates the pos afterwards. 

arguments:
```eval_rst
======================= ========= 
`bytes_t * <#bytes-t>`_  **b**    
``size_t *``             **pos**  
======================= ========= 
```
returns: `char *`


#### b_new_fixed_bytes

```c
bytes_t* b_new_fixed_bytes(bytes_t *b, size_t *pos, int len);
```

reads bytes with a fixed length on the current position and updates the pos afterwards. 

arguments:
```eval_rst
======================= ========= 
`bytes_t * <#bytes-t>`_  **b**    
``size_t *``             **pos**  
``int``                  **len**  
======================= ========= 
```
returns: [`bytes_t *`](#bytes-t)


#### bb_newl

```c
bytes_builder_t* bb_newl(size_t l);
```

creates a new bytes_builder 

arguments:
```eval_rst
========== ======= 
``size_t``  **l**  
========== ======= 
```
returns: [`bytes_builder_t *`](#bytes-builder-t)


#### bb_free

```c
void bb_free(bytes_builder_t *bb);
```

frees a bytebuilder and its content. 

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
======================================= ======== 
```

#### bb_check_size

```c
int bb_check_size(bytes_builder_t *bb, size_t len);
```

internal helper to increase the buffer if needed 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``size_t``                               **len**  
======================================= ========= 
```
returns: `int`


#### bb_write_chars

```c
void bb_write_chars(bytes_builder_t *bb, char *c, int len);
```

writes a string to the builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``char *``                               **c**    
``int``                                  **len**  
======================================= ========= 
```

#### bb_write_dyn_bytes

```c
void bb_write_dyn_bytes(bytes_builder_t *bb, const bytes_t *src);
```

writes bytes to the builder with a prefixed length. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
`bytes_tconst , * <#bytes-t>`_           **src**  
======================================= ========= 
```

#### bb_write_fixed_bytes

```c
void bb_write_fixed_bytes(bytes_builder_t *bb, const bytes_t *src);
```

writes fixed bytes to the builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
`bytes_tconst , * <#bytes-t>`_           **src**  
======================================= ========= 
```

#### bb_write_int

```c
void bb_write_int(bytes_builder_t *bb, uint32_t val);
```

writes a ineteger to the builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``uint32_t``                             **val**  
======================================= ========= 
```

#### bb_write_long

```c
void bb_write_long(bytes_builder_t *bb, uint64_t val);
```

writes s long to the builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``uint64_t``                             **val**  
======================================= ========= 
```

#### bb_write_long_be

```c
void bb_write_long_be(bytes_builder_t *bb, uint64_t val, int len);
```

writes any integer value with the given length of bytes 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``uint64_t``                             **val**  
``int``                                  **len**  
======================================= ========= 
```

#### bb_write_byte

```c
void bb_write_byte(bytes_builder_t *bb, uint8_t val);
```

writes a single byte to the builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``uint8_t``                              **val**  
======================================= ========= 
```

#### bb_write_raw_bytes

```c
void bb_write_raw_bytes(bytes_builder_t *bb, void *ptr, size_t len);
```

writes the bytes to the builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
``void *``                               **ptr**  
``size_t``                               **len**  
======================================= ========= 
```

#### bb_clear

```c
void bb_clear(bytes_builder_t *bb);
```

resets the content of the builder. 

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
======================================= ======== 
```

#### bb_replace

```c
void bb_replace(bytes_builder_t *bb, int offset, int delete_len, uint8_t *data, int data_len);
```

replaces or deletes a part of the content. 

arguments:
```eval_rst
======================================= ================ 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**          
``int``                                  **offset**      
``int``                                  **delete_len**  
``uint8_t *``                            **data**        
``int``                                  **data_len**    
======================================= ================ 
```

#### bb_move_to_bytes

```c
bytes_t* bb_move_to_bytes(bytes_builder_t *bb);
```

frees the builder and moves the content in a newly created bytes struct (which needs to be freed later). 

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
======================================= ======== 
```
returns: [`bytes_t *`](#bytes-t)


#### bb_read_long

```c
uint64_t bb_read_long(bytes_builder_t *bb, size_t *i);
```

reads a long from the builder 

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
``size_t *``                             **i**   
======================================= ======== 
```
returns: `uint64_t`


#### bb_read_int

```c
uint32_t bb_read_int(bytes_builder_t *bb, size_t *i);
```

reads a int from the builder 

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
``size_t *``                             **i**   
======================================= ======== 
```
returns: `uint32_t`


#### bytes

```c
static bytes_t bytes(uint8_t *a, uint32_t len);
```

converts the given bytes to a bytes struct 

arguments:
```eval_rst
============= ========= 
``uint8_t *``  **a**    
``uint32_t``   **len**  
============= ========= 
```
returns: [`bytes_t`](#bytes-t)


#### cloned_bytes

```c
bytes_t cloned_bytes(bytes_t data);
```

cloned the passed data 

arguments:
```eval_rst
===================== ========== 
`bytes_t <#bytes-t>`_  **data**  
===================== ========== 
```
returns: [`bytes_t`](#bytes-t)


#### b_optimize_len

```c
static void b_optimize_len(bytes_t *b);
```

< changed the data and len to remove leading 0-bytes

arguments:
```eval_rst
======================= ======= 
`bytes_t * <#bytes-t>`_  **b**  
======================= ======= 
```

### data.h

json-parser.

The parser can read from :

- json
- bin

When reading from json all '0x'... values will be stored as bytes_t. If the value is lower than 0xFFFFFFF, it is converted as integer. 

File: [src/core/util/data.h](https://github.com/slockit/in3-c/blob/master/src/core/util/data.h)

#### DATA_DEPTH_MAX

the max DEPTH of the JSON-data allowed. 

It will throw an error if reached. 

```c
#define DATA_DEPTH_MAX 11
```


#### printX

```c
#define printX printf
```


#### fprintX

```c
#define fprintX fprintf
```


#### snprintX

```c
#define snprintX snprintf
```


#### vprintX

```c
#define vprintX vprintf
```


#### d_key_t


```c
typedef uint16_t d_key_t
```

#### d_token_t

a token holding any kind of value. 

use d_type, d_len or the cast-function to get the value. 


The stuct contains following fields:

```eval_rst
============= ========== =====================================================================
``uint8_t *``  **data**  the byte or string-data
``uint32_t``   **len**   the length of the content (or number of properties) depending + type.
``d_key_t``    **key**   the key of the property.
============= ========== =====================================================================
```

#### str_range_t

internal type used to represent the a range within a string. 


The stuct contains following fields:

```eval_rst
========== ========== ==================================
``char *``  **data**  pointer to the start of the string
``size_t``  **len**   len of the characters
========== ========== ==================================
```

#### json_ctx_t

parser for json or binary-data. 

it needs to freed after usage. 


The stuct contains following fields:

```eval_rst
=========================== =============== ============================================================
`d_token_t * <#d-token-t>`_  **result**     the list of all tokens. 
                                            
                                            the first token is the main-token as returned by the parser.
``char *``                   **c**          
``size_t``                   **allocated**  pointer to the src-data
``size_t``                   **len**        amount of tokens allocated result
``size_t``                   **depth**      number of tokens in result
=========================== =============== ============================================================
```

#### d_iterator_t

iterator over elements of a array opf object. 

usage: 

```c
for (d_iterator_t iter = d_iter( parent ); iter.left ; d_iter_next(&iter)) {
  uint32_t val = d_int(iter.token);
}
```

The stuct contains following fields:

```eval_rst
=========================== =========== =====================
`d_token_t * <#d-token-t>`_  **token**  current token
``int``                      **left**   number of result left
=========================== =========== =====================
```

#### d_to_bytes

```c
bytes_t d_to_bytes(d_token_t *item);
```

returns the byte-representation of token. 

In case of a number it is returned as bigendian. booleans as 0x01 or 0x00 and NULL as 0x. Objects or arrays will return 0x. 

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
=========================== ========== 
```
returns: [`bytes_t`](#bytes-t)


#### d_bytes_to

```c
int d_bytes_to(d_token_t *item, uint8_t *dst, const int max);
```

writes the byte-representation to the dst. 

details see d_to_bytes. 

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
``uint8_t *``                **dst**   
``const int``                **max**   
=========================== ========== 
```
returns: `int`


#### d_bytes

```c
bytes_t* d_bytes(const d_token_t *item);
```

returns the value as bytes (Carefully, make sure that the token is a bytes-type!) 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: [`bytes_t *`](#bytes-t)


#### d_bytesl

```c
bytes_t* d_bytesl(d_token_t *item, size_t l);
```

returns the value as bytes with length l (may reallocates) 

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
``size_t``                   **l**     
=========================== ========== 
```
returns: [`bytes_t *`](#bytes-t)


#### d_string

```c
char* d_string(const d_token_t *item);
```

converts the value as string. 

Make sure the type is string! 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: `char *`


#### d_int

```c
int32_t d_int(const d_token_t *item);
```

returns the value as integer. 

only if type is integer 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: `int32_t`


#### d_intd

```c
int32_t d_intd(const d_token_t *item, const uint32_t def_val);
```

returns the value as integer or if NULL the default. 

only if type is integer 

arguments:
```eval_rst
================================== ============= 
`d_token_tconst , * <#d-token-t>`_  **item**     
``const uint32_t``                  **def_val**  
================================== ============= 
```
returns: `int32_t`


#### d_long

```c
uint64_t d_long(const d_token_t *item);
```

returns the value as long. 

only if type is integer or bytes, but short enough 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: `uint64_t`


#### d_longd

```c
uint64_t d_longd(const d_token_t *item, const uint64_t def_val);
```

returns the value as long or if NULL the default. 

only if type is integer or bytes, but short enough 

arguments:
```eval_rst
================================== ============= 
`d_token_tconst , * <#d-token-t>`_  **item**     
``const uint64_t``                  **def_val**  
================================== ============= 
```
returns: `uint64_t`


#### d_create_bytes_vec

```c
bytes_t** d_create_bytes_vec(const d_token_t *arr);
```

arguments:
```eval_rst
================================== ========= 
`d_token_tconst , * <#d-token-t>`_  **arr**  
================================== ========= 
```
returns: [`bytes_t **`](#bytes-t)


#### d_type

```c
static d_type_t d_type(const d_token_t *item);
```

creates a array of bytes from JOSN-array 

type of the token 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: [`d_type_t`](#d-type-t)


#### d_len

```c
static int d_len(const d_token_t *item);
```

number of elements in the token (only for object or array, other will return 0) 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: `int`


#### d_eq

```c
bool d_eq(const d_token_t *a, const d_token_t *b);
```

compares 2 token and if the value is equal 

arguments:
```eval_rst
================================== ======= 
`d_token_tconst , * <#d-token-t>`_  **a**  
`d_token_tconst , * <#d-token-t>`_  **b**  
================================== ======= 
```
returns: `bool`


#### keyn

```c
d_key_t keyn(const char *c, const size_t len);
```

generates the keyhash for the given stringrange as defined by len 

arguments:
```eval_rst
================ ========= 
``const char *``  **c**    
``const size_t``  **len**  
================ ========= 
```
returns: `d_key_t`


#### d_get

```c
d_token_t* d_get(d_token_t *item, const uint16_t key);
```

returns the token with the given propertyname (only if item is a object) 

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
``const uint16_t``           **key**   
=========================== ========== 
```
returns: [`d_token_t *`](#d-token-t)


#### d_get_or

```c
d_token_t* d_get_or(d_token_t *item, const uint16_t key1, const uint16_t key2);
```

returns the token with the given propertyname or if not found, tries the other. 

(only if item is a object) 

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
``const uint16_t``           **key1**  
``const uint16_t``           **key2**  
=========================== ========== 
```
returns: [`d_token_t *`](#d-token-t)


#### d_get_at

```c
d_token_t* d_get_at(d_token_t *item, const uint32_t index);
```

returns the token of an array with the given index 

arguments:
```eval_rst
=========================== =========== 
`d_token_t * <#d-token-t>`_  **item**   
``const uint32_t``           **index**  
=========================== =========== 
```
returns: [`d_token_t *`](#d-token-t)


#### d_next

```c
d_token_t* d_next(d_token_t *item);
```

returns the next sibling of an array or object 

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
=========================== ========== 
```
returns: [`d_token_t *`](#d-token-t)


#### d_serialize_binary

```c
void d_serialize_binary(bytes_builder_t *bb, d_token_t *t);
```

write the token as binary data into the builder 

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
`d_token_t * <#d-token-t>`_              **t**   
======================================= ======== 
```

#### parse_binary

```c
json_ctx_t* parse_binary(const bytes_t *data);
```

parses the data and returns the context with the token, which needs to be freed after usage! 

arguments:
```eval_rst
============================== ========== 
`bytes_tconst , * <#bytes-t>`_  **data**  
============================== ========== 
```
returns: [`json_ctx_t *`](#json-ctx-t)


#### parse_binary_str

```c
json_ctx_t* parse_binary_str(const char *data, int len);
```

parses the data and returns the context with the token, which needs to be freed after usage! 

arguments:
```eval_rst
================ ========== 
``const char *``  **data**  
``int``           **len**   
================ ========== 
```
returns: [`json_ctx_t *`](#json-ctx-t)


#### parse_json

```c
json_ctx_t* parse_json(char *js);
```

parses json-data, which needs to be freed after usage! 

arguments:
```eval_rst
========== ======== 
``char *``  **js**  
========== ======== 
```
returns: [`json_ctx_t *`](#json-ctx-t)


#### json_free

```c
void json_free(json_ctx_t *parser_ctx);
```

frees the parse-context after usage 

arguments:
```eval_rst
============================= ================ 
`json_ctx_t * <#json-ctx-t>`_  **parser_ctx**  
============================= ================ 
```

#### d_to_json

```c
str_range_t d_to_json(const d_token_t *item);
```

returns the string for a object or array. 

This only works for json as string. For binary it will not work! 

arguments:
```eval_rst
================================== ========== 
`d_token_tconst , * <#d-token-t>`_  **item**  
================================== ========== 
```
returns: [`str_range_t`](#str-range-t)


#### d_create_json

```c
char* d_create_json(d_token_t *item);
```

creates a json-string. 

It does not work for objects if the parsed data were binary! 

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
=========================== ========== 
```
returns: `char *`


#### json_create

```c
json_ctx_t* json_create();
```

returns: [`json_ctx_t *`](#json-ctx-t)


#### json_create_null

```c
d_token_t* json_create_null(json_ctx_t *jp);
```

arguments:
```eval_rst
============================= ======== 
`json_ctx_t * <#json-ctx-t>`_  **jp**  
============================= ======== 
```
returns: [`d_token_t *`](#d-token-t)


#### json_create_bool

```c
d_token_t* json_create_bool(json_ctx_t *jp, bool value);
```

arguments:
```eval_rst
============================= =========== 
`json_ctx_t * <#json-ctx-t>`_  **jp**     
``bool``                       **value**  
============================= =========== 
```
returns: [`d_token_t *`](#d-token-t)


#### json_create_int

```c
d_token_t* json_create_int(json_ctx_t *jp, uint64_t value);
```

arguments:
```eval_rst
============================= =========== 
`json_ctx_t * <#json-ctx-t>`_  **jp**     
``uint64_t``                   **value**  
============================= =========== 
```
returns: [`d_token_t *`](#d-token-t)


#### json_create_string

```c
d_token_t* json_create_string(json_ctx_t *jp, char *value);
```

arguments:
```eval_rst
============================= =========== 
`json_ctx_t * <#json-ctx-t>`_  **jp**     
``char *``                     **value**  
============================= =========== 
```
returns: [`d_token_t *`](#d-token-t)


#### json_create_bytes

```c
d_token_t* json_create_bytes(json_ctx_t *jp, bytes_t value);
```

arguments:
```eval_rst
============================= =========== 
`json_ctx_t * <#json-ctx-t>`_  **jp**     
`bytes_t <#bytes-t>`_          **value**  
============================= =========== 
```
returns: [`d_token_t *`](#d-token-t)


#### json_create_object

```c
d_token_t* json_create_object(json_ctx_t *jp);
```

arguments:
```eval_rst
============================= ======== 
`json_ctx_t * <#json-ctx-t>`_  **jp**  
============================= ======== 
```
returns: [`d_token_t *`](#d-token-t)


#### json_create_array

```c
d_token_t* json_create_array(json_ctx_t *jp);
```

arguments:
```eval_rst
============================= ======== 
`json_ctx_t * <#json-ctx-t>`_  **jp**  
============================= ======== 
```
returns: [`d_token_t *`](#d-token-t)


#### json_object_add_prop

```c
d_token_t* json_object_add_prop(d_token_t *object, d_key_t key, d_token_t *value);
```

arguments:
```eval_rst
=========================== ============ 
`d_token_t * <#d-token-t>`_  **object**  
``d_key_t``                  **key**     
`d_token_t * <#d-token-t>`_  **value**   
=========================== ============ 
```
returns: [`d_token_t *`](#d-token-t)


#### json_array_add_value

```c
d_token_t* json_array_add_value(d_token_t *object, d_token_t *value);
```

arguments:
```eval_rst
=========================== ============ 
`d_token_t * <#d-token-t>`_  **object**  
`d_token_t * <#d-token-t>`_  **value**   
=========================== ============ 
```
returns: [`d_token_t *`](#d-token-t)


#### d_get_keystr

```c
char* d_get_keystr(d_key_t k);
```

returns the string for a key. 

This only works track_keynames was activated before! 

arguments:
```eval_rst
=========== ======= 
``d_key_t``  **k**  
=========== ======= 
```
returns: `char *`


#### d_track_keynames

```c
void d_track_keynames(uint8_t v);
```

activates the keyname-cache, which stores the string for the keys when parsing. 

arguments:
```eval_rst
=========== ======= 
``uint8_t``  **v**  
=========== ======= 
```

#### d_clear_keynames

```c
void d_clear_keynames();
```

delete the cached keynames 


#### key

```c
static d_key_t key(const char *c);
```

arguments:
```eval_rst
================ ======= 
``const char *``  **c**  
================ ======= 
```
returns: `d_key_t`


#### d_get_stringk

```c
static char* d_get_stringk(d_token_t *r, d_key_t k);
```

reads token of a property as string. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``d_key_t``                  **k**  
=========================== ======= 
```
returns: `char *`


#### d_get_string

```c
static char* d_get_string(d_token_t *r, char *k);
```

reads token of a property as string. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``char *``                   **k**  
=========================== ======= 
```
returns: `char *`


#### d_get_string_at

```c
static char* d_get_string_at(d_token_t *r, uint32_t pos);
```

reads string at given pos of an array. 

arguments:
```eval_rst
=========================== ========= 
`d_token_t * <#d-token-t>`_  **r**    
``uint32_t``                 **pos**  
=========================== ========= 
```
returns: `char *`


#### d_get_intk

```c
static int32_t d_get_intk(d_token_t *r, d_key_t k);
```

reads token of a property as int. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``d_key_t``                  **k**  
=========================== ======= 
```
returns: `int32_t`


#### d_get_intkd

```c
static int32_t d_get_intkd(d_token_t *r, d_key_t k, uint32_t d);
```

reads token of a property as int. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``d_key_t``                  **k**  
``uint32_t``                 **d**  
=========================== ======= 
```
returns: `int32_t`


#### d_get_int

```c
static int32_t d_get_int(d_token_t *r, char *k);
```

reads token of a property as int. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``char *``                   **k**  
=========================== ======= 
```
returns: `int32_t`


#### d_get_int_at

```c
static int32_t d_get_int_at(d_token_t *r, uint32_t pos);
```

reads a int at given pos of an array. 

arguments:
```eval_rst
=========================== ========= 
`d_token_t * <#d-token-t>`_  **r**    
``uint32_t``                 **pos**  
=========================== ========= 
```
returns: `int32_t`


#### d_get_longk

```c
static uint64_t d_get_longk(d_token_t *r, d_key_t k);
```

reads token of a property as long. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``d_key_t``                  **k**  
=========================== ======= 
```
returns: `uint64_t`


#### d_get_longkd

```c
static uint64_t d_get_longkd(d_token_t *r, d_key_t k, uint64_t d);
```

reads token of a property as long. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``d_key_t``                  **k**  
``uint64_t``                 **d**  
=========================== ======= 
```
returns: `uint64_t`


#### d_get_long

```c
static uint64_t d_get_long(d_token_t *r, char *k);
```

reads token of a property as long. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``char *``                   **k**  
=========================== ======= 
```
returns: `uint64_t`


#### d_get_long_at

```c
static uint64_t d_get_long_at(d_token_t *r, uint32_t pos);
```

reads long at given pos of an array. 

arguments:
```eval_rst
=========================== ========= 
`d_token_t * <#d-token-t>`_  **r**    
``uint32_t``                 **pos**  
=========================== ========= 
```
returns: `uint64_t`


#### d_get_bytesk

```c
static bytes_t* d_get_bytesk(d_token_t *r, d_key_t k);
```

reads token of a property as bytes. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``d_key_t``                  **k**  
=========================== ======= 
```
returns: [`bytes_t *`](#bytes-t)


#### d_get_bytes

```c
static bytes_t* d_get_bytes(d_token_t *r, char *k);
```

reads token of a property as bytes. 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **r**  
``char *``                   **k**  
=========================== ======= 
```
returns: [`bytes_t *`](#bytes-t)


#### d_get_bytes_at

```c
static bytes_t* d_get_bytes_at(d_token_t *r, uint32_t pos);
```

reads bytes at given pos of an array. 

arguments:
```eval_rst
=========================== ========= 
`d_token_t * <#d-token-t>`_  **r**    
``uint32_t``                 **pos**  
=========================== ========= 
```
returns: [`bytes_t *`](#bytes-t)


#### d_is_binary_ctx

```c
static bool d_is_binary_ctx(json_ctx_t *ctx);
```

check if the parser context was created from binary data. 

arguments:
```eval_rst
============================= ========= 
`json_ctx_t * <#json-ctx-t>`_  **ctx**  
============================= ========= 
```
returns: `bool`


#### d_get_byteskl

```c
bytes_t* d_get_byteskl(d_token_t *r, d_key_t k, uint32_t minl);
```

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **r**     
``d_key_t``                  **k**     
``uint32_t``                 **minl**  
=========================== ========== 
```
returns: [`bytes_t *`](#bytes-t)


#### d_getl

```c
d_token_t* d_getl(d_token_t *item, uint16_t k, uint32_t minl);
```

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **item**  
``uint16_t``                 **k**     
``uint32_t``                 **minl**  
=========================== ========== 
```
returns: [`d_token_t *`](#d-token-t)


#### d_iter

```c
static d_iterator_t d_iter(d_token_t *parent);
```

creates a iterator for a object or array 

arguments:
```eval_rst
=========================== ============ 
`d_token_t * <#d-token-t>`_  **parent**  
=========================== ============ 
```
returns: [`d_iterator_t`](#d-iterator-t)


#### d_iter_next

```c
static bool d_iter_next(d_iterator_t *const iter);
```

fetched the next token an returns a boolean indicating whther there is a next or not. 

arguments:
```eval_rst
====================================== ========== 
`d_iterator_t *const <#d-iterator-t>`_  **iter**  
====================================== ========== 
```
returns: `bool`


### debug.h

logs debug data only if the DEBUG-flag is set. 

File: [src/core/util/debug.h](https://github.com/slockit/in3-c/blob/master/src/core/util/debug.h)

#### dbg_log (msg,...)

logs a debug-message including file and linenumber 


#### dbg_log_raw (msg,...)

logs a debug-message without the filename 


#### msg_dump

```c
void msg_dump(const char *s, const unsigned char *data, unsigned len);
```

dumps the given data as hex coded bytes to stdout 

arguments:
```eval_rst
========================= ========== 
``const char *``           **s**     
``const unsigned char *``  **data**  
``unsigned``               **len**   
========================= ========== 
```

### error.h

defines the return-values of a function call. 

File: [src/core/util/error.h](https://github.com/slockit/in3-c/blob/master/src/core/util/error.h)

#### DEPRECATED

depreacted-attribute 

```c
#define DEPRECATED __attribute__((deprecated))
```


#### OPTIONAL_T (t)

Optional type similar to C++ std::optional Optional types must be defined prior to usage (e.g. 

DEFINE_OPTIONAL_T(int)) Use OPTIONAL_T_UNDEFINED(t) & OPTIONAL_T_VALUE(t, v) for easy initialization (rvalues) Note: Defining optional types for pointers is ill-formed by definition. This is because redundant 

```c
#define OPTIONAL_T (t) opt_##t
```


#### DEFINE_OPTIONAL_T (t)

Optional types must be defined prior to usage (e.g. 

DEFINE_OPTIONAL_T(int)) Use OPTIONAL_T_UNDEFINED(t) & OPTIONAL_T_VALUE(t, v) for easy initialization (rvalues) 

```c
#define DEFINE_OPTIONAL_T (t) typedef struct {           \
    t    value;              \
    bool defined;            \
  } OPTIONAL_T(t)
```


#### OPTIONAL_T_UNDEFINED (t)

marks a used value as undefined. 

```c
#define OPTIONAL_T_UNDEFINED (t) ((OPTIONAL_T(t)){.defined = false})
```


#### OPTIONAL_T_VALUE (t,v)

sets the value of an optional type. 

```c
#define OPTIONAL_T_VALUE (t,v) ((OPTIONAL_T(t)){.value = v, .defined = true})
```


#### in3_errmsg

```c
char* in3_errmsg(in3_ret_t err);
```

converts a error code into a string. 

These strings are constants and do not need to be freed. 

arguments:
```eval_rst
========================= ========= ==============
`in3_ret_t <#in3-ret-t>`_  **err**  the error code
========================= ========= ==============
```
returns: `char *`


### scache.h

util helper on byte arrays. 

File: [src/core/util/scache.h](https://github.com/slockit/in3-c/blob/master/src/core/util/scache.h)

#### cache_entry_t

represents a single cache entry in a linked list. 

These are used within a request context to cache values and automaticly free them. 


The stuct contains following fields:

```eval_rst
======================================= =============== ==============================================================================
`bytes_t <#bytes-t>`_                    **key**        an optional key of the entry
`bytes_t <#bytes-t>`_                    **value**      the value
``uint8_t``                              **buffer**     the buffer is used to store extra data, which will be cleaned when freed.
``bool``                                 **must_free**  if true, the cache-entry will be freed when the request context is cleaned up.
`cache_entrystruct , * <#cache-entry>`_  **next**       pointer to the next entry.
======================================= =============== ==============================================================================
```

#### in3_cache_get_entry

```c
bytes_t* in3_cache_get_entry(cache_entry_t *cache, bytes_t *key);
```

get the entry for a given key. 

arguments:
```eval_rst
=================================== =========== ==================================
`cache_entry_t * <#cache-entry-t>`_  **cache**  the root entry of the linked list.
`bytes_t * <#bytes-t>`_              **key**    the key to compare with
=================================== =========== ==================================
```
returns: [`bytes_t *`](#bytes-t)


#### in3_cache_add_entry

```c
cache_entry_t* in3_cache_add_entry(cache_entry_t **cache, bytes_t key, bytes_t value);
```

adds an entry to the linked list. 

arguments:
```eval_rst
==================================== =========== ==================================
`cache_entry_t ** <#cache-entry-t>`_  **cache**  the root entry of the linked list.
`bytes_t <#bytes-t>`_                 **key**    an optional key
`bytes_t <#bytes-t>`_                 **value**  the value of the entry
==================================== =========== ==================================
```
returns: [`cache_entry_t *`](#cache-entry-t)


#### in3_cache_free

```c
void in3_cache_free(cache_entry_t *cache);
```

clears all entries in the linked list. 

arguments:
```eval_rst
=================================== =========== ==================================
`cache_entry_t * <#cache-entry-t>`_  **cache**  the root entry of the linked list.
=================================== =========== ==================================
```

#### in3_cache_add_ptr

```c
static cache_entry_t* in3_cache_add_ptr(cache_entry_t **cache, void *ptr);
```

adds a pointer, which should be freed when the context is freed. 

arguments:
```eval_rst
==================================== =========== =======================================
`cache_entry_t ** <#cache-entry-t>`_  **cache**  the root entry of the linked list.
``void *``                            **ptr**    pointer to memory which shold be freed.
==================================== =========== =======================================
```
returns: [`cache_entry_t *`](#cache-entry-t)


### stringbuilder.h

simple string buffer used to dynamicly add content. 

File: [src/core/util/stringbuilder.h](https://github.com/slockit/in3-c/blob/master/src/core/util/stringbuilder.h)

#### sb_add_hexuint (sb,i)

shortcut macro for adding a uint to the stringbuilder using sizeof(i) to automaticly determine the size 

```c
#define sb_add_hexuint (sb,i) sb_add_hexuint_l(sb, i, sizeof(i))
```


#### sb_t

string build struct, which is able to hold and modify a growing string. 


The stuct contains following fields:

```eval_rst
========== ============== ====================================
``char *``  **data**      the current string (null terminated)
``size_t``  **allocted**  number of bytes currently allocated
``size_t``  **len**       the current length of the string
========== ============== ====================================
```

#### sb_new

```c
sb_t* sb_new(const char *chars);
```

creates a new stringbuilder and copies the inital characters into it. 

arguments:
```eval_rst
================ =========== 
``const char *``  **chars**  
================ =========== 
```
returns: [`sb_t *`](#sb-t)


#### sb_init

```c
sb_t* sb_init(sb_t *sb);
```

initializes a stringbuilder by allocating memory. 

arguments:
```eval_rst
================= ======== 
`sb_t * <#sb-t>`_  **sb**  
================= ======== 
```
returns: [`sb_t *`](#sb-t)


#### sb_free

```c
void sb_free(sb_t *sb);
```

frees all resources of the stringbuilder 

arguments:
```eval_rst
================= ======== 
`sb_t * <#sb-t>`_  **sb**  
================= ======== 
```

#### sb_add_char

```c
sb_t* sb_add_char(sb_t *sb, char c);
```

add a single character 

arguments:
```eval_rst
================= ======== 
`sb_t * <#sb-t>`_  **sb**  
``char``           **c**   
================= ======== 
```
returns: [`sb_t *`](#sb-t)


#### sb_add_chars

```c
sb_t* sb_add_chars(sb_t *sb, const char *chars);
```

adds a string 

arguments:
```eval_rst
================= =========== 
`sb_t * <#sb-t>`_  **sb**     
``const char *``   **chars**  
================= =========== 
```
returns: [`sb_t *`](#sb-t)


#### sb_add_range

```c
sb_t* sb_add_range(sb_t *sb, const char *chars, int start, int len);
```

add a string range 

arguments:
```eval_rst
================= =========== 
`sb_t * <#sb-t>`_  **sb**     
``const char *``   **chars**  
``int``            **start**  
``int``            **len**    
================= =========== 
```
returns: [`sb_t *`](#sb-t)


#### sb_add_key_value

```c
sb_t* sb_add_key_value(sb_t *sb, const char *key, const char *value, int value_len, bool as_string);
```

adds a value with an optional key. 

if as_string is true the value will be quoted. 

arguments:
```eval_rst
================= =============== 
`sb_t * <#sb-t>`_  **sb**         
``const char *``   **key**        
``const char *``   **value**      
``int``            **value_len**  
``bool``           **as_string**  
================= =============== 
```
returns: [`sb_t *`](#sb-t)


#### sb_add_bytes

```c
sb_t* sb_add_bytes(sb_t *sb, const char *prefix, const bytes_t *bytes, int len, bool as_array);
```

add bytes as 0x-prefixed hexcoded string (including an optional prefix), if len>1 is passed bytes maybe an array ( if as_array==true) 


 

arguments:
```eval_rst
============================== ============== 
`sb_t * <#sb-t>`_               **sb**        
``const char *``                **prefix**    
`bytes_tconst , * <#bytes-t>`_  **bytes**     
``int``                         **len**       
``bool``                        **as_array**  
============================== ============== 
```
returns: [`sb_t *`](#sb-t)


#### sb_add_hexuint_l

```c
sb_t* sb_add_hexuint_l(sb_t *sb, uintmax_t uint, size_t l);
```

add a integer value as hexcoded, 0x-prefixed string 

Other types not supported

arguments:
```eval_rst
================= ========== 
`sb_t * <#sb-t>`_  **sb**    
``uintmax_t``      **uint**  
``size_t``         **l**     
================= ========== 
```
returns: [`sb_t *`](#sb-t)


### utils.h

utility functions. 

File: [src/core/util/utils.h](https://github.com/slockit/in3-c/blob/master/src/core/util/utils.h)

#### SWAP (a,b)

simple swap macro for integral types 

```c
#define SWAP (a,b) {                \
    void* p = a;   \
    a       = b;   \
    b       = p;   \
  }
```


#### min (a,b)

simple min macro for interagl types 

```c
#define min (a,b) ((a) < (b) ? (a) : (b))
```


#### max (a,b)

simple max macro for interagl types 

```c
#define max (a,b) ((a) > (b) ? (a) : (b))
```


#### IS_APPROX (n1,n2,err)

Check if n1 & n2 are at max err apart Expects n1 & n2 to be integral types. 

```c
#define IS_APPROX (n1,n2,err) ((n1 > n2) ? ((n1 - n2) <= err) : ((n2 - n1) <= err))
```


#### optimize_len (a,l)

changes to pointer (a) and it length (l) to remove leading 0 bytes. 

```c
#define optimize_len (a,l) while (l > 1 && *a == 0) { \
    l--;                     \
    a++;                     \
  }
```


#### TRY (exp)

executes the expression and expects the return value to be a int indicating the error. 

if the return value is negative it will stop and return this value otherwise continue. 

```c
#define TRY (exp) {                        \
    int _r = (exp);        \
    if (_r < 0) return _r; \
  }
```


#### TRY_SET (var,exp)

executes the expression and expects the return value to be a int indicating the error. 

the return value will be set to a existing variable (var). if the return value is negative it will stop and return this value otherwise continue. 

```c
#define TRY_SET (var,exp) {                          \
    var = (exp);             \
    if (var < 0) return var; \
  }
```


#### TRY_GOTO (exp)

executes the expression and expects the return value to be a int indicating the error. 

if the return value is negative it will stop and jump (goto) to a marked position "clean". it also expects a previously declared variable "in3_ret_t res". 

```c
#define TRY_GOTO (exp) {                          \
    res = (exp);             \
    if (res < 0) goto clean; \
  }
```


#### bytes_to_long

```c
uint64_t bytes_to_long(const uint8_t *data, int len);
```

converts the bytes to a unsigned long (at least the last max len bytes) 

arguments:
```eval_rst
=================== ========== 
``const uint8_t *``  **data**  
``int``              **len**   
=================== ========== 
```
returns: `uint64_t`


#### bytes_to_int

```c
static uint32_t bytes_to_int(const uint8_t *data, int len);
```

converts the bytes to a unsigned int (at least the last max len bytes) 

arguments:
```eval_rst
=================== ========== 
``const uint8_t *``  **data**  
``int``              **len**   
=================== ========== 
```
returns: `uint32_t`


#### char_to_long

```c
uint64_t char_to_long(const char *a, int l);
```

converts a character into a uint64_t 

arguments:
```eval_rst
================ ======= 
``const char *``  **a**  
``int``           **l**  
================ ======= 
```
returns: `uint64_t`


#### hexchar_to_int

```c
uint8_t hexchar_to_int(char c);
```

converts a hexchar to byte (4bit) 

arguments:
```eval_rst
======== ======= 
``char``  **c**  
======== ======= 
```
returns: `uint8_t`


#### u64_to_str

```c
const unsigned char* u64_to_str(uint64_t value, char *pBuf, int szBuf);
```

converts a uint64_t to string (char*); buffer-size min. 

21 bytes 

arguments:
```eval_rst
============ =========== 
``uint64_t``  **value**  
``char *``    **pBuf**   
``int``       **szBuf**  
============ =========== 
```
returns: `const unsigned char *`


#### hex_to_bytes

```c
int hex_to_bytes(const char *hexdata, int hexlen, uint8_t *out, int outlen);
```

convert a c hex string to a byte array storing it into an existing buffer. 

arguments:
```eval_rst
================ ============= 
``const char *``  **hexdata**  
``int``           **hexlen**   
``uint8_t *``     **out**      
``int``           **outlen**   
================ ============= 
```
returns: `int`


#### hex_to_new_bytes

```c
bytes_t* hex_to_new_bytes(const char *buf, int len);
```

convert a c string to a byte array creating a new buffer 

arguments:
```eval_rst
================ ========= 
``const char *``  **buf**  
``int``           **len**  
================ ========= 
```
returns: [`bytes_t *`](#bytes-t)


#### bytes_to_hex

```c
int bytes_to_hex(const uint8_t *buffer, int len, char *out);
```

convefrts a bytes into hex 

arguments:
```eval_rst
=================== ============ 
``const uint8_t *``  **buffer**  
``int``              **len**     
``char *``           **out**     
=================== ============ 
```
returns: `int`


#### sha3

```c
bytes_t* sha3(const bytes_t *data);
```

hashes the bytes and creates a new bytes_t 

arguments:
```eval_rst
============================== ========== 
`bytes_tconst , * <#bytes-t>`_  **data**  
============================== ========== 
```
returns: [`bytes_t *`](#bytes-t)


#### sha3_to

```c
int sha3_to(bytes_t *data, void *dst);
```

writes 32 bytes to the pointer. 

arguments:
```eval_rst
======================= ========== 
`bytes_t * <#bytes-t>`_  **data**  
``void *``               **dst**   
======================= ========== 
```
returns: `int`


#### long_to_bytes

```c
void long_to_bytes(uint64_t val, uint8_t *dst);
```

converts a long to 8 bytes 

arguments:
```eval_rst
============= ========= 
``uint64_t``   **val**  
``uint8_t *``  **dst**  
============= ========= 
```

#### int_to_bytes

```c
void int_to_bytes(uint32_t val, uint8_t *dst);
```

converts a int to 4 bytes 

arguments:
```eval_rst
============= ========= 
``uint32_t``   **val**  
``uint8_t *``  **dst**  
============= ========= 
```

#### _strdupn

```c
char* _strdupn(const char *src, int len);
```

duplicate the string 

arguments:
```eval_rst
================ ========= 
``const char *``  **src**  
``int``           **len**  
================ ========= 
```
returns: `char *`


#### min_bytes_len

```c
int min_bytes_len(uint64_t val);
```

calculate the min number of byte to represents the len 

arguments:
```eval_rst
============ ========= 
``uint64_t``  **val**  
============ ========= 
```
returns: `int`


#### uint256_set

```c
void uint256_set(const uint8_t *src, wlen_t src_len, bytes32_t dst);
```

sets a variable value to 32byte word. 

arguments:
```eval_rst
========================= ============= 
``const uint8_t *``        **src**      
`wlen_t <#wlen-t>`_        **src_len**  
`bytes32_t <#bytes32-t>`_  **dst**      
========================= ============= 
```

#### str_replace

```c
char* str_replace(const char *orig, const char *rep, const char *with);
```

replaces a string and returns a copy. 

arguments:
```eval_rst
================ ========== 
``const char *``  **orig**  
``const char *``  **rep**   
``const char *``  **with**  
================ ========== 
```
returns: `char *`


#### str_replace_pos

```c
char* str_replace_pos(const char *orig, size_t pos, size_t len, const char *rep);
```

replaces a string at the given position. 

arguments:
```eval_rst
================ ========== 
``const char *``  **orig**  
``size_t``        **pos**   
``size_t``        **len**   
``const char *``  **rep**   
================ ========== 
```
returns: `char *`


#### str_find

```c
char* str_find(const char *haystack, const char *needle);
```

lightweight strstr() replacements 

arguments:
```eval_rst
================ ============== 
``const char *``  **haystack**  
``const char *``  **needle**    
================ ============== 
```
returns: `char *`


#### memiszero

```c
static bool memiszero(uint8_t *ptr, size_t l);
```

arguments:
```eval_rst
============= ========= 
``uint8_t *``  **ptr**  
``size_t``     **l**    
============= ========= 
```
returns: `bool`


## Module transport/curl 




### in3_curl.h

transport-handler using libcurl. 

File: [src/transport/curl/in3_curl.h](https://github.com/slockit/in3-c/blob/master/src/transport/curl/in3_curl.h)

#### send_curl

```c
in3_ret_t send_curl(in3_request_t *req);
```

a transport function using curl. 

You can use it by setting the transport-function-pointer in the in3_t->transport to this function:

```c
#include <in3/in3_curl.h>
...
c->transport = send_curl;
```
arguments:
```eval_rst
=================================== ========= 
`in3_request_t * <#in3-request-t>`_  **req**  
=================================== ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_register_curl

```c
void in3_register_curl();
```

registers curl as a default transport. 


## Module transport/http 




### in3_http.h

transport-handler using simple http. 

File: [src/transport/http/in3_http.h](https://github.com/slockit/in3-c/blob/master/src/transport/http/in3_http.h)

#### send_http

```c
in3_ret_t send_http(in3_request_t *req);
```

a very simple transport function, which allows to send http-requests without a dependency to curl. 

Here each request will be transformed to http instead of https.

You can use it by setting the transport-function-pointer in the in3_t->transport to this function:

```c
#include <in3/in3_http.h>
...
c->transport = send_http;
```
arguments:
```eval_rst
=================================== ========= 
`in3_request_t * <#in3-request-t>`_  **req**  
=================================== ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


## Module verifier/eth1/basic 




### eth_basic.h

Ethereum Nanon verification. 

File: [src/verifier/eth1/basic/eth_basic.h](https://github.com/slockit/in3-c/blob/master/src/verifier/eth1/basic/eth_basic.h)

#### in3_verify_eth_basic

```c
in3_ret_t in3_verify_eth_basic(in3_vctx_t *v);
```

entry-function to execute the verification context. 

arguments:
```eval_rst
============================= ======= 
`in3_vctx_t * <#in3-vctx-t>`_  **v**  
============================= ======= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_tx_values

```c
in3_ret_t eth_verify_tx_values(in3_vctx_t *vc, d_token_t *tx, bytes_t *raw);
```

verifies internal tx-values. 

arguments:
```eval_rst
============================= ========= 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**   
`d_token_t * <#d-token-t>`_    **tx**   
`bytes_t * <#bytes-t>`_        **raw**  
============================= ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_eth_getTransaction

```c
in3_ret_t eth_verify_eth_getTransaction(in3_vctx_t *vc, bytes_t *tx_hash);
```

verifies a transaction. 

arguments:
```eval_rst
============================= ============= 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**       
`bytes_t * <#bytes-t>`_        **tx_hash**  
============================= ============= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_eth_getTransactionByBlock

```c
in3_ret_t eth_verify_eth_getTransactionByBlock(in3_vctx_t *vc, d_token_t *blk, uint32_t tx_idx);
```

verifies a transaction by block hash/number and id. 

arguments:
```eval_rst
============================= ============ 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**      
`d_token_t * <#d-token-t>`_    **blk**     
``uint32_t``                   **tx_idx**  
============================= ============ 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_account_proof

```c
in3_ret_t eth_verify_account_proof(in3_vctx_t *vc);
```

verify account-proofs 

arguments:
```eval_rst
============================= ======== 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**  
============================= ======== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_eth_getBlock

```c
in3_ret_t eth_verify_eth_getBlock(in3_vctx_t *vc, bytes_t *block_hash, uint64_t blockNumber);
```

verifies a block 

arguments:
```eval_rst
============================= ================= 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**           
`bytes_t * <#bytes-t>`_        **block_hash**   
``uint64_t``                   **blockNumber**  
============================= ================= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_register_eth_basic

```c
void in3_register_eth_basic();
```

this function should only be called once and will register the eth-nano verifier. 


#### eth_verify_eth_getLog

```c
in3_ret_t eth_verify_eth_getLog(in3_vctx_t *vc, int l_logs);
```

verify logs 

arguments:
```eval_rst
============================= ============ 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**      
``int``                        **l_logs**  
============================= ============ 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_handle_intern

```c
in3_ret_t eth_handle_intern(in3_ctx_t *ctx, in3_response_t **response);
```

this is called before a request is send 

arguments:
```eval_rst
====================================== ============== 
`in3_ctx_t * <#in3-ctx-t>`_             **ctx**       
`in3_response_t ** <#in3-response-t>`_  **response**  
====================================== ============== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### signer.h

Ethereum Nano verification. 

File: [src/verifier/eth1/basic/signer.h](https://github.com/slockit/in3-c/blob/master/src/verifier/eth1/basic/signer.h)

#### eth_set_pk_signer

```c
in3_ret_t eth_set_pk_signer(in3_t *in3, bytes32_t pk);
```

simply signer with one private key. 

since the pk pointting to the 32 byte private key is not cloned, please make sure, you manage memory allocation correctly!

simply signer with one private key. 

arguments:
```eval_rst
========================= ========= 
`in3_t * <#in3-t>`_        **in3**  
`bytes32_t <#bytes32-t>`_  **pk**   
========================= ========= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### trie.h

Patricia Merkle Tree Imnpl 

File: [src/verifier/eth1/basic/trie.h](https://github.com/slockit/in3-c/blob/master/src/verifier/eth1/basic/trie.h)

#### in3_hasher_t

hash-function 


```c
typedef void(* in3_hasher_t) (bytes_t *src, uint8_t *dst)
```


#### in3_codec_add_t

codec to organize the encoding of the nodes 


```c
typedef void(* in3_codec_add_t) (bytes_builder_t *bb, bytes_t *val)
```


#### in3_codec_finish_t


```c
typedef void(* in3_codec_finish_t) (bytes_builder_t *bb, bytes_t *dst)
```


#### in3_codec_decode_size_t


```c
typedef int(* in3_codec_decode_size_t) (bytes_t *src)
```

returns: `int(*`


#### in3_codec_decode_index_t


```c
typedef int(* in3_codec_decode_index_t) (bytes_t *src, int index, bytes_t *dst)
```

returns: `int(*`


#### trie_node_t

single node in the merkle trie. 


The stuct contains following fields:

```eval_rst
======================================= ================ ===============================================
``uint8_t``                              **hash**        the hash of the node
`bytes_t <#bytes-t>`_                    **data**        the raw data
`bytes_t <#bytes-t>`_                    **items**       the data as list
``uint8_t``                              **own_memory**  if true this is a embedded node with own memory
`trie_node_type_t <#trie-node-type-t>`_  **type**        type of the node
`trie_nodestruct , * <#trie-node>`_      **next**        used as linked list
======================================= ================ ===============================================
```

#### trie_codec_t

the codec used to encode nodes. 


The stuct contains following fields:

```eval_rst
===================================== =================== 
`in3_codec_add_t <#in3-codec-add-t>`_  **encode_add**     
``in3_codec_finish_t``                 **encode_finish**  
``in3_codec_decode_size_t``            **decode_size**    
``in3_codec_decode_index_t``           **decode_item**    
===================================== =================== 
```

#### trie_t

a merkle trie implementation. 

This is a Patricia Merkle Tree. 


The stuct contains following fields:

```eval_rst
================================= ============ ==============================
`in3_hasher_t <#in3-hasher-t>`_    **hasher**  hash-function.
`trie_codec_t * <#trie-codec-t>`_  **codec**   encoding of the nocds.
`bytes32_t <#bytes32-t>`_          **root**    The root-hash.
`trie_node_t * <#trie-node-t>`_    **nodes**   linked list of containes nodes
================================= ============ ==============================
```

#### trie_new

```c
trie_t* trie_new();
```

creates a new Merkle Trie. 

returns: [`trie_t *`](#trie-t)


#### trie_free

```c
void trie_free(trie_t *val);
```

frees all resources of the trie. 

arguments:
```eval_rst
===================== ========= 
`trie_t * <#trie-t>`_  **val**  
===================== ========= 
```

#### trie_set_value

```c
void trie_set_value(trie_t *t, bytes_t *key, bytes_t *value);
```

sets a value in the trie. 

The root-hash will be updated automaticly. 

arguments:
```eval_rst
======================= =========== 
`trie_t * <#trie-t>`_    **t**      
`bytes_t * <#bytes-t>`_  **key**    
`bytes_t * <#bytes-t>`_  **value**  
======================= =========== 
```

## Module verifier/eth1/evm 




### big.h

Ethereum Nanon verification. 

File: [src/verifier/eth1/evm/big.h](https://github.com/slockit/in3-c/blob/master/src/verifier/eth1/evm/big.h)

#### big_is_zero

```c
uint8_t big_is_zero(uint8_t *data, wlen_t l);
```

arguments:
```eval_rst
=================== ========== 
``uint8_t *``        **data**  
`wlen_t <#wlen-t>`_  **l**     
=================== ========== 
```
returns: `uint8_t`


#### big_shift_left

```c
void big_shift_left(uint8_t *a, wlen_t len, int bits);
```

arguments:
```eval_rst
=================== ========== 
``uint8_t *``        **a**     
`wlen_t <#wlen-t>`_  **len**   
``int``              **bits**  
=================== ========== 
```

#### big_shift_right

```c
void big_shift_right(uint8_t *a, wlen_t len, int bits);
```

arguments:
```eval_rst
=================== ========== 
``uint8_t *``        **a**     
`wlen_t <#wlen-t>`_  **len**   
``int``              **bits**  
=================== ========== 
```

#### big_cmp

```c
int big_cmp(const uint8_t *a, const wlen_t len_a, const uint8_t *b, const wlen_t len_b);
```

arguments:
```eval_rst
========================= =========== 
``const uint8_t *``        **a**      
`wlen_tconst  <#wlen-t>`_  **len_a**  
``const uint8_t *``        **b**      
`wlen_tconst  <#wlen-t>`_  **len_b**  
========================= =========== 
```
returns: `int`


#### big_signed

```c
int big_signed(uint8_t *val, wlen_t len, uint8_t *dst);
```

returns 0 if the value is positive or 1 if negavtive. 

in this case the absolute value is copied to dst. 

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **val**  
`wlen_t <#wlen-t>`_  **len**  
``uint8_t *``        **dst**  
=================== ========= 
```
returns: `int`


#### big_int

```c
int32_t big_int(uint8_t *val, wlen_t len);
```

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **val**  
`wlen_t <#wlen-t>`_  **len**  
=================== ========= 
```
returns: `int32_t`


#### big_add

```c
int big_add(uint8_t *a, wlen_t len_a, uint8_t *b, wlen_t len_b, uint8_t *out, wlen_t max);
```

arguments:
```eval_rst
=================== =========== 
``uint8_t *``        **a**      
`wlen_t <#wlen-t>`_  **len_a**  
``uint8_t *``        **b**      
`wlen_t <#wlen-t>`_  **len_b**  
``uint8_t *``        **out**    
`wlen_t <#wlen-t>`_  **max**    
=================== =========== 
```
returns: `int`


#### big_sub

```c
int big_sub(uint8_t *a, wlen_t len_a, uint8_t *b, wlen_t len_b, uint8_t *out);
```

arguments:
```eval_rst
=================== =========== 
``uint8_t *``        **a**      
`wlen_t <#wlen-t>`_  **len_a**  
``uint8_t *``        **b**      
`wlen_t <#wlen-t>`_  **len_b**  
``uint8_t *``        **out**    
=================== =========== 
```
returns: `int`


#### big_mul

```c
int big_mul(uint8_t *a, wlen_t la, uint8_t *b, wlen_t lb, uint8_t *res, wlen_t max);
```

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **a**    
`wlen_t <#wlen-t>`_  **la**   
``uint8_t *``        **b**    
`wlen_t <#wlen-t>`_  **lb**   
``uint8_t *``        **res**  
`wlen_t <#wlen-t>`_  **max**  
=================== ========= 
```
returns: `int`


#### big_div

```c
int big_div(uint8_t *a, wlen_t la, uint8_t *b, wlen_t lb, wlen_t sig, uint8_t *res);
```

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **a**    
`wlen_t <#wlen-t>`_  **la**   
``uint8_t *``        **b**    
`wlen_t <#wlen-t>`_  **lb**   
`wlen_t <#wlen-t>`_  **sig**  
``uint8_t *``        **res**  
=================== ========= 
```
returns: `int`


#### big_mod

```c
int big_mod(uint8_t *a, wlen_t la, uint8_t *b, wlen_t lb, wlen_t sig, uint8_t *res);
```

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **a**    
`wlen_t <#wlen-t>`_  **la**   
``uint8_t *``        **b**    
`wlen_t <#wlen-t>`_  **lb**   
`wlen_t <#wlen-t>`_  **sig**  
``uint8_t *``        **res**  
=================== ========= 
```
returns: `int`


#### big_exp

```c
int big_exp(uint8_t *a, wlen_t la, uint8_t *b, wlen_t lb, uint8_t *res);
```

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **a**    
`wlen_t <#wlen-t>`_  **la**   
``uint8_t *``        **b**    
`wlen_t <#wlen-t>`_  **lb**   
``uint8_t *``        **res**  
=================== ========= 
```
returns: `int`


#### big_log256

```c
int big_log256(uint8_t *a, wlen_t len);
```

arguments:
```eval_rst
=================== ========= 
``uint8_t *``        **a**    
`wlen_t <#wlen-t>`_  **len**  
=================== ========= 
```
returns: `int`


### code.h

code cache. 

File: [src/verifier/eth1/evm/code.h](https://github.com/slockit/in3-c/blob/master/src/verifier/eth1/evm/code.h)

#### in3_get_code

```c
in3_ret_t in3_get_code(in3_vctx_t *vc, address_t address, cache_entry_t **target);
```

fetches the code and adds it to the context-cache as cache_entry. 

So calling this function a second time will take the result from cache. 

arguments:
```eval_rst
==================================== ============= 
`in3_vctx_t * <#in3-vctx-t>`_         **vc**       
`address_t <#address-t>`_             **address**  
`cache_entry_t ** <#cache-entry-t>`_  **target**   
==================================== ============= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


### evm.h

main evm-file. 

File: [src/verifier/eth1/evm/evm.h](https://github.com/slockit/in3-c/blob/master/src/verifier/eth1/evm/evm.h)

#### gas_options


#### EVM_ERROR_EMPTY_STACK

the no more elements on the stack 


 

```c
#define EVM_ERROR_EMPTY_STACK -20
```


#### EVM_ERROR_INVALID_OPCODE

the opcode is not supported 


 

```c
#define EVM_ERROR_INVALID_OPCODE -21
```


#### EVM_ERROR_BUFFER_TOO_SMALL

reading data from a position, which is not initialized 


 

```c
#define EVM_ERROR_BUFFER_TOO_SMALL -22
```


#### EVM_ERROR_ILLEGAL_MEMORY_ACCESS

the memory-offset does not exist 


 

```c
#define EVM_ERROR_ILLEGAL_MEMORY_ACCESS -23
```


#### EVM_ERROR_INVALID_JUMPDEST

the jump destination is not marked as valid destination 


 

```c
#define EVM_ERROR_INVALID_JUMPDEST -24
```


#### EVM_ERROR_INVALID_PUSH

the push data is empy 

```c
#define EVM_ERROR_INVALID_PUSH -25
```


#### EVM_ERROR_UNSUPPORTED_CALL_OPCODE

error handling the call, usually because static-calls are not allowed to change state 


 

```c
#define EVM_ERROR_UNSUPPORTED_CALL_OPCODE -26
```


#### EVM_ERROR_TIMEOUT

the evm ran into a loop 


 

```c
#define EVM_ERROR_TIMEOUT -27
```


#### EVM_ERROR_INVALID_ENV

the enviroment could not deliver the data 


 

```c
#define EVM_ERROR_INVALID_ENV -28
```


#### EVM_ERROR_OUT_OF_GAS

not enough gas to exewcute the opcode 


 

```c
#define EVM_ERROR_OUT_OF_GAS -29
```


#### EVM_ERROR_BALANCE_TOO_LOW

not enough funds to transfer the requested value. 

```c
#define EVM_ERROR_BALANCE_TOO_LOW -30
```


#### EVM_ERROR_STACK_LIMIT

stack limit reached 


 

```c
#define EVM_ERROR_STACK_LIMIT -31
```


#### EVM_ERROR_SUCCESS_CONSUME_GAS

write success but consume all gas 

```c
#define EVM_ERROR_SUCCESS_CONSUME_GAS -32
```


#### EVM_PROP_FRONTIER

```c
#define EVM_PROP_FRONTIER 1
```


#### EVM_PROP_EIP150

```c
#define EVM_PROP_EIP150 2
```


#### EVM_PROP_EIP158

```c
#define EVM_PROP_EIP158 4
```


#### EVM_PROP_CONSTANTINOPL

```c
#define EVM_PROP_CONSTANTINOPL 16
```


#### EVM_PROP_ISTANBUL

```c
#define EVM_PROP_ISTANBUL 32
```


#### EVM_PROP_NO_FINALIZE

```c
#define EVM_PROP_NO_FINALIZE 32768
```


#### EVM_PROP_STATIC

```c
#define EVM_PROP_STATIC 256
```


#### EVM_ENV_BALANCE

```c
#define EVM_ENV_BALANCE 1
```


#### EVM_ENV_CODE_SIZE

```c
#define EVM_ENV_CODE_SIZE 2
```


#### EVM_ENV_CODE_COPY

```c
#define EVM_ENV_CODE_COPY 3
```


#### EVM_ENV_BLOCKHASH

```c
#define EVM_ENV_BLOCKHASH 4
```


#### EVM_ENV_STORAGE

```c
#define EVM_ENV_STORAGE 5
```


#### EVM_ENV_BLOCKHEADER

```c
#define EVM_ENV_BLOCKHEADER 6
```


#### EVM_ENV_CODE_HASH

```c
#define EVM_ENV_CODE_HASH 7
```


#### EVM_ENV_NONCE

```c
#define EVM_ENV_NONCE 8
```


#### MATH_ADD

```c
#define MATH_ADD 1
```


#### MATH_SUB

```c
#define MATH_SUB 2
```


#### MATH_MUL

```c
#define MATH_MUL 3
```


#### MATH_DIV

```c
#define MATH_DIV 4
```


#### MATH_SDIV

```c
#define MATH_SDIV 5
```


#### MATH_MOD

```c
#define MATH_MOD 6
```


#### MATH_SMOD

```c
#define MATH_SMOD 7
```


#### MATH_EXP

```c
#define MATH_EXP 8
```


#### MATH_SIGNEXP

```c
#define MATH_SIGNEXP 9
```


#### CALL_CALL

```c
#define CALL_CALL 0
```


#### CALL_CODE

```c
#define CALL_CODE 1
```


#### CALL_DELEGATE

```c
#define CALL_DELEGATE 2
```


#### CALL_STATIC

```c
#define CALL_STATIC 3
```


#### OP_AND

```c
#define OP_AND 0
```


#### OP_OR

```c
#define OP_OR 1
```


#### OP_XOR

```c
#define OP_XOR 2
```


#### EVM_DEBUG_BLOCK (...)


#### OP_LOG (...)

```c
#define OP_LOG (...) EVM_ERROR_UNSUPPORTED_CALL_OPCODE
```


#### OP_SLOAD_GAS (...)


#### OP_CREATE (...)

```c
#define OP_CREATE (...) EVM_ERROR_UNSUPPORTED_CALL_OPCODE
```


#### OP_ACCOUNT_GAS (...)

```c
#define OP_ACCOUNT_GAS (...) 0
```


#### OP_SELFDESTRUCT (...)

```c
#define OP_SELFDESTRUCT (...) EVM_ERROR_UNSUPPORTED_CALL_OPCODE
```


#### OP_EXTCODECOPY_GAS (evm)


#### OP_SSTORE (...)

```c
#define OP_SSTORE (...) EVM_ERROR_UNSUPPORTED_CALL_OPCODE
```


#### EVM_CALL_MODE_STATIC

```c
#define EVM_CALL_MODE_STATIC 1
```


#### EVM_CALL_MODE_DELEGATE

```c
#define EVM_CALL_MODE_DELEGATE 2
```


#### EVM_CALL_MODE_CALLCODE

```c
#define EVM_CALL_MODE_CALLCODE 3
```


#### EVM_CALL_MODE_CALL

```c
#define EVM_CALL_MODE_CALL 4
```


#### evm_state_t

the current state of the evm 


The enum type contains the following values:

```eval_rst
======================== = =====================================
 **EVM_STATE_INIT**      0 just initialised, but not yet started
 **EVM_STATE_RUNNING**   1 started and still running
 **EVM_STATE_STOPPED**   2 successfully stopped
 **EVM_STATE_REVERTED**  3 stopped, but results must be reverted
======================== = =====================================
```

#### evm_get_env

This function provides data from the enviroment. 

depending on the key the function will set the out_data-pointer to the result. This means the enviroment is responsible for memory management and also to clean up resources afterwards.


```c
typedef int(* evm_get_env) (void *evm, uint16_t evm_key, uint8_t *in_data, int in_len, uint8_t **out_data, int offset, int len)
```

returns: `int(*`


#### storage_t


The stuct contains following fields:

```eval_rst
=============================================== =========== 
`bytes32_t <#bytes32-t>`_                        **key**    
`bytes32_t <#bytes32-t>`_                        **value**  
`account_storagestruct , * <#account-storage>`_  **next**   
=============================================== =========== 
```

#### logs_t


The stuct contains following fields:

```eval_rst
========================= ============ 
`bytes_t <#bytes-t>`_      **topics**  
`bytes_t <#bytes-t>`_      **data**    
`logsstruct , * <#logs>`_  **next**    
========================= ============ 
```

#### account_t


The stuct contains following fields:

```eval_rst
=============================== ============= 
`address_t <#address-t>`_        **address**  
`bytes32_t <#bytes32-t>`_        **balance**  
`bytes32_t <#bytes32-t>`_        **nonce**    
`bytes_t <#bytes-t>`_            **code**     
`storage_t * <#storage-t>`_      **storage**  
`accountstruct , * <#account>`_  **next**     
=============================== ============= 
```

#### evm_t


The stuct contains following fields:

```eval_rst
===================================== ====================== ======================================================
`bytes_builder_t <#bytes-builder-t>`_  **stack**             
`bytes_builder_t <#bytes-builder-t>`_  **memory**            
``int``                                **stack_size**        
`bytes_t <#bytes-t>`_                  **code**              
``uint32_t``                           **pos**               
`evm_state_t <#evm-state-t>`_          **state**             
`bytes_t <#bytes-t>`_                  **last_returned**     
`bytes_t <#bytes-t>`_                  **return_data**       
``uint32_t *``                         **invalid_jumpdest**  
``uint32_t``                           **properties**        
`evm_get_env <#evm-get-env>`_          **env**               
``void *``                             **env_ptr**           
``uint64_t``                           **chain_id**          the chain_id as returned by the opcode
``uint8_t *``                          **address**           the address of the current storage
``uint8_t *``                          **account**           the address of the code
``uint8_t *``                          **origin**            the address of original sender of the root-transaction
``uint8_t *``                          **caller**            the address of the parent sender
`bytes_t <#bytes-t>`_                  **call_value**        value send
`bytes_t <#bytes-t>`_                  **call_data**         data send in the tx
`bytes_t <#bytes-t>`_                  **gas_price**         current gasprice
``uint64_t``                           **gas**               
````                                   **gas_options**       
===================================== ====================== ======================================================
```

#### evm_stack_push

```c
int evm_stack_push(evm_t *evm, uint8_t *data, uint8_t len);
```

arguments:
```eval_rst
=================== ========== 
`evm_t * <#evm-t>`_  **evm**   
``uint8_t *``        **data**  
``uint8_t``          **len**   
=================== ========== 
```
returns: `int`


#### evm_stack_push_ref

```c
int evm_stack_push_ref(evm_t *evm, uint8_t **dst, uint8_t len);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint8_t **``       **dst**  
``uint8_t``          **len**  
=================== ========= 
```
returns: `int`


#### evm_stack_push_int

```c
int evm_stack_push_int(evm_t *evm, uint32_t val);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint32_t``         **val**  
=================== ========= 
```
returns: `int`


#### evm_stack_push_long

```c
int evm_stack_push_long(evm_t *evm, uint64_t val);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint64_t``         **val**  
=================== ========= 
```
returns: `int`


#### evm_stack_get_ref

```c
int evm_stack_get_ref(evm_t *evm, uint8_t pos, uint8_t **dst);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint8_t``          **pos**  
``uint8_t **``       **dst**  
=================== ========= 
```
returns: `int`


#### evm_stack_pop

```c
int evm_stack_pop(evm_t *evm, uint8_t *dst, uint8_t len);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint8_t *``        **dst**  
``uint8_t``          **len**  
=================== ========= 
```
returns: `int`


#### evm_stack_pop_ref

```c
int evm_stack_pop_ref(evm_t *evm, uint8_t **dst);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint8_t **``       **dst**  
=================== ========= 
```
returns: `int`


#### evm_stack_pop_byte

```c
int evm_stack_pop_byte(evm_t *evm, uint8_t *dst);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
``uint8_t *``        **dst**  
=================== ========= 
```
returns: `int`


#### evm_stack_pop_int

```c
int32_t evm_stack_pop_int(evm_t *evm);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
=================== ========= 
```
returns: `int32_t`


#### evm_stack_peek_len

```c
int evm_stack_peek_len(evm_t *evm);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
=================== ========= 
```
returns: `int`


#### evm_run

```c
int evm_run(evm_t *evm, address_t code_address);
```

arguments:
```eval_rst
========================= ================== 
`evm_t * <#evm-t>`_        **evm**           
`address_t <#address-t>`_  **code_address**  
========================= ================== 
```
returns: `int`


#### evm_sub_call

```c
int evm_sub_call(evm_t *parent, uint8_t address[20], uint8_t account[20], uint8_t *value, wlen_t l_value, uint8_t *data, uint32_t l_data, uint8_t caller[20], uint8_t origin[20], uint64_t gas, wlen_t mode, uint32_t out_offset, uint32_t out_len);
```

handle internal calls. 

arguments:
```eval_rst
=================== ================ 
`evm_t * <#evm-t>`_  **parent**      
``uint8_t``          **address**     
``uint8_t``          **account**     
``uint8_t *``        **value**       
`wlen_t <#wlen-t>`_  **l_value**     
``uint8_t *``        **data**        
``uint32_t``         **l_data**      
``uint8_t``          **caller**      
``uint8_t``          **origin**      
``uint64_t``         **gas**         
`wlen_t <#wlen-t>`_  **mode**        
``uint32_t``         **out_offset**  
``uint32_t``         **out_len**     
=================== ================ 
```
returns: `int`


#### evm_ensure_memory

```c
int evm_ensure_memory(evm_t *evm, uint32_t max_pos);
```

arguments:
```eval_rst
=================== ============= 
`evm_t * <#evm-t>`_  **evm**      
``uint32_t``         **max_pos**  
=================== ============= 
```
returns: `int`


#### in3_get_env

```c
int in3_get_env(void *evm_ptr, uint16_t evm_key, uint8_t *in_data, int in_len, uint8_t **out_data, int offset, int len);
```

arguments:
```eval_rst
============== ============== 
``void *``      **evm_ptr**   
``uint16_t``    **evm_key**   
``uint8_t *``   **in_data**   
``int``         **in_len**    
``uint8_t **``  **out_data**  
``int``         **offset**    
``int``         **len**       
============== ============== 
```
returns: `int`


#### evm_call

```c
int evm_call(void *vc, uint8_t address[20], uint8_t *value, wlen_t l_value, uint8_t *data, uint32_t l_data, uint8_t caller[20], uint64_t gas, uint64_t chain_id, bytes_t **result);
```

run a evm-call 

arguments:
```eval_rst
======================== ============== 
``void *``                **vc**        
``uint8_t``               **address**   
``uint8_t *``             **value**     
`wlen_t <#wlen-t>`_       **l_value**   
``uint8_t *``             **data**      
``uint32_t``              **l_data**    
``uint8_t``               **caller**    
``uint64_t``              **gas**       
``uint64_t``              **chain_id**  
`bytes_t ** <#bytes-t>`_  **result**    
======================== ============== 
```
returns: `int`


#### evm_print_stack

```c
void evm_print_stack(evm_t *evm, uint64_t last_gas, uint32_t pos);
```

arguments:
```eval_rst
=================== ============== 
`evm_t * <#evm-t>`_  **evm**       
``uint64_t``         **last_gas**  
``uint32_t``         **pos**       
=================== ============== 
```

#### evm_free

```c
void evm_free(evm_t *evm);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
=================== ========= 
```

#### evm_execute

```c
int evm_execute(evm_t *evm);
```

arguments:
```eval_rst
=================== ========= 
`evm_t * <#evm-t>`_  **evm**  
=================== ========= 
```
returns: `int`


### gas.h

evm gas defines. 

File: [src/verifier/eth1/evm/gas.h](https://github.com/slockit/in3-c/blob/master/src/verifier/eth1/evm/gas.h)

#### op_exec (m,gas)

```c
#define op_exec (m,gas) return m;
```


#### subgas (g)


#### GAS_CC_NET_SSTORE_NOOP_GAS

Once per SSTORE operation if the value doesn't change. 

```c
#define GAS_CC_NET_SSTORE_NOOP_GAS 200
```


#### GAS_CC_NET_SSTORE_INIT_GAS

Once per SSTORE operation from clean zero. 

```c
#define GAS_CC_NET_SSTORE_INIT_GAS 20000
```


#### GAS_CC_NET_SSTORE_CLEAN_GAS

Once per SSTORE operation from clean non-zero. 

```c
#define GAS_CC_NET_SSTORE_CLEAN_GAS 5000
```


#### GAS_CC_NET_SSTORE_DIRTY_GAS

Once per SSTORE operation from dirty. 

```c
#define GAS_CC_NET_SSTORE_DIRTY_GAS 200
```


#### GAS_CC_NET_SSTORE_CLEAR_REFUND

Once per SSTORE operation for clearing an originally existing storage slot. 

```c
#define GAS_CC_NET_SSTORE_CLEAR_REFUND 15000
```


#### GAS_CC_NET_SSTORE_RESET_REFUND

Once per SSTORE operation for resetting to the original non-zero value. 

```c
#define GAS_CC_NET_SSTORE_RESET_REFUND 4800
```


#### GAS_CC_NET_SSTORE_RESET_CLEAR_REFUND

Once per SSTORE operation for resetting to the original zero valuev. 

```c
#define GAS_CC_NET_SSTORE_RESET_CLEAR_REFUND 19800
```


#### G_ZERO

Nothing is paid for operations of the set Wzero. 

```c
#define G_ZERO 0
```


#### G_JUMPDEST

JUMP DEST. 

```c
#define G_JUMPDEST 1
```


#### G_BASE

This is the amount of gas to pay for operations of the set Wbase. 

```c
#define G_BASE 2
```


#### G_VERY_LOW

This is the amount of gas to pay for operations of the set Wverylow. 

```c
#define G_VERY_LOW 3
```


#### G_LOW

This is the amount of gas to pay for operations of the set Wlow. 

```c
#define G_LOW 5
```


#### G_MID

This is the amount of gas to pay for operations of the set Wmid. 

```c
#define G_MID 8
```


#### G_HIGH

This is the amount of gas to pay for operations of the set Whigh. 

```c
#define G_HIGH 10
```


#### G_EXTCODE

This is the amount of gas to pay for operations of the set Wextcode. 

```c
#define G_EXTCODE 700
```


#### G_BALANCE

This is the amount of gas to pay for a BALANCE operation. 

```c
#define G_BALANCE 400
```


#### G_SLOAD

This is paid for an SLOAD operation. 

```c
#define G_SLOAD 200
```


#### G_SSET

This is paid for an SSTORE operation when the storage value is set to non-zero from zero. 

```c
#define G_SSET 20000
```


#### G_SRESET

This is the amount for an SSTORE operation when the storage value's zeroness remains unchanged or is set to zero. 

```c
#define G_SRESET 5000
```


#### R_SCLEAR

This is the refund given (added into the refund counter) when the storage value is set to zero from non-zero. 

```c
#define R_SCLEAR 15000
```


#### R_SELFDESTRUCT

This is the refund given (added into the refund counter) for self-destructing an account. 

```c
#define R_SELFDESTRUCT 24000
```


#### G_SELFDESTRUCT

This is the amount of gas to pay for a SELFDESTRUCT operation. 

```c
#define G_SELFDESTRUCT 5000
```


#### G_CREATE

This is paid for a CREATE operation. 

```c
#define G_CREATE 32000
```


#### G_CODEDEPOSIT

This is paid per byte for a CREATE operation to succeed in placing code into the state. 

```c
#define G_CODEDEPOSIT 200
```


#### G_CALL

This is paid for a CALL operation. 

```c
#define G_CALL 700
```


#### G_CALLVALUE

This is paid for a non-zero value transfer as part of the CALL operation. 

```c
#define G_CALLVALUE 9000
```


#### G_CALLSTIPEND

This is a stipend for the called contract subtracted from Gcallvalue for a non-zero value transfer. 

```c
#define G_CALLSTIPEND 2300
```


#### G_NEWACCOUNT

This is paid for a CALL or for a SELFDESTRUCT operation which creates an account. 

```c
#define G_NEWACCOUNT 25000
```


#### G_EXP

This is a partial payment for an EXP operation. 

```c
#define G_EXP 10
```


#### G_EXPBYTE

This is a partial payment when multiplied by dlog256(exponent)e for the EXP operation. 

```c
#define G_EXPBYTE 50
```


#### G_MEMORY

This is paid for every additional word when expanding memory. 

```c
#define G_MEMORY 3
```


#### G_TXCREATE

This is paid by all contract-creating transactions after the Homestead transition. 

```c
#define G_TXCREATE 32000
```


#### G_TXDATA_ZERO

This is paid for every zero byte of data or code for a transaction. 

```c
#define G_TXDATA_ZERO 4
```


#### G_TXDATA_NONZERO

This is paid for every non-zero byte of data or code for a transaction. 

```c
#define G_TXDATA_NONZERO 68
```


#### G_TRANSACTION

This is paid for every transaction. 

```c
#define G_TRANSACTION 21000
```


#### G_LOG

This is a partial payment for a LOG operation. 

```c
#define G_LOG 375
```


#### G_LOGDATA

This is paid for each byte in a LOG operation's data. 

```c
#define G_LOGDATA 8
```


#### G_LOGTOPIC

This is paid for each topic of a LOG operation. 

```c
#define G_LOGTOPIC 375
```


#### G_SHA3

This is paid for each SHA3 operation. 

```c
#define G_SHA3 30
```


#### G_SHA3WORD

This is paid for each word (rounded up) for input data to a SHA3 operation. 

```c
#define G_SHA3WORD 6
```


#### G_COPY

This is a partial payment for *COPY operations, multiplied by the number of words copied, rounded up. 

```c
#define G_COPY 3
```


#### G_BLOCKHASH

This is a payment for a BLOCKHASH operation. 

```c
#define G_BLOCKHASH 20
```


#### G_PRE_EC_RECOVER

Precompile EC RECOVER. 

```c
#define G_PRE_EC_RECOVER 3000
```


#### G_PRE_SHA256

Precompile SHA256. 

```c
#define G_PRE_SHA256 60
```


#### G_PRE_SHA256_WORD

Precompile SHA256 per word. 

```c
#define G_PRE_SHA256_WORD 12
```


#### G_PRE_RIPEMD160

Precompile RIPEMD160. 

```c
#define G_PRE_RIPEMD160 600
```


#### G_PRE_RIPEMD160_WORD

Precompile RIPEMD160 per word. 

```c
#define G_PRE_RIPEMD160_WORD 120
```


#### G_PRE_IDENTITY

Precompile IDENTIY (copyies data) 

```c
#define G_PRE_IDENTITY 15
```


#### G_PRE_IDENTITY_WORD

Precompile IDENTIY per word. 

```c
#define G_PRE_IDENTITY_WORD 3
```


#### G_PRE_MODEXP_GQUAD_DIVISOR

Gquaddivisor from modexp precompile for gas calculation. 

```c
#define G_PRE_MODEXP_GQUAD_DIVISOR 20
```


#### G_PRE_ECADD

Gas costs for curve addition precompile. 

```c
#define G_PRE_ECADD 500
```


#### G_PRE_ECMUL

Gas costs for curve multiplication precompile. 

```c
#define G_PRE_ECMUL 40000
```


#### G_PRE_ECPAIRING

Base gas costs for curve pairing precompile. 

```c
#define G_PRE_ECPAIRING 100000
```


#### G_PRE_ECPAIRING_WORD

Gas costs regarding curve pairing precompile input length. 

```c
#define G_PRE_ECPAIRING_WORD 80000
```


#### EVM_STACK_LIMIT

max elements of the stack 

```c
#define EVM_STACK_LIMIT 1024
```


#### EVM_MAX_CODE_SIZE

max size of the code 

```c
#define EVM_MAX_CODE_SIZE 24576
```


#### FRONTIER_G_EXPBYTE

fork values 

This is a partial payment when multiplied by dlog256(exponent)e for the EXP operation. 

```c
#define FRONTIER_G_EXPBYTE 10
```


#### FRONTIER_G_SLOAD

This is a partial payment when multiplied by dlog256(exponent)e for the EXP operation. 

```c
#define FRONTIER_G_SLOAD 50
```


#### FREE_EVM (...)


#### INIT_EVM (...)


#### INIT_GAS (...)


#### SUBGAS (...)


#### FINALIZE_SUBCALL_GAS (...)


#### UPDATE_SUBCALL_GAS (...)


#### FINALIZE_AND_REFUND_GAS (...)


#### KEEP_TRACK_GAS (evm)

```c
#define KEEP_TRACK_GAS (evm) 0
```


#### SELFDESTRUCT_GAS (evm,g)

```c
#define SELFDESTRUCT_GAS (evm,g) EVM_ERROR_UNSUPPORTED_CALL_OPCODE
```


#### UPDATE_ACCOUNT_CODE (...)


## Module verifier/eth1/full 




### eth_full.h

Ethereum Nanon verification. 

File: [src/verifier/eth1/full/eth_full.h](https://github.com/slockit/in3-c/blob/master/src/verifier/eth1/full/eth_full.h)

#### in3_verify_eth_full

```c
int in3_verify_eth_full(in3_vctx_t *v);
```

entry-function to execute the verification context. 

arguments:
```eval_rst
============================= ======= 
`in3_vctx_t * <#in3-vctx-t>`_  **v**  
============================= ======= 
```
returns: `int`


#### in3_register_eth_full

```c
void in3_register_eth_full();
```

this function should only be called once and will register the eth-full verifier. 


## Module verifier/eth1/nano 




### chainspec.h

Ethereum chain specification 

File: [src/verifier/eth1/nano/chainspec.h](https://github.com/slockit/in3-c/blob/master/src/verifier/eth1/nano/chainspec.h)

#### BLOCK_LATEST

```c
#define BLOCK_LATEST 0xFFFFFFFFFFFFFFFF
```


#### eip_transition_t


The stuct contains following fields:

```eval_rst
============ ====================== 
``uint64_t``  **transition_block**  
``eip_t``     **eips**              
============ ====================== 
```

#### consensus_transition_t


The stuct contains following fields:

```eval_rst
=============================================== ====================== 
``uint64_t``                                     **transition_block**  
`eth_consensus_type_t <#eth-consensus-type-t>`_  **type**              
`bytes_t <#bytes-t>`_                            **validators**        
``uint8_t *``                                    **contract**          
=============================================== ====================== 
```

#### chainspec_t


The stuct contains following fields:

```eval_rst
===================================================== =============================== 
``uint64_t``                                           **network_id**                 
``uint64_t``                                           **account_start_nonce**        
``uint32_t``                                           **eip_transitions_len**        
`eip_transition_t * <#eip-transition-t>`_              **eip_transitions**            
``uint32_t``                                           **consensus_transitions_len**  
`consensus_transition_t * <#consensus-transition-t>`_  **consensus_transitions**      
===================================================== =============================== 
```

#### __attribute__

```c
struct __attribute__((__packed__)) eip_;
```

defines the flags for the current activated EIPs. 

Since it does not make sense to support a evm defined before Homestead, homestead EIP is always turned on! 

< REVERT instruction

< Bitwise shifting instructions in EVM

< Gas cost changes for IO-heavy operations

< Simple replay attack protection

< EXP cost increase

< Contract code size limit

< Precompiled contracts for addition and scalar multiplication on the elliptic curve alt_bn128 




< Precompiled contracts for optimal ate pairing check on the elliptic curve alt_bn128 




< Big integer modular exponentiation 




< New opcodes: RETURNDATASIZE and RETURNDATACOPY 




< New opcode STATICCALL 




< Embedding transaction status code in receipts

< Skinny CREATE2 




< EXTCODEHASH opcode

< Net gas metering for SSTORE without dirty maps 




arguments:
```eval_rst
================  
``(__packed__)``  
================  
```
returns: `struct`


#### chainspec_create_from_json

```c
chainspec_t* chainspec_create_from_json(d_token_t *data);
```

arguments:
```eval_rst
=========================== ========== 
`d_token_t * <#d-token-t>`_  **data**  
=========================== ========== 
```
returns: [`chainspec_t *`](#chainspec-t)


#### chainspec_get_eip

```c
eip_t chainspec_get_eip(chainspec_t *spec, uint64_t block_number);
```

arguments:
```eval_rst
=============================== ================== 
`chainspec_t * <#chainspec-t>`_  **spec**          
``uint64_t``                     **block_number**  
=============================== ================== 
```
returns: `eip_t`


#### chainspec_get_consensus

```c
consensus_transition_t* chainspec_get_consensus(chainspec_t *spec, uint64_t block_number);
```

arguments:
```eval_rst
=============================== ================== 
`chainspec_t * <#chainspec-t>`_  **spec**          
``uint64_t``                     **block_number**  
=============================== ================== 
```
returns: [`consensus_transition_t *`](#consensus-transition-t)


#### chainspec_to_bin

```c
in3_ret_t chainspec_to_bin(chainspec_t *spec, bytes_builder_t *bb);
```

arguments:
```eval_rst
======================================= ========== 
`chainspec_t * <#chainspec-t>`_          **spec**  
`bytes_builder_t * <#bytes-builder-t>`_  **bb**    
======================================= ========== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### chainspec_from_bin

```c
chainspec_t* chainspec_from_bin(void *raw);
```

arguments:
```eval_rst
========== ========= 
``void *``  **raw**  
========== ========= 
```
returns: [`chainspec_t *`](#chainspec-t)


#### chainspec_get

```c
chainspec_t* chainspec_get(chain_id_t chain_id);
```

arguments:
```eval_rst
=========================== ============== 
`chain_id_t <#chain-id-t>`_  **chain_id**  
=========================== ============== 
```
returns: [`chainspec_t *`](#chainspec-t)


#### chainspec_put

```c
void chainspec_put(chain_id_t chain_id, chainspec_t *spec);
```

arguments:
```eval_rst
=============================== ============== 
`chain_id_t <#chain-id-t>`_      **chain_id**  
`chainspec_t * <#chainspec-t>`_  **spec**      
=============================== ============== 
```

### eth_nano.h

Ethereum Nanon verification. 

File: [src/verifier/eth1/nano/eth_nano.h](https://github.com/slockit/in3-c/blob/master/src/verifier/eth1/nano/eth_nano.h)

#### in3_verify_eth_nano

```c
in3_ret_t in3_verify_eth_nano(in3_vctx_t *v);
```

entry-function to execute the verification context. 

arguments:
```eval_rst
============================= ======= 
`in3_vctx_t * <#in3-vctx-t>`_  **v**  
============================= ======= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_blockheader

```c
in3_ret_t eth_verify_blockheader(in3_vctx_t *vc, bytes_t *header, bytes_t *expected_blockhash);
```

verifies a blockheader. 

verifies a blockheader. 

arguments:
```eval_rst
============================= ======================== 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**                  
`bytes_t * <#bytes-t>`_        **header**              
`bytes_t * <#bytes-t>`_        **expected_blockhash**  
============================= ======================== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_signature

```c
int eth_verify_signature(in3_vctx_t *vc, bytes_t *msg_hash, d_token_t *sig);
```

verifies a single signature blockheader. 

This function will return a positive integer with a bitmask holding the bit set according to the address that signed it. This is based on the signatiures in the request-config. 

arguments:
```eval_rst
============================= ============== 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**        
`bytes_t * <#bytes-t>`_        **msg_hash**  
`d_token_t * <#d-token-t>`_    **sig**       
============================= ============== 
```
returns: `int`


#### ecrecover_signature

```c
bytes_t* ecrecover_signature(bytes_t *msg_hash, d_token_t *sig);
```

returns the address of the signature if the msg_hash is correct 

arguments:
```eval_rst
=========================== ============== 
`bytes_t * <#bytes-t>`_      **msg_hash**  
`d_token_t * <#d-token-t>`_  **sig**       
=========================== ============== 
```
returns: [`bytes_t *`](#bytes-t)


#### eth_verify_eth_getTransactionReceipt

```c
in3_ret_t eth_verify_eth_getTransactionReceipt(in3_vctx_t *vc, bytes_t *tx_hash);
```

verifies a transaction receipt. 

arguments:
```eval_rst
============================= ============= 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**       
`bytes_t * <#bytes-t>`_        **tx_hash**  
============================= ============= 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_in3_nodelist

```c
in3_ret_t eth_verify_in3_nodelist(in3_vctx_t *vc, uint32_t node_limit, bytes_t *seed, d_token_t *required_addresses);
```

verifies the nodelist. 

arguments:
```eval_rst
============================= ======================== 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**                  
``uint32_t``                   **node_limit**          
`bytes_t * <#bytes-t>`_        **seed**                
`d_token_t * <#d-token-t>`_    **required_addresses**  
============================= ======================== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### eth_verify_in3_whitelist

```c
in3_ret_t eth_verify_in3_whitelist(in3_vctx_t *vc);
```

verifies the nodelist. 

arguments:
```eval_rst
============================= ======== 
`in3_vctx_t * <#in3-vctx-t>`_  **vc**  
============================= ======== 
```
returns: [`in3_ret_t`](#in3-ret-t) the [result-status](#in3-ret-t) of the function. 

*Please make sure you check if it was successfull (`==IN3_OK`)*


#### in3_register_eth_nano

```c
void in3_register_eth_nano();
```

this function should only be called once and will register the eth-nano verifier. 


#### create_tx_path

```c
bytes_t* create_tx_path(uint32_t index);
```

helper function to rlp-encode the transaction_index. 

The result must be freed after use! 

arguments:
```eval_rst
============ =========== 
``uint32_t``  **index**  
============ =========== 
```
returns: [`bytes_t *`](#bytes-t)


### merkle.h

Merkle Proof Verification. 

File: [src/verifier/eth1/nano/merkle.h](https://github.com/slockit/in3-c/blob/master/src/verifier/eth1/nano/merkle.h)

#### MERKLE_DEPTH_MAX

```c
#define MERKLE_DEPTH_MAX 64
```


#### trie_verify_proof

```c
int trie_verify_proof(bytes_t *rootHash, bytes_t *path, bytes_t **proof, bytes_t *expectedValue);
```

verifies a merkle proof. 

expectedValue == NULL : value must not exist expectedValue.data ==NULL : please copy the data I want to evaluate it afterwards. expectedValue.data !=NULL : the value must match the data.

arguments:
```eval_rst
======================== =================== 
`bytes_t * <#bytes-t>`_   **rootHash**       
`bytes_t * <#bytes-t>`_   **path**           
`bytes_t ** <#bytes-t>`_  **proof**          
`bytes_t * <#bytes-t>`_   **expectedValue**  
======================== =================== 
```
returns: `int`


#### trie_path_to_nibbles

```c
uint8_t* trie_path_to_nibbles(bytes_t path, int use_prefix);
```

helper function split a path into 4-bit nibbles. 

The result must be freed after use!

arguments:
```eval_rst
===================== ================ 
`bytes_t <#bytes-t>`_  **path**        
``int``                **use_prefix**  
===================== ================ 
```
returns: `uint8_t *` : the resulting bytes represent a 4bit-number each and are terminated with a 0xFF. 




#### trie_matching_nibbles

```c
int trie_matching_nibbles(uint8_t *a, uint8_t *b);
```

helper function to find the number of nibbles matching both paths. 

arguments:
```eval_rst
============= ======= 
``uint8_t *``  **a**  
``uint8_t *``  **b**  
============= ======= 
```
returns: `int`


#### trie_free_proof

```c
void trie_free_proof(bytes_t **proof);
```

used to free the NULL-terminated proof-array. 

arguments:
```eval_rst
======================== =========== 
`bytes_t ** <#bytes-t>`_  **proof**  
======================== =========== 
```

### rlp.h

RLP-En/Decoding as described in the [Ethereum RLP-Spec](https://github.com/ethereum/wiki/wiki/RLP).

This decoding works without allocating new memory. 

File: [src/verifier/eth1/nano/rlp.h](https://github.com/slockit/in3-c/blob/master/src/verifier/eth1/nano/rlp.h)

#### rlp_decode

```c
int rlp_decode(bytes_t *b, int index, bytes_t *dst);
```

this function decodes the given bytes and returns the element with the given index by updating the reference of dst. 

the bytes will only hold references and do **not** need to be freed!

```c
bytes_t* tx_raw = serialize_tx(tx);

bytes_t item;

// decodes the tx_raw by letting the item point to range of the first element, which should be the body of a list.
if (rlp_decode(tx_raw, 0, &item) !=2) return -1 ;

// now decode the 4th element (which is the value) and let item point to that range.
if (rlp_decode(&item, 4, &item) !=1) return -1 ;
```
arguments:
```eval_rst
======================= =========== 
`bytes_t * <#bytes-t>`_  **b**      
``int``                  **index**  
`bytes_t * <#bytes-t>`_  **dst**    
======================= =========== 
```
returns: `int` : - 0 : means item out of range
- 1 : item found
- 2 : list found ( you can then decode the same bytes again)




#### rlp_decode_in_list

```c
int rlp_decode_in_list(bytes_t *b, int index, bytes_t *dst);
```

this function expects a list item (like the blockheader as first item and will then find the item within this list). 

It is a shortcut for

```c
// decode the list
if (rlp_decode(b,0,dst)!=2) return 0;
// and the decode the item
return rlp_decode(dst,index,dst);
```
arguments:
```eval_rst
======================= =========== 
`bytes_t * <#bytes-t>`_  **b**      
``int``                  **index**  
`bytes_t * <#bytes-t>`_  **dst**    
======================= =========== 
```
returns: `int` : - 0 : means item out of range
- 1 : item found
- 2 : list found ( you can then decode the same bytes again)




#### rlp_decode_len

```c
int rlp_decode_len(bytes_t *b);
```

returns the number of elements found in the data. 

arguments:
```eval_rst
======================= ======= 
`bytes_t * <#bytes-t>`_  **b**  
======================= ======= 
```
returns: `int`


#### rlp_encode_item

```c
void rlp_encode_item(bytes_builder_t *bb, bytes_t *val);
```

encode a item as single string and add it to the bytes_builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
`bytes_t * <#bytes-t>`_                  **val**  
======================================= ========= 
```

#### rlp_encode_list

```c
void rlp_encode_list(bytes_builder_t *bb, bytes_t *val);
```

encode a the value as list of already encoded items. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**   
`bytes_t * <#bytes-t>`_                  **val**  
======================================= ========= 
```

#### rlp_encode_to_list

```c
bytes_builder_t* rlp_encode_to_list(bytes_builder_t *bb);
```

converts the data in the builder to a list. 

This function is optimized to not increase the memory more than needed and is fastet than creating a second builder to encode the data.

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
======================================= ======== 
```
returns: [`bytes_builder_t *`](#bytes-builder-t) : the same builder. 




#### rlp_encode_to_item

```c
bytes_builder_t* rlp_encode_to_item(bytes_builder_t *bb);
```

converts the data in the builder to a rlp-encoded item. 

This function is optimized to not increase the memory more than needed and is fastet than creating a second builder to encode the data.

arguments:
```eval_rst
======================================= ======== 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**  
======================================= ======== 
```
returns: [`bytes_builder_t *`](#bytes-builder-t) : the same builder. 




#### rlp_add_length

```c
void rlp_add_length(bytes_builder_t *bb, uint32_t len, uint8_t offset);
```

helper to encode the prefix for a value 

arguments:
```eval_rst
======================================= ============ 
`bytes_builder_t * <#bytes-builder-t>`_  **bb**      
``uint32_t``                             **len**     
``uint8_t``                              **offset**  
======================================= ============ 
```

### serialize.h

serialization of ETH-Objects.

This incoming tokens will represent their values as properties based on [JSON-RPC](https://github.com/ethereum/wiki/wiki/JSON-RPC). 

File: [src/verifier/eth1/nano/serialize.h](https://github.com/slockit/in3-c/blob/master/src/verifier/eth1/nano/serialize.h)

#### BLOCKHEADER_PARENT_HASH

```c
#define BLOCKHEADER_PARENT_HASH 0
```


#### BLOCKHEADER_SHA3_UNCLES

```c
#define BLOCKHEADER_SHA3_UNCLES 1
```


#### BLOCKHEADER_MINER

```c
#define BLOCKHEADER_MINER 2
```


#### BLOCKHEADER_STATE_ROOT

```c
#define BLOCKHEADER_STATE_ROOT 3
```


#### BLOCKHEADER_TRANSACTIONS_ROOT

```c
#define BLOCKHEADER_TRANSACTIONS_ROOT 4
```


#### BLOCKHEADER_RECEIPT_ROOT

```c
#define BLOCKHEADER_RECEIPT_ROOT 5
```


#### BLOCKHEADER_LOGS_BLOOM

```c
#define BLOCKHEADER_LOGS_BLOOM 6
```


#### BLOCKHEADER_DIFFICULTY

```c
#define BLOCKHEADER_DIFFICULTY 7
```


#### BLOCKHEADER_NUMBER

```c
#define BLOCKHEADER_NUMBER 8
```


#### BLOCKHEADER_GAS_LIMIT

```c
#define BLOCKHEADER_GAS_LIMIT 9
```


#### BLOCKHEADER_GAS_USED

```c
#define BLOCKHEADER_GAS_USED 10
```


#### BLOCKHEADER_TIMESTAMP

```c
#define BLOCKHEADER_TIMESTAMP 11
```


#### BLOCKHEADER_EXTRA_DATA

```c
#define BLOCKHEADER_EXTRA_DATA 12
```


#### BLOCKHEADER_SEALED_FIELD1

```c
#define BLOCKHEADER_SEALED_FIELD1 13
```


#### BLOCKHEADER_SEALED_FIELD2

```c
#define BLOCKHEADER_SEALED_FIELD2 14
```


#### BLOCKHEADER_SEALED_FIELD3

```c
#define BLOCKHEADER_SEALED_FIELD3 15
```


#### serialize_tx_receipt

```c
bytes_t* serialize_tx_receipt(d_token_t *receipt);
```

creates rlp-encoded raw bytes for a receipt. 

The bytes must be freed with b_free after use!

arguments:
```eval_rst
=========================== ============= 
`d_token_t * <#d-token-t>`_  **receipt**  
=========================== ============= 
```
returns: [`bytes_t *`](#bytes-t)


#### serialize_tx

```c
bytes_t* serialize_tx(d_token_t *tx);
```

creates rlp-encoded raw bytes for a transaction. 

The bytes must be freed with b_free after use!

arguments:
```eval_rst
=========================== ======== 
`d_token_t * <#d-token-t>`_  **tx**  
=========================== ======== 
```
returns: [`bytes_t *`](#bytes-t)


#### serialize_tx_raw

```c
bytes_t* serialize_tx_raw(bytes_t nonce, bytes_t gas_price, bytes_t gas_limit, bytes_t to, bytes_t value, bytes_t data, uint64_t v, bytes_t r, bytes_t s);
```

creates rlp-encoded raw bytes for a transaction from direct values. 

The bytes must be freed with b_free after use! 

arguments:
```eval_rst
===================== =============== 
`bytes_t <#bytes-t>`_  **nonce**      
`bytes_t <#bytes-t>`_  **gas_price**  
`bytes_t <#bytes-t>`_  **gas_limit**  
`bytes_t <#bytes-t>`_  **to**         
`bytes_t <#bytes-t>`_  **value**      
`bytes_t <#bytes-t>`_  **data**       
``uint64_t``           **v**          
`bytes_t <#bytes-t>`_  **r**          
`bytes_t <#bytes-t>`_  **s**          
===================== =============== 
```
returns: [`bytes_t *`](#bytes-t)


#### serialize_account

```c
bytes_t* serialize_account(d_token_t *a);
```

creates rlp-encoded raw bytes for a account. 

The bytes must be freed with b_free after use! 

arguments:
```eval_rst
=========================== ======= 
`d_token_t * <#d-token-t>`_  **a**  
=========================== ======= 
```
returns: [`bytes_t *`](#bytes-t)


#### serialize_block_header

```c
bytes_t* serialize_block_header(d_token_t *block);
```

creates rlp-encoded raw bytes for a blockheader. 

The bytes must be freed with b_free after use!

arguments:
```eval_rst
=========================== =========== 
`d_token_t * <#d-token-t>`_  **block**  
=========================== =========== 
```
returns: [`bytes_t *`](#bytes-t)


#### rlp_add

```c
int rlp_add(bytes_builder_t *rlp, d_token_t *t, int ml);
```

adds the value represented by the token rlp-encoded to the byte_builder. 

arguments:
```eval_rst
======================================= ========= 
`bytes_builder_t * <#bytes-builder-t>`_  **rlp**  
`d_token_t * <#d-token-t>`_              **t**    
``int``                                  **ml**   
======================================= ========= 
```
returns: `int` : 0 if added -1 if the value could not be handled. 




