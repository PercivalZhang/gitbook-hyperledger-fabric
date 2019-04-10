# chaincode开发者

## 什么是Chaincode

Chaincode是一个程序，用Go、node.js、Java实现的一套接口。Chaincode运行在一个安全的Docker容器中，与背书节点进程隔离。Chaincode通过应用提交的交易来初始化和管理账本的状态。

Chaincode专门用于处理一个网络内部成员达成一致认可的业务逻辑，因此它与smart contrac智能合约是非常相似的。在提案交易中，可以调用Chaincode来更新和查询账本。如果被授予适当的权限，两个Chaincode，无论在不在同一个channel通道内，都可以相互调用。需要注明的是：如果被调用的Chaincode与发起调用的Chaincode不在一个channel通道内，只允许调用只读Query查询操作。也就是说，被调用的Chaincode只是一个Query查询操作，并不参与调用方接下来提交阶段的状态验证。

接下来的部分，我们将从一个开发者的角度来探索Chaincode。我们将展示一个简单的chaincode示例程序，并且讲解Chaincode Shim API中每个方法的用途。

## Chaincode API

每一个chaincode程序都必须实现Chaincode接口，在接收到的交易中，将对程序的方法进行调用。你可以从如下的链接中找到该接口不同语言版本的参考文档：

- [go](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#Chaincode)
- [node.js](https://fabric-shim.github.io/ChaincodeInterface.html)
- [Java](https://fabric-chaincode-java.github.io/org/hyperledger/fabric/shim/Chaincode.html)

当chaincode收到instantiate或者upgrade交易时，Init方法将会被调用，chaincode可以执行包括应用状态初始化在内的任何初始化操作；当收到invoke交易时，Invoke方法将会被调用，chaincode可以处理交易提案。

在chaincode "shim" APIs 中，另外一个接口时ChaincodeStubInterface，接口不用语言版本的参考文档如下：

- [go](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#ChaincodeStubInterface)
- [node.js](https://fabric-shim.github.io/ChaincodeStub.html)
- [Java](https://fabric-chaincode-java.github.io/org/hyperledger/fabric/shim/ChaincodeStub.html)

该接口用于访问和修改账本，以及chaincode间的相互调用。

本教程使用GO语言编写Chaincode，通过实现一个管理简单资产的简单chaincode应用，来展示这些APIs的使用。

## 简单资产Chaincode

我们的应用是一个简单的示例chaincode，用于在账本上创建出键值对的资产。

### 选择代码位置

若以前没有做过Go语言编程，先确保安装Go语言环境，并且系统被正确配置。

首先，在$GOPATH/src下面，为chaincode应用创建一个目录。

为了更简单些，使用如下命令：

```
mkdir -p $GOPATH/src/sacc && cd $GOPATH/src/sacc
```

接下来，创建源文件：

```
touch sacc.go
```

### 准备工作

既然每个chaincode都要实现Chaincode接口，尤其是Init和Invoke方法，因此需要为我们的chaincode应用，添加必要的Go依赖引入声明。引入chaincode shim package和peer protobuf package。接下来，添加一个结构类型的变量SimpleAsset，作为Chaincode shime方法的接收者。

```go
package main

import (
    "fmt"

    "github.com/hyperledger/fabric/core/chaincode/shim"
    "github.com/hyperledger/fabric/protos/peer"
)

// SimpleAsset implements a simple chaincode to manage an asset
type SimpleAsset struct {
}
```

### Chaincode初始化

接下来，实现Init方法。

```go
// Init is called during chaincode instantiation to initialize any data.
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {

}
```

> 备注
>
> chaincode升级的时候，也会调用该方法。当需要编写chaincode，用于升级一个已经存在的chaincode时候，确保正确适当的修改Init方法。如果没有"迁移"或者其他作为升级的一部分需要初始化，提供一个空的Init方法即可。

下一步，我门将通过调用方法[ChaincodeStubInterface.GetStringArgs](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#ChaincodeStub.GetStringArgs)，来获取Init调用时，传入的参数。在我们的例子，我们期望时一个key-value键值对。

```go
// Init is called during chaincode instantiation to initialize any
// data. Note that chaincode upgrade also calls this function to reset
// or to migrate data, so be careful to avoid a scenario where you
// inadvertently clobber your ledger's data!
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {
  // Get the args from the transaction proposal
  args := stub.GetStringArgs()
  if len(args) != 2 {
    return shim.Error("Incorrect arguments. Expecting a key and a value")
  }
}
```

接下来，既然已经确认了调用有效，我们将把初始状态存入账本ledger。我们通过将键，值分别作为参数传入方法[ChaincodeStubInterface.PutState](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#ChaincodeStub.PutState)，并调用该方法，来实现状态写入。假定一切运行正常，该方法将返回一个peer.Response对象来表明初始化成功。

```go
// Init is called during chaincode instantiation to initialize any
// data. Note that chaincode upgrade also calls this function to reset
// or to migrate data, so be careful to avoid a scenario where you
// inadvertently clobber your ledger's data!
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {
  // Get the args from the transaction proposal
  args := stub.GetStringArgs()
  if len(args) != 2 {
    return shim.Error("Incorrect arguments. Expecting a key and a value")
  }

  // Set up any variables or assets here by calling stub.PutState()

  // We store the key and the value on the ledger
  err := stub.PutState(args[0], []byte(args[1]))
  if err != nil {
    return shim.Error(fmt.Sprintf("Failed to create asset: %s", args[0]))
  }
  return shim.Success(nil)
}
```

### Chaincode调用

首先，添加Invoke方法签名。

```go
// Invoke is called per transaction on the chaincode. Each transaction is
// either a 'get' or a 'set' on the asset created by Init function. The 'set'
// method may create a new asset by specifying a new key-value pair.
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {

}
```

跟前面的Init方法一样，我们依然需要从ChaincodeStubInterface中提取输入参数。Invoke方法的参数是要被调用的chaincode应用方法的名字。在我们这个例子中，我们的应用只有两个简单的方法：set和get，用于设置一个资产的值和获取资产当前的状态。第一步通过调用 [ChaincodeStubInterface.GetFunctionAndParameters](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#ChaincodeStub.GetFunctionAndParameters)，来提取chaincode应用方法名称以及参数。

```go
// Invoke is called per transaction on the chaincode. Each transaction is
// either a 'get' or a 'set' on the asset created by Init function. The Set
// method may create a new asset by specifying a new key-value pair.
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
    // Extract the function and args from the transaction proposal
    fn, args := stub.GetFunctionAndParameters()

}
```

接下来，我们要验证应用方法的名字是不是set或者get，并且调用那些chaincode应用方法，通过shim.Success 或者shim.Error将结果序列化为一个gRPC protobuf message，封装为Response对象返回。

```go
// Invoke is called per transaction on the chaincode. Each transaction is
// either a 'get' or a 'set' on the asset created by Init function. The Set
// method may create a new asset by specifying a new key-value pair.
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
    // Extract the function and args from the transaction proposal
    fn, args := stub.GetFunctionAndParameters()

    var result string
    var err error
    if fn == "set" {
            result, err = set(stub, args)
    } else {
            result, err = get(stub, args)
    }
    if err != nil {
            return shim.Error(err.Error())
    }

    // Return the result as success payload
    return shim.Success([]byte(result))
}
```

实现Chaincode应用

在前面，提到我们的chaincode应用实现了两个方法，这两个方法可以通过Invoke方法来调用。现在，就来实现这两个方法。注意，我们在先前提到过，为了访问账本的状态，必须利用chaincode shim API中的[ChaincodeStubInterface.PutState](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#ChaincodeStub.PutState)和[ChaincodeStubInterface.GetState](https://godoc.org/github.com/hyperledger/fabric/core/chaincode/shim#ChaincodeStub.GetState)两个方法。

```go
// Set stores the asset (both key and value) on the ledger. If the key exists,
// it will override the value with the new one
func set(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 2 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key and a value")
    }

    err := stub.PutState(args[0], []byte(args[1]))
    if err != nil {
            return "", fmt.Errorf("Failed to set asset: %s", args[0])
    }
    return args[1], nil
}

// Get returns the value of the specified asset key
func get(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 1 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key")
    }

    value, err := stub.GetState(args[0])
    if err != nil {
            return "", fmt.Errorf("Failed to get asset: %s with error: %s", args[0], err)
    }
    if value == nil {
            return "", fmt.Errorf("Asset not found: %s", args[0])
    }
    return string(value), nil
}
```

### 最终代码

最后，我们需要添加main入口函数，它将调用shim.Start方法。下面是该chaincode应用的完整源代码。

```go
package main

import (
    "fmt"

    "github.com/hyperledger/fabric/core/chaincode/shim"
    "github.com/hyperledger/fabric/protos/peer"
)

// SimpleAsset implements a simple chaincode to manage an asset
type SimpleAsset struct {
}

// Init is called during chaincode instantiation to initialize any
// data. Note that chaincode upgrade also calls this function to reset
// or to migrate data.
func (t *SimpleAsset) Init(stub shim.ChaincodeStubInterface) peer.Response {
    // Get the args from the transaction proposal
    args := stub.GetStringArgs()
    if len(args) != 2 {
            return shim.Error("Incorrect arguments. Expecting a key and a value")
    }

    // Set up any variables or assets here by calling stub.PutState()

    // We store the key and the value on the ledger
    err := stub.PutState(args[0], []byte(args[1]))
    if err != nil {
            return shim.Error(fmt.Sprintf("Failed to create asset: %s", args[0]))
    }
    return shim.Success(nil)
}

// Invoke is called per transaction on the chaincode. Each transaction is
// either a 'get' or a 'set' on the asset created by Init function. The Set
// method may create a new asset by specifying a new key-value pair.
func (t *SimpleAsset) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
    // Extract the function and args from the transaction proposal
    fn, args := stub.GetFunctionAndParameters()

    var result string
    var err error
    if fn == "set" {
            result, err = set(stub, args)
    } else { // assume 'get' even if fn is nil
            result, err = get(stub, args)
    }
    if err != nil {
            return shim.Error(err.Error())
    }

    // Return the result as success payload
    return shim.Success([]byte(result))
}

// Set stores the asset (both key and value) on the ledger. If the key exists,
// it will override the value with the new one
func set(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 2 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key and a value")
    }

    err := stub.PutState(args[0], []byte(args[1]))
    if err != nil {
            return "", fmt.Errorf("Failed to set asset: %s", args[0])
    }
    return args[1], nil
}

// Get returns the value of the specified asset key
func get(stub shim.ChaincodeStubInterface, args []string) (string, error) {
    if len(args) != 1 {
            return "", fmt.Errorf("Incorrect arguments. Expecting a key")
    }

    value, err := stub.GetState(args[0])
    if err != nil {
            return "", fmt.Errorf("Failed to get asset: %s with error: %s", args[0], err)
    }
    if value == nil {
            return "", fmt.Errorf("Asset not found: %s", args[0])
    }
    return string(value), nil
}

// main function starts up the chaincode in the container during instantiate
func main() {
    if err := shim.Start(new(SimpleAsset)); err != nil {
            fmt.Printf("Error starting SimpleAsset chaincode: %s", err)
    }
}
```

### 编译Chaincode

编译chaincode。

```shell
go get -u github.com/hyperledger/fabric/core/chaincode/shim
go build
```

假定没有错误，我们就能继续下一步，测试chaincode。

### 使用dev开发模式测试chaincode

正常流程，chaincode由peer节点来启动和维护。然而在开发模式下，chaincode由用户编译和启动。这个模式在chaincode开发阶段十分有用，有助于chaincode的迭代开发：快速编码、编译、运行、调试。

我们通过预先生成的orderer排序节点和通道部件，来开启一个开发模式下的开发网络。在这个网络里，用户可以马上进入到chaincode编译和调用驱动的环节。

### 安装超级账本Fabric范例

如果还没安装，请[安装范例代码, 二进制文以及Docker镜像](https://hyperledger-fabric.readthedocs.io/en/release-1.4/install.html).

进入`fabric-samples`下面的 `chaincode-docker-devmode`目录：

```shell
cd chaincode-docker-devmode
```

接下来开启三个命令行终端窗口，每一个都进入`chaincode-docker-devmode`目录.

### 终端1 - 启动网络

```shell
docker-compose -f docker-compose-simple.yaml up
```

上面的命令会启动一个网络，该网络配置为`SingleSampleMSPSolo` orderer profile ，并且以开发模式启动peer节点。同时还会启动两个其他的容器：一个用于chaincode环境；一个命令行，用于跟chaincode交互。创建和加入通道的命令内置在命令行容器中，因此可以立即进入chaincode的调用。

### 终端2 - 编译和启动chaincode

```shell
docker exec -it chaincode bash
```

应该看到如下的输出：

```shell
root@d2629980e76b:/opt/gopath/src/chaincode#
```

接下来编译chaincode：

```shell
cd sacc
go build
```

现在可以运行chaincode了：

```shell
CORE_PEER_ADDRESS=peer:7052 CORE_CHAINCODE_ID_NAME=mycc:0 ./sacc
```

由peer直接启动chaincode，通过日志可以看出chaincode由peer成功注册。注意：在这个阶段，chaincode还没有关联到任何通道。在接下来的步骤中，通过`instantiate`命令来实现这一点。

### 终端3 - 使用chaincode

即使在`--peer-chaincodedev`模式，依然需要先安装chaincode，只有这样，生命周期管理的系统chaincode才能正常执行例行的检查。这个需求依赖，可能在将来的开发模式中，被移除。

我们将利用命令行容器来驱动这些调用。

#### 启动命令行容器

```shell
docker exec -it cli bash
```

#### 安装和初始化chaincode

```shell
peer chaincode install -p chaincodedev/chaincode/sacc -n mycc -v 0
peer chaincode instantiate -n mycc -v 0 -c '{"Args":["a","10"]}' -C myc
```

现在可以调用invoke来改变变量a的值为20.

```shell
peer chaincode invoke -n mycc -c '{"Args":["set", "a", "20"]}' -C myc
```

最后通过query查询，可以获取变量a的值为20.

```shell
peer chaincode query -n mycc -c '{"Args":["query","a"]}' -C myc
```

### 测试其他的chaincode

默认，我们只挂载了sacc chaincode。然而，你可以通过将其他的chaincode应用添加到chaincode目录下，并且重启网络，就可以很容易的测试其他的chaincodes。在chaincode容器里就能访问这些chaincodes。

### Chaincode访问控制

Chaincode通过调用GetCreator()方法，利用客户端（提交者）的证书，来实现访问控制决策。另外Go shim提供了扩展APIs，通过该APIs可以从提交者的证书里面提取客户端身份，从而用于访问控制决策。

例如一个键值形式的资产，可以将客户身份识别信息作为值的一部分（例如可以通过一个JSON属性，来标明资产的主人）。只有身份匹配成功的客户端才可以更新键值。Chaincode可以调用客户端身份识别库扩展APIs，来获取提交者的信息，从而实现访问控制决策。

阅读[客户端身份识别库文档CID](https://github.com/hyperledger/fabric/blob/master/core/chaincode/shim/ext/cid/README.md)，获取更多细节。

关于如何将客户身份识别扩展，作为依赖添加到chaincode中，请参考[Go语言版chaincode外部依赖管理](https://hyperledger-fabric.readthedocs.io/en/release-1.4/chaincode4ade.html#vendoring)。

### Chaincode加密

特定场景下，针对一个键对应的值进行部分或者全部加密，十分有用。例如，如果一个人的社会保险号码或者家庭住址被写入账本，然后你很可能不希望这种信息以明文的方式出现。Chaincode加密是利用[entities extension](https://github.com/hyperledger/fabric/tree/master/core/chaincode/shim/ext/entities)来实现的。entities extension封装了商品工厂以及相应的方法，比如加密和椭圆曲线数字签名等，来执行密码学相关的操作。举个例子：为了加密，chaincode调用者必须通过临时字段，传入一个密码学的key，这个key可以用于后续的查询，以及对加密状态数据进行正确的解密。

更多的信息和例子，可以参考`fabric/examples`目录下的[Encc例子](https://github.com/hyperledger/fabric/tree/master/examples/chaincode/go/enccc_example)。特别要注意`utils.go`这个助手程序，它会加载chaincode shim APIs和Entities扩展，并且生成一类新的方法（例如：`encryptAndPutState`和`getStateAndDecrypt`），供chaincode加密示例代码调用。通过这种方式，chaincode就能赋予基础版shim APIs中的`Get`和`Put`以加密和解密的功能。