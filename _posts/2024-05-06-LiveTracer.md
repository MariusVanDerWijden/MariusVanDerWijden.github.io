---
layout:        post
title:         The go-ethereum live tracer
date:          2024-05-06 10:00:00
summary:       In this post, I share an example tracer built which logs all transactions that concern your address (both ETH transfers and token transfers)
categories:    blog
tags:          ethereum livetracer tracing 
---

We recently merged a long awaited update for our tracing infrastructure into geth. 
Sina Mahmoodi has been working on it for quite some time and since I was busy with other stuff, 
I hadn't had the time to really look into it. 
So today I decided to check the new live tracing framework out and build a small tracer to showcase its capabilities.

Our [website](https://geth.ethereum.org/docs/developers/evm-tracing/live-tracing) has a short tutorial for building live tracers and there's a no-op tracer in the code that can be used as a template.

The tracer I build collects all transactions that concern a set of addresses. This is particularly useful if you want to see all transactions that send or received funds from your addresses. There are other solutions to this, mainly by looking at the receipts and direct transfers, but this will miss some sources of funds (selfdestructs, nested transfers, ...).

If you want to be 100% accurrate you need to look at all balance changes of your address and at all storage slots that correspond to your address of all contracts. However since the layout of mappings in solidity is quite complex, I decided not to be 100% accurate and omit tokens that do not emit events.

# The tracer
Once you copied the tracer code (in the Appendix) to `eth/tracers/live/address.go` and rebuild geth, you can run the live tracer with the command `./build/bin/geth --vmtrace "address" --vmtrace.jsonconfig '{"path":".", "addresses":["0xb02A2EdA1b317FBd16760128836B0Ac59B560e9D"]}'`

There are some caveats with this basic tracer though, balance changes can be reverted if a transaction reverts and it does only pick up on very specific events from ERC-20 contracts. It does not correctly track non-standard tokens. Also the tracer will overwrite any existing csv files on startup.

## Output

The tracer will output CSV files for every address you specified in the config. 
The CSV files look like this:
```
TxHash,Sender,Currency,Previous,New,Reason
0x48791fbdfcdfb63c4e1c424eeeb4c1cbbb635acc96339da79988b255caf91f7c,0x75a8326d027C65F9Db2d973Eb736D3c8127F7da7,ETH,0,10000000000000000000000000,10
0x8f7f0a2c61de8ac4f1985085c01c895c02bdd729fb5373b2ff32abacaa6f220a,0xb02A2EdA1b317FBd16760128836B0Ac59B560e9D,ETH,10000000000000000000000000,9999999999985924314901000,6
0x8f7f0a2c61de8ac4f1985085c01c895c02bdd729fb5373b2ff32abacaa6f220a,0xb02A2EdA1b317FBd16760128836B0Ac59B560e9D,ETH,9999999999985924314901000,9999999994985924314901000,10
0x908406379a3416aeeab318ca9b9f1ee644301bfa37b89c0668aabe53b926446b,
```
and can be consumed by all programs that use CSV (like Excel/Libreoffice).

The log reason can be decoded using the declarations in `core/tracing/hooks.go`

# Conclusion 

Building a custom live tracer has never been easier (or more fun). The framework allows for creating hooks into multiple different levels, from the block level (with OnBlockStart, OnBlockEnd), via the transaction level (OnTxStart, OnTxEnd) and the call level (OnEnter, OnExit) down to the Opcode level (OnOpcode). 
This allows developers to create complex tracers for their tracing needs. 

The tracers are run continuously during block processing and thus can be used for creating custom indizes for the blockchain. The examples for this are endless, you can create a database of all users that interacted with your contract or you can log events for interactions that don't emit events, writing indexers for ordinal type systems becomes a breeze.

Overall its a super nice way in my opinion to build custom tracers.
Kudos to @sina_mahmoodi for creating this framework.

Theres more documentation for the live tracer on our website: https://geth.ethereum.org/docs/developers/evm-tracing/live-tracing

# Appendix

```go
package live

import (
	"encoding/csv"
	"encoding/json"
	"fmt"
	"math/big"
	"os"

	"github.com/ethereum/go-ethereum/accounts/abi"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/tracing"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/eth/tracers"
)

func init() {
	tracers.LiveDirectory.Register("address", newAddressTracer)
}

// addressTracer is a live tracer that tracks all transactions that an address was involved in.
type addressTracer struct {
	addresses []common.Address
	writer    map[common.Address]*csv.Writer

	currentHash   common.Hash
	currentSender common.Address
}

type addressTracerConfig struct {
	Path      string   `json:"path"`      // Path to the directory where the tracer logs will be stored
	Addresses []string `json:"addresses"` // Addresses to be watched
}

func newAddressTracer(cfg json.RawMessage) (*tracing.Hooks, error) {
	var config addressTracerConfig
	if cfg != nil {
		if err := json.Unmarshal(cfg, &config); err != nil {
			return nil, fmt.Errorf("failed to parse config: %v", err)
		}
		if config.Path == "" {
			return nil, fmt.Errorf("no path provided")
		}
	}

	var (
		addresses []common.Address
		writer    = make(map[common.Address]*csv.Writer)
	)
	for _, addr := range config.Addresses {
		a := common.HexToAddress(addr)
		addresses = append(addresses, a)
		file, err := os.Create(fmt.Sprintf("%v/%v.csv", config.Path, addr))
		if err != nil {
			return nil, err
		}
		writer[a] = csv.NewWriter(file)
		writer[a].Write([]string{
			"TxHash",
			"Sender",
			"Currency",
			"Previous",
			"New",
			"Reason",
		})
		writer[a].Flush()
	}

	t := &addressTracer{
		addresses: addresses,
		writer:    writer,
	}
	return &tracing.Hooks{
		OnTxStart:       t.OnTxStart,
		OnBalanceChange: t.OnBalanceChange,
		OnStorageChange: t.OnStorageChange,
	}, nil
}

func (t *addressTracer) OnTxStart(vm *tracing.VMContext, tx *types.Transaction, from common.Address) {
	t.currentHash = tx.Hash()
	t.currentSender = from
}

func (t *addressTracer) OnBalanceChange(a common.Address, prev, new *big.Int, reason tracing.BalanceChangeReason) {
	for _, addr := range t.addresses {
		if addr == a {
			t.writeRecord(addr, "ETH", prev.String(), new.String(), fmt.Sprint(reason))
			break
		}
	}
}

var (
	transferTopic   = crypto.Keccak256Hash([]byte("Transfer(address,address,uint256)"))
	transferType, _ = abi.NewType("tuple(address,address,uint256)", "", nil)
)

func (t *addressTracer) OnLog(l *types.Log) {
	for _, topic := range l.Topics {
		if topic == transferTopic {
			unpacked, err := (abi.Arguments{{Type: transferType}}).Unpack(l.Data[4:])
			if err != nil {
				continue
			}
			from, ok := unpacked[0].(common.Address)
			if !ok {
				continue
			}
			to, ok := unpacked[1].(common.Address)
			if !ok {
				continue
			}
			tokens, ok := unpacked[2].(*big.Int)
			if !ok {
				continue
			}
			for _, addr := range t.addresses {
				if addr == from || addr == to {
					t.writeRecord(addr, l.Address.Hex(), "", tokens.String(), "token transfer")
					break
				}
			}
		}
	}
}

func (t *addressTracer) OnStorageChange(a common.Address, k, prev, new common.Hash) {
	slot := common.BytesToAddress(k.Bytes()[12:])
	for _, addr := range t.addresses {
		if addr == slot {
			t.writeRecord(addr, a.Hex(), prev.String(), new.String(), "token transfer")
			break
		}
	}
}

func (t *addressTracer) writeRecord(addr common.Address, currency, previous, new, reason string) {
	t.writer[addr].Write([]string{
		t.currentHash.Hex(),
		t.currentSender.Hex(),
		currency,
		previous,
		new,
		reason,
	})
	t.writer[addr].Flush()
}
```