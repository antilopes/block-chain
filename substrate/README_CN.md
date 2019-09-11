Substrate

----------

目录
1. 一句话介绍
2. Description
3. Usage
3.1. The Basics of Substrate
3.2. Extrinsics
3.3. Runtime and API
3.4. Inherent Extrinsics
3.5. Block-authoring Logic
4. Roadmap
4.1. So far
4.2. In progress
4.3. The future
5. Trying out Substrate Node
5.1. On Mac and Ubuntu
6. Building
6.1. Hacking on Substrate
6.2. WASM binaries
6.3. Joining the Flaming Fir Testnet
6.4. Joining the Emberic Elm Testnet
7. Key management
7.1. Recommended RPC call
7.2. Advanced RPC call
8. Documentation
8.1. Viewing documentation for Substrate packages
8.2. Contributing to documentation for Substrate packages
8.3. Contributing to documentation (tests, extended examples, macros) for Substrate packages
9. Contributing
9.1. Contributing Guidelines
9.2. Contributor Code of Conduct
10. License

------------

# 1.一句话介绍
Substrate 是下一代区块链创新框架。

# 2.更多说明
从本质上讲，Substrate是三种技术的结合：WebAssembly，Libp2p和GRANDPA共识。关于GRANDPA，请参阅此定义，介绍和正式规范。它既是用于构建一条新区块链的库，也是区块链客户端的“骨架密钥”，能够与任何基于Substrate的链同步。

Substrate具有使他能够称为“下一代区块链”的三个明显的特征：动态的、自定义状态转换功能，与生俱来的轻客户端功能，以及具有快速出块、自适应、自定义的不断进化的共识算法。在WebAssembly中编码的STF称为“runtime”。 这定义了execute_block函数，并且可以指定铆接算法，事务语义，日志记录机制和替换其自身或区块链状态（“治理”）的任何方面的所有内容。由于运行时完全是动态的，所以这些都可以随时切换或升级。 substrate非常像活的有机体。

想了解更多，去看：https://www.parity.io/what-is-substrate/

# 3.用法

Substrate仍然是一个早期阶段的项目，虽然它已经被用作Polkadot等重大项目的基础，但使用它仍然是一项重大挑战。 特别是，您应该对区块链概念和基本密码学有很好的了解。 Header，block，client，Hash，transaction和signature等术语应该是熟悉的。目前你需要一个Rust的工作知识才能做任何有趣的事情（尽管最终，我们的目标不是这样）。

Substrate有下面三种方式使用：  
简单：通过运行Substrate二进制可执行文件：**substrate**，并配置包含当前runtime的genesis区块。在这种情况下，你只需要build Substrate，配置JSON文件，然后启动你的区块链。这为您提供了最少量的可定制性，主要允许您更改各种包含的运行时模块的创建参数，例如余额，赌注，块期，费用和治理。  

模块化：通过将基板运行时模块库中的模块混合到一个新的运行时，并可能更改或重新配置Substrate客户端的块创作逻辑。这为您提供了超出自己的区块链逻辑的自由度，让您可以更改数据类型，添加或删除模块，最重要的是，添加自己的模块。在不触及块创作逻辑的情况下可以更改很多（因为它是通用的）。如果是这种情况，则现有的Substrate二进制文件可用于块创作和同步。如果需要调整块创作逻辑，则必须将新的更改块创作二进制文件构建为单独的项目并由验证器使用。这就是Polkadot继电器链的构建方式，应该足以满足近中期的几乎所有情况。

通用：可以忽略整个Substrate Runtime模块库，并从头开始设计和实现整个运行时。如果需要，可以使用Rust之外的语言完成此操作，前提是它可以以WebAssembly为目标。如果可以使运行时与现有客户端的块创作逻辑兼容，那么您可以简单地从Wasm blob构建一个新的创世块，并使用现有的基于Rust的Substrate客户端启动您的链。如果没有，那么您将需要相应地更改客户端的块创作逻辑。对于大多数项目来说，这可能是一个无用的选择，但提供了完全的灵活性，允许为Substrate范例提供长期深远的升级途径。

## 3.1.Substrate基础知识
Substrate是一个区块链平台，具有完全通用的状态转换功能。 也就是说，它确实带有关于底层数据结构的标准和约定（特别是关于运行时模块库）。 粗略地说，就惯例而言，这些核心数据类型在实际的非协商标准和通用+结构+ s方面对应于+ trait + s。

``` text
Header := Parent + ExtrinsicsRoot + StorageRoot + Digest
Block := Header + Extrinsics + Justifications
```

## 3.2.外部参数(Extrinsics)
Extrinsics in Substrate are pieces of information from "the outside world" that are contained in the blocks of the chain. You might think "ahh, that means transactions": in fact, no. Extrinsics fall into two broad categories of which only one is transactions. The other is known as inherents. The difference between these two is that transactions are signed and gossiped on the network and can be deemed useful per se. This fits the mold of what you would call transactions in Bitcoin or Ethereum.

Inherents, meanwhile, are not passed on the network and are not signed. They represent data which describes the environment but which cannot call upon anything to prove it such as a signature. Rather they are assumed to be "true" simply because a sufficiently large number of validators have agreed on them being reasonable.

To give an example, there is the timestamp inherent, which sets the current timestamp of the block. This is not a fixed part of Substrate, but does come as part of the Substrate Runtime Module Library to be used as desired. No signature could fundamentally prove that a block were authored at a given time in quite the same way that a signature can "prove" the desire to spend some particular funds. Rather, it is the business of each validator to ensure that they believe the timestamp is set to something reasonable before they agree that the block candidate is valid.

Other examples include the parachain-heads extrinsic in Polkadot and the "note-missed-proposal" extrinsic used in the Substrate Runtime Module Library to determine and punish or deactivate offline validators.

## 3.3.运行时和API
Substrate chains all have a runtime. The runtime is a WebAssembly "blob" that includes a number of entry-points. Some entry-points are required as part of the underlying Substrate specification. Others are merely convention and required for the default implementation of the Substrate client to be able to author blocks.

If you want to develop a chain with Substrate, you will need to implement the Core trait. This Core trait generates an API with the minimum necessary functionality to interact with your runtime. A special macro is provided called impl_runtime_apis! that help you implement runtime API traits. All runtime API trait implementations need to be done in one call of the impl_runtime_apis! macro. All parameters and return values need to implement parity-codec to be encodable and decodable.

Here’s a snippet of the Polkadot API implementation as of PoC-3:

``` rust
impl_runtime_apis! {
	impl client_api::Core<Block> for Runtime {
		fn version() -> RuntimeVersion {
			VERSION
		}

		fn execute_block(block: Block) {
			Executive::execute_block(block)
		}

		fn initialize_block(header: <Block as BlockT>::Header) {
			Executive::initialize_block(&header)
		}
	}
	// ---snip---
}
```

## 3.4. Inherent Extrinsics
The Substrate Runtime Module Library includes functionality for timestamps and slashing. If used, these rely on "trusted" external information being passed in via inherent extrinsics. The Substrate reference block authoring client software will expect to be able to call into the runtime API with collated data (in the case of the reference Substrate authoring client, this is merely the current timestamp and which nodes were offline) in order to return the appropriate extrinsics ready for inclusion. If new inherent extrinsic types and data are to be used in a modified runtime, then it is this function (and its argument type) that would change.

## 3.5. Block-authoring Logic
In Substrate, there is a major distinction between blockchain syncing and block authoring ("authoring" is a more general term for what is called "mining" in Bitcoin). The first case might be referred to as a "full node" (or "light node" - Substrate supports both): authoring necessarily requires a synced node and, therefore, all authoring clients must necessarily be able to synchronize. However, the reverse is not true. The primary functionality that authoring nodes have which is not in "sync nodes" is threefold: transaction queue logic, inherent transaction knowledge and BFT consensus logic. BFT consensus logic is provided as a core element of Substrate and can be ignored since it is only exposed in the SDK under the authorities() API entry.

Transaction queue logic in Substrate is designed to be as generic as possible, allowing a runtime to express which transactions are fit for inclusion in a block through the initialize_block and apply_extrinsic calls. However, more subtle aspects like prioritization and replacement policy must currently be expressed "hard coded" as part of the blockchain’s authoring code. That said, Substrate’s reference implementation for a transaction queue should be sufficient for an initial chain implementation.

Inherent extrinsic knowledge is again somewhat generic, and the actual construction of the extrinsics is, by convention, delegated to the "soft code" in the runtime. If ever there needs to be additional extrinsic information in the chain, then both the block authoring logic will need to be altered to provide it into the runtime and the runtime’s inherent_extrinsics call will need to use this extra information in order to construct any additional extrinsic transactions for inclusion in the block.

# 4. Roadmap

## 4.1. So far

* 0.1 "PoC-1": PBFT consensus, Wasm runtime engine, basic runtime modules.

* 0.2 "PoC-2": Libp2p

## 4.2. In progress

* AfG consensus

* Improved PoS

* Smart contract runtime module

## 4.3. The future

* Splitting out runtime modules into separate repo

* Introduce substrate executable (the skeleton-key runtime)

* Introduce basic but extensible transaction queue and block-builder and place them in the executable.

* DAO runtime module

* Audit

