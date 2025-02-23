# Conflux-Abigen
Conflux-Abigen is migrated from Ethereum tool Abigen. 

This page introduces the concept of server side native Dapps: Go language bindings to any Conflux contract that is compile time type safe, highly performant and best of all, can be generated fully automatically from a contract ABI and optionally the CVM bytecode.

## How to use

### Install
download the code
```
$ git clone https://github.com/Conflux-Chain/conflux-abigen.git
```

then install
```
$ go install ./cmd/cfxabigen
```

The exectuion file named `cfxabigen` will be installed to `$GOPATH/bin/cfxabigen`

### Generating the bindings

The single essential thing needed to generate a Go binding to an Conflux contract is the
contract's ABI definition `JSON` file. For our [`Token`](./example/token/token.sol) contract tutorial you can obtain this
by compiling the Solidity code yourself

To generate a binding, simply call:

```
$ cfxabigen --abi token.abi --pkg main --type Token --out token.go
```
when you need deploy it, you should generate a binding with bytecoe:
```
$ cfxabigen --abi token.abi --bin token.bin --pkg main --type Token --out token.go
```
but the simplest way is directly generate by solidity file:
```
$ cfxabigen --sol token.sol --pkg main --out token.go
```

Where the flags are:

 * `--abi`: Mandatory path to the contract ABI to bind to
 * `--bin`: Path to the Conflux contract bytecode (generate deploy method)
 * `--sol`: Path to the Conflux contract Solidity source to build and bind
 * `--solc`: Solidity compiler to use if source builds are requested (default: "solc")
 * `--type`: Optional Go type name to assign to the binding struct
 * `--pkg`: Mandatory Go package name to place the Go code into
 * `--out`: Optional output path for the generated Go source file (not set = stdout)
 
This will generate a type-safe Go binding for the Token contract. The generated code will
look something like [`token.go`](https://gist.github.com/karalabe/5839509295afa4f7e2215bc4116c7a8f),

### Prepare
Interact with contract using binding, you need use a backend which implements functions includes rpc invoking and transaction signing, generally creat a new sdk.Client instance.

### Deploy contract
Interacting with existing contracts is nice, but let’s take it up a notch and deploy a brand new contract onto the Conflux blockchain! To do so however, the contract ABI we used to generate the binding is not enough. We need the compiled bytecode too to allow deploying it.

```
$ cfxabigen --abi token.abi  --bin token.bin --pkg main --out token.go
```
This will generate something similar to token.go. If you quickly skim this file, you’ll find an extra `DeployToken` function that was just injected compared to the previous code. Beside all the parameters specified by Solidity, it also needs the usual authorization options to deploy the contract with and the Conflux backend to deploy the contract through.

```golang
package main

import (
	"fmt"
	"log"
	"math/big"
	"time"

	"abigen/bind"

	sdk "github.com/Conflux-Chain/go-conflux-sdk"
	"github.com/Conflux-Chain/go-conflux-sdk/types"
	"github.com/Conflux-Chain/go-conflux-sdk/types/cfxaddress"
	"github.com/sirupsen/logrus"
)

func main() {
	client, err := sdk.NewClient("ws://test.confluxrpc.com/ws", sdk.ClientOption{
		KeystorePath: "../keystore",
	})
	if err != nil {
		log.Fatal(err)
	}

	err = client.AccountManager.UnlockDefault("hello")
	if err != nil {
		log.Fatal(err)
	}

	tx, hash, _, err := DeployMyToken(nil, client, big.NewInt(10000), "ABC", 18, "ABC")
	if err != nil {
		panic(err)
	}

	receipt, err := client.WaitForTransationReceipt(*hash, time.Second)
	if err != nil {
		panic(err)
	}

	logrus.WithFields(logrus.Fields{
		"tx":               tx,
		"hash":             hash,
		"contract address": receipt.ContractCreated,
	}).Info("deploy token done")
```

### Accessing an Conflux contract
To interact with a contract deployed on the blockchain, you'll need to know the `address`
of the contract itself, and need to specify a `backend` through which to access Conflux.

```golang
package main

import (
	"fmt"
	"log"
	"math/big"
	"time"

	"abigen/bind"

	sdk "github.com/Conflux-Chain/go-conflux-sdk"
	"github.com/Conflux-Chain/go-conflux-sdk/types"
	"github.com/Conflux-Chain/go-conflux-sdk/types/cfxaddress"
	"github.com/sirupsen/logrus"
)

func main() {
	client, err := sdk.NewClient("ws://test.confluxrpc.com/ws", sdk.ClientOption{
		KeystorePath: "../keystore",
	})
	if err != nil {
		panic(err)
	}

	err = client.AccountManager.UnlockDefault("hello")
	if err != nil {
		panic(err)
	}

	contractAddr := cfxaddress.MustNew("cfxtest:acd7apn6pnfhna7w1pa8evzhwhv3085vjjp1b8bav5")
	instance, err := NewMyToken(contractAddr, client)
	if err != nil {
		panic(err)
	}

	err = client.AccountManager.UnlockDefault("hello")
	if err != nil {
		panic(err)
	}

	user := cfxaddress.MustNew("cfxtest:aasfup1wgjyxkzy3575cbnn87xj5tam2zud125twew")
	result, err := instance.BalanceOf(nil, user.MustGetCommonAddress())
	if err != nil {
		panic(err)
	}

	logrus.WithField("balance", result).WithField("user", user).Info("access contract") // "bar"
}
```

### Transacting with an Conflux contract
```golang
package main

import (
	"fmt"
	"log"
	"math/big"
	"time"

	"abigen/bind"

	sdk "github.com/Conflux-Chain/go-conflux-sdk"
	"github.com/Conflux-Chain/go-conflux-sdk/types"
	"github.com/Conflux-Chain/go-conflux-sdk/types/cfxaddress"
	"github.com/sirupsen/logrus"
)

func main() {
	client, err := sdk.NewClient("ws://test.confluxrpc.com/ws", sdk.ClientOption{
		KeystorePath: "../keystore",
	})
	if err != nil {
		panic(err)
	}

	err = client.AccountManager.UnlockDefault("hello")
	if err != nil {
		panic(err)
	}

	contractAddr := cfxaddress.MustNew("cfxtest:acd7apn6pnfhna7w1pa8evzhwhv3085vjjp1b8bav5")
	instance, err := NewMyToken(contractAddr, client)
	if err != nil {
		panic(err)
	}

	err = client.AccountManager.UnlockDefault("hello")
	if err != nil {
		panic(err)
	}

	to := cfxaddress.MustNew("cfxtest:aasfup1wgjyxkzy3575cbnn87xj5tam2zud125twew")
	tx, hash, err := instance.Transfer(nil, to.MustGetCommonAddress(), big.NewInt(1))
	if err != nil {
		panic(err)
	}

	logrus.WithField("tx", tx).WithField("hash", hash).Info("transfer") 
	receipt, err := client.WaitForTransationReceipt(*hash, time.Second)
	if err != nil {
		panic(err)
	}

	logrus.WithField("transfer receipt", receipt).Info()
}
```
### Watch event
```golang
package main

import (
	"log"
	"math/big"
	"time"

	"abigen/bind"

	sdk "github.com/Conflux-Chain/go-conflux-sdk"
	"github.com/Conflux-Chain/go-conflux-sdk/middleware"
	"github.com/Conflux-Chain/go-conflux-sdk/types"
	"github.com/Conflux-Chain/go-conflux-sdk/types/cfxaddress"
	"github.com/ethereum/go-ethereum/common"
	"github.com/sirupsen/logrus"
)

func main() {
	client, err := sdk.NewClient("ws://test.confluxrpc.com/ws", sdk.ClientOption{
		KeystorePath: "../keystore",
	})
	if err != nil {
		panic(err)
	}

	err = client.AccountManager.UnlockDefault("hello")
	if err != nil {
		panic(err)
	}

	contractAddr := cfxaddress.MustNew("cfxtest:acd7apn6pnfhna7w1pa8evzhwhv3085vjjp1b8bav5")
	instance, err := NewMyToken(contractAddr, client)
	if err != nil {
		panic(err)
	}

	eventCh := make(chan *MyTokenTransfer, 100)
	reorgCh := make(chan types.ChainReorg, 100)
	sub, err := instance.WatchTransfer(nil, eventCh, reorgCh, nil, nil)
	if err != nil {
		panic(err)
	}

	for {
		select {
		case l, ok := <-eventCh:
			logrus.WithFields(logrus.Fields{
				"log": l,
				"ok":  ok,
			}).Info("receive setted log")

		case r, ok := <-reorgCh:
			logrus.WithFields(logrus.Fields{
				"reorg": r,
				"ok":    ok,
			}).Info("receive setted log")

		case err := <-sub.Err():
			panic(err)
		}
	}
}
```
### Filter event
```golang
package main

import (
	"log"
	"math/big"
	"time"

	"abigen/bind"

	sdk "github.com/Conflux-Chain/go-conflux-sdk"
	"github.com/Conflux-Chain/go-conflux-sdk/middleware"
	"github.com/Conflux-Chain/go-conflux-sdk/types"
	"github.com/Conflux-Chain/go-conflux-sdk/types/cfxaddress"
	"github.com/ethereum/go-ethereum/common"
	"github.com/sirupsen/logrus"
)

func main() {
	client, err := sdk.NewClient("ws://test.confluxrpc.com/ws", sdk.ClientOption{
		KeystorePath: "../keystore",
	})
	if err != nil {
		panic(err)
	}
	client.UseCallRpcMiddleware(middleware.CallRpcConsoleMiddleware)

	err = client.AccountManager.UnlockDefault("hello")
	if err != nil {
		panic(err)
	}

	contractAddr := cfxaddress.MustNew("cfxtest:acd7apn6pnfhna7w1pa8evzhwhv3085vjjp1b8bav5")
	instance, err := NewMyToken(contractAddr, client)
	if err != nil {
		panic(err)
	}

	start := big.NewInt(35779622)
	end := big.NewInt(35779722)

	it, err := instance.FilterTransfer(&bind.FilterOpts{
		Start: types.NewEpochNumber(types.NewBigIntByRaw(start)),
		End:   types.NewEpochNumber(types.NewBigIntByRaw(end)),
	}, []common.Address{common.HexToAddress("0x1502ADd5a4a14c85C525e30a850c58fA15325f8C")}, nil,
	)

	if err != nil {
		panic(err)
	}

	for {
		if it.Next() {
			logrus.WithField("Transfer", it.Event).Info("Transfer log")
		} else {
			if err := it.Error(); err != nil {
				panic(err)
			}
			return
		}
	}
}
```
### Project integration (i.e. `go generate`)

The `abigen` command was made in such a way as to play beautifully together with existing
Go toolchains: instead of having to remember the exact command needed to bind an Conflux
contract into a Go project, we can leverage `go generate` to remember all the nitty-gritty
details.

Place the binding generation command into a Go source file before the package definition:

```
//go:generate abigen --sol token.sol --pkg main --out token.go
```
for example

```golang
//go:generate cfxabigen --sol token.sol --pkg main --out token.go
package main

import (
	"fmt"
	"log"
	"math/big"
	"time"

	"abigen/bind"

	sdk "github.com/Conflux-Chain/go-conflux-sdk"
	"github.com/Conflux-Chain/go-conflux-sdk/types"
	"github.com/Conflux-Chain/go-conflux-sdk/types/cfxaddress"
	"github.com/sirupsen/logrus"
)

func main() {
	...
}
```

After which whenever the Solidity contract is modified, instead of needing to remember and
run the above command, we can simply call `go generate` on the package (or even the entire
source tree via `go generate ./...`), and it will correctly generate the new bindings for us.


