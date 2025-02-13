<div align="center">

# **Bittensor** <!-- omit in toc -->
[![Discord Chat](https://img.shields.io/discord/308323056592486420.svg)](https://discord.gg/bittensor)
[![PyPI version](https://badge.fury.io/py/bittensor.svg)](https://badge.fury.io/py/bittensor)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT) 

---

### Internet-scale Neural Networks <!-- omit in toc -->

[Discord](https://discord.gg/bittensor) • [Docs](https://docs.bittensor.com/) • [Network](https://www.bittensor.com/network) • [Research](https://drive.google.com/file/d/1VnsobL6lIAAqcA1_Tbm8AYIQscfJV4KU) • [Code](https://github.com/opentensor/BitTensor)

</div>

This repository contains Bittensor's Python API, which can be used for the following purposes:

1. Querying the Bittensor network as a [client](https://github.com/opentensor/bittensor#31-client).
2. Running and building Bittensor miners and validators for [mining TAO](https://github.com/opentensor/bittensor#43-running-a-template-miner).
3. Pulling network [state information](https://github.com/opentensor/bittensor#3-using-bittensor).
4. Managing [TAO wallets](https://github.com/opentensor/bittensor#41-cli), balances, transfers, etc.

Bittensor is a mining network, similar to Bitcoin, that includes built-in incentives designed to encourage miners to provide value by hosting trained or training machine learning models. These models can be queried by clients seeking inference over inputs, such as token-based text generations or numerical embeddings from a large foundation model like GPT-NeoX-20B.

Token-based incentives are designed to drive the network's growth and distribute the value generated by the network directly to the individuals producing that value, without intermediaries. The network is open to all participants, and no individual or group has full control over what is learned, who can profit from it, or who can access it.

To learn more about Bittensor, please read our [paper](https://drive.google.com/file/d/1VnsobL6lIAAqcA1_Tbm8AYIQscfJV4KU/view).

- [1. Documentation](#1-documentation)
- [2. Install](#2-install)
- [3. Using Bittensor](#3-using-bittensor)
  - [3.1. Client](#31-client)
  - [3.2. Server](#32-server)
  - [3.3. Validator](#33-validator)
- [4. Features](#4-features)
  - [4.1. Using the CLI](#41-cli)
  - [4.2. Selecting the network to join](#42-selecting-the-network-to-join)
  - [4.3. Selecting ports to use](https://github.com/quac88/bittensor/blob/master/README.md#43-selecting-ports-to-use)
  - [4.4. Running a core validator](#43-running-a-template-miner)
  - [4.5. Running a core server](#44-running-a-template-server)
  - [4.6. Syncing with the chain/ Finding the ranks/stake/uids of other nodes](#46-syncing-with-the-chain-finding-the-ranksstakeuids-of-other-nodes)
  - [4.7. Finding and creating the endpoints for other nodes in the network](#47-finding-and-creating-the-endpoints-for-other-nodes-in-the-network)
  - [4.8. Querying others in the network](#48-querying-others-in-the-network)
- [5. Release](#5-release)
- [6. License](#6-license)
- [7. Acknowledgments](#7-acknowledgments)

## 1. Documentation

https://docs.bittensor.com/

## 2. Install
Three ways to install Bittensor

1. Through the installer:
```
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/opentensor/bittensor/master/scripts/install.sh)"
```

2. With pip:
```bash
$ pip3 install bittensor
```

3. From source:
```
$ git clone https://github.com/opentensor/bittensor.git
$ python3 -m pip install -e bittensor/
```

## 3. Using Bittensor

The following examples showcase how to use the Bittensor API for 3 separate purposes.

### 3.1. Client 

Querying the network for generations.

```python
import bittensor
subtensor = bittensor.subtensor( network = 'nakamoto' )
wallet = bittensor.wallet().create_if_non_existent()
graph = bittensor.metagraph( subtensor = subtensor ).sync()
print ( bittensor.dendrite( subtensor = subtensor, wallet = wallet ).generate
        ( 
            endpoints = graph.endpoints[graph.incentive.sort()[1][-1]],  # The highest ranked peer.
            prompt = "The quick brown fox jumped over the lazy dog", 
            num_to_generate = 20
        )
)
```

Querying the network for representations.

```python
import bittensor
subtensor = bittensor.subtensor( network = 'nakamoto' )
wallet = bittensor.wallet().create_if_non_existent()
graph = bittensor.metagraph( subtensor = subtensor ).sync()
print ( bittensor.dendrite( subtensor = subtensor, wallet = wallet ).text_last_hidden_state
        (
            endpoints = graph.endpoints[graph.incentive.sort()[1][-1]],  # The highest ranked peer.
            inputs = "The quick brown fox jumped over the lazy dog"
        )
)
...
# Apply model. 
...
loss.backward() # Accumulate gradients on endpoints.
```

### 3.2. Server

Serving a custom model.

```python
import bittensor
import torch
from transformers import GPT2Model, GPT2Config

model = GPT2Model( GPT2Config(vocab_size = bittensor.__vocab_size__, n_embd = bittensor.__network_dim__ , n_head = 8))
optimizer = torch.optim.SGD( [ {"params": model.parameters()} ], lr = 0.01 )

def forward_text( pubkey, inputs_x ):
    return model( inputs_x )
  
def backward_text( pubkey, inputs_x, grads_dy ):
    with torch.enable_grad():
        outputs_y = model( inputs_x.to(device) ).last_hidden_state
        torch.autograd.backward (
            tensors = [ outputs_y.to(device) ],
            grad_tensors = [ grads_dy.to(device) ]
        )
        optimizer.step()
        optimizer.zero_grad() 
        
subtensor = bittensor.subtensor( network = 'nakamoto' )
wallet = bittensor.wallet().create().register( subtensor = subtensor )
axon = bittensor.axon (
    subtensor = subtensor,
    wallet = wallet,
    forward_text = forward_text,
    backward_text = backward_text
).start().serve()
bittensor.neurons.text.core_server.neuron( subtensor = subtensor ).run()
```
*A Server will default to port '8091.' You can change this by modifying the config to a port of your choosing.*

### 3.3. Validator 

Validating models by setting weights.

```python
import bittensor
import torch

graph = bittensor.metagraph().sync()
dataset = bittensor.dataset()
chain_weights = torch.ones( [graph.n.item()], dtype = torch.float32 )

for batch in dataset.dataloader( 10 ):
    ...
    # Train chain_weights.
    ...
bittensor.subtensor().set_weights (
    weights = chain_weights,
    uids = graph.uids,
    wait_for_inclusion = True,
    wallet = bittensor.wallet(),
)
```
## 4. Features

### 4.1. CLI

Creating a new wallet.
```bash
$ btcli new_coldkey
$ btcli new_hotkey
```

Listing your wallets
```bash
$ btcli list
```

Registering a wallet ~ after installing [cubit](https://github.com/opentensor/cubit/#install)
```bash
$ btcli register --cuda --cuda.TPB 256 --cuda.update_interval 50_000
```
*~ note that these cuda parameters may need to be [adjusted](https://docs.bittensor.com/Arguments.html#cuda)*

Running a miner
```bash
$ btcli run
```

Checking balances
```bash
$ btcli overview
```

View the metagraph.
```bash
$ btcli metagraph
```

Transfering funds
```bash
$ btcli transfer
```

Staking/Unstaking from a hotkey
```bash
$ btcli stake
$ btcli unstake
```

### 4.2. Selecting the network to join 
There are two open Bittensor networks: staging (Nobunaga) and main (Nakamoto, Local).

- Nobunaga (staging)
- Nakamoto (main)
- Local (localhost, mirrors nakamoto)

```bash
$ export NETWORK=local 
$ python (..) --subtensor.network $NETWORK
or
$ btcli run (..) --subtensor.network $NETWORK
```

### 4.3. Selecting ports to use
Each instance of your core server will need to run on a different port
```bash
$ export BT_AXON_PORT=<>
python (..) --axon.port <>
or 
$ btcli run (..) --axon.port <>
```

### 4.4. Running a core validator

The following command will run Bittensor's core validator

```bash
$ cd bittensor
$ python ./bittensor/_neuron/text/core_validator/main.py
```

OR with customized settings

```bash
$ cd bittensor
$ python3 ./bittensor/_neuron/text/core_validator/main.py --wallet.name <WALLET NAME> --wallet.hotkey <HOTKEY NAME>
```

For the full list of settings, please run

```bash
$ python3 ~/.bittensor/bittensor/bittensor/_neuron/neurons/text/core_validator/main.py --help
```

### 4.5. Running a core server

The core server follows a similar run structure as the core validator. 

```bash
$ cd bittensor
$ python3 ./bittensor/_neuron/text/core_server/main.py --wallet.name <WALLET NAME> --wallet.hotkey <HOTKEY NAME>
```

For the full list of settings, please run

```bash
$ cd bittensor
$ python3 ./bittensor/_neuron/text/core_server/main.py --help
```
~note that you will need to set the port, model, and device as CPU or GPU.

### 4.6. Syncing with the chain/ Finding the ranks/stake/uids of other nodes

Information from the chain is collected/formated by the metagraph.

```bash
btcli metagraph
```
and
```python
import bittensor

meta = bittensor.metagraph()
meta.sync()

# --- uid ---
print(meta.uids)

# --- hotkeys ---
print(meta.hotkeys)

# --- ranks ---
print(meta.R)

# --- stake ---
print(meta.S)
```

### 4.7. Finding and creating the endpoints for other nodes in the network

```python
import bittensor

subtensor = bittensor.subtensor( network = 'nakamoto' )
meta = bittensor.metagraph( subtensor = subtensor )
meta.sync()

### Address for the node uid 0
endpoint_as_tensor = meta.endpoints[0]
endpoint_as_object = meta.endpoint_objs[0]
```

### 4.8. Querying others in the network

```python
import bittensor

subtensor = bittensor.subtensor( network = 'nakamoto' )
meta = bittensor.metagraph( subtensor = subtensor )
meta.sync()

### Address for the node uid 0
endpoint_0 = meta.endpoints[0]

### Creating the wallet, and dendrite
wallet = bittensor.wallet().create().register( subtensor = subtensor )
den = bittensor.dendrite(wallet = wallet)
representations, _, _ = den.forward_text (
    endpoints = endpoint_0,
    inputs = "Hello World"
)
```

## 5. Release
The release manager should follow the instructions of the [RELEASE_GUIDELINES.md](./RELEASE_GUIDELINES.md) document.

## 6. License
The MIT License (MIT)
Copyright © 2021 Yuma Rao

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.


## 7. Acknowledgments
**learning-at-home/hivemind**
