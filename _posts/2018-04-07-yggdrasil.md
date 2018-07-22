## Yggdrasil - Block storage(Block chain) library

It-chain-Engine의 아키텍처를 수정하면서 기존 코드의 몇 부분을 라이브러리화 하기로 결정하고 각각을 서브 프로젝트(Yggdrasill, Heimdal, Teserract, Bifrost)로 현재 개발중에 있다. 각 프로젝트명은 북유럽신화에서 가져왔으며, 이번 포스팅에서 설명할 Yggdrasil([노르드 신화](https://ko.wikipedia.org/wiki/%EB%85%B8%EB%A5%B4%EB%93%9C_%EC%8B%A0%ED%99%94)의 중심을 이루는 [세계수](https://ko.wikipedia.org/wiki/%EC%84%B8%EA%B3%84%EC%88%98)로, 아홉 개의 세계를 연결하는 존재)는 블록 저장 라이브러리 이다.

**블록체인 네트워크의 노드들로 구성된 세계**가 있다면 그 노드들을 제일 밑에서 받쳐주는 블록체인이 마치 북유럽 신화의 세계수와 비슷하다는 생각이 들어 블록체인에 블록을 저장하는 라이브러리 이름을 Yggdrasill로 정하게 되었다.

![스크린샷 2018-03-11 오후 8.35.42](https://www.dropbox.com/s/by221xxm15hnory/yggdrasil.jpeg?dl=1)

Yggdrasill은 사용자의 커스텀 Block 및 Transaction을 지원하는 Block 저장소(Blockchain) 라이브러리이다. 원하는 커스텀 Block struct를 정의하고 Yggdrasill의 Block interface를 구현하면, 어떠한 커스텀 Block도 Blockchain의 형태로 저장, 검증 할 수 있다.



## Block Interface

```Go
//interface에 맞춰 설계
//interface를 implement하는 모든 custom block을 사용 가능하게 구현.
type Block interface{
	//transaction 저장
	PutTransaction(transaction tx.Transaction)
	
	//hash로 transaction 찾기
	FindTransactionIndexByHash(txHash string)
	
	//db에 저장할 형태
	Serialize() ([]byte, error)
	
	//Block hash생성
	GenerateHash() error
	
	//Block hash조회 
	GetHash() string
	
	//transaction 가져오기
	GetTransactions() []tx.Transaction
	
	//block 높이
	GetHeight() uint64
	
	//인자로 받은 []byte가 이전 Block인지 검증하는 로직
	IsPrev(serializedBlock []byte) bool
}
```

Yggdrasill의 Block interface는 다음과 같다. Block interface를 구현하는 어떠한 Block을 모두 Blockchain의 형태로 저장가능하다. 블록이 DB 혹은 File에 저장되기 위해서는 결국 []byte로 변환되어야 하는데 아래의 

```Go
Serialize() ([]byte, error)
```

함수를 직접 구현해서 Block을 어떻게 []byte로 변환할지를 직접 구현하면 된다.



## Transaction Interface

```
type Transaction interface{
	Serialize() ([]byte, error)
	GetID() string
	GetHash() string
}
```

Yggdrasill의 Transaction interface도 Block interface와 같다. 



## Default Block

```GO
type DefaultBlock struct {
	Header       BlockHeader
	MerkleTree   [][]string
	Transactions []tx.Transaction
}

type BlockHeader struct {
	Height             uint64
	PreviousHash       string
	Version            string
	MerkleTreeRootHash string
	TimeStamp          time.Time
	CreatorID          string
	Signature          []byte
	BlockHash          string
	MerkleTreeHeight   int
	TransactionCount   int
}

func (block DefaultBlock) IsPrev(serializedBlock []byte) bool {
	lastBlock := &DefaultBlock{}
	err := util.Deserialize(serializedBlock, lastBlock)

	if err != nil {
		return false
	}

	if (block.GetHeight() == lastBlock.GetHeight()+1) && (lastBlock.GetHash() == block.Header.PreviousHash) {
		return true
	}
	return false
}

```

Yggdrasill에서 정의한 interface를 만족하는 모든 Block을 저장할 수 있지만, Default Block구조체를 제공하고 있다. 위의 그림은 기본으로 제공되는 Block구조체와 함수의 모습이며, 예시로 설명하기 위해 IsPrev함수만 남겨 두었다. IsPrev함수는 Blockchain의 LastBlock을 받아서 이 Block이 LastBlock인지 검사하는 함수이다. 이 함수는 꼭 Block을 정의한 쪽에서 구현을 해주어야 하는데, Yggdrasill은 Block의 구조를 모르기 때문에 DB에서 마지막 Block을 가져오더라도 []byte로만 가져올 수 있기 때문에 height나 hash같은 값들을 가져올 수 없기 때문에 Block을 정의하는 사용자 입장에서 Block을 Deserialize해서 검사해주는 로직을 정의해 주어야 한다.



## Yggdrasill AddBlock Function

```Go
func (y *Yggdrasil) AddBlock(block block.Block) error {
	utilDB := y.DBProvider.GetDBHandle(UTIL_DB)
	lastBlock, err := utilDB.Get([]byte(LAST_BLOCK_KEY))

	if err != nil {
		return err
	}

	if lastBlock != nil && !block.IsPrev(lastBlock) {
		return NewBlockError("height or prevHash is not matched")
	}

	serializedBlock, err := block.Serialize()
	if err != nil {
		return err
	}

	blockHashDB := y.DBProvider.GetDBHandle(BLOCK_HASH_DB)
	blockNumberDB := y.DBProvider.GetDBHandle(BLOCK_NUMBER_DB)
	transactionDB := y.DBProvider.GetDBHandle(TRANSACTION_DB)

	err = blockHashDB.Put([]byte(block.GetHash()), serializedBlock, true)
	if err != nil {
		return err
	}

	err = blockNumberDB.Put([]byte(fmt.Sprint(block.GetHeight())), []byte(block.GetHash()), true)
	if err != nil {
		return err
	}

	err = utilDB.Put([]byte(LAST_BLOCK_KEY), serializedBlock, true)
	if err != nil {
		return err
	}

	for _, tx := range block.GetTransactions() {
		serializedTX, err := tx.Serialize()
		if err != nil {
			return err
		}

		err = transactionDB.Put([]byte(tx.GetID()), serializedTX, true)
		if err != nil {
			return err
		}

		err = utilDB.Put([]byte(tx.GetID()), []byte(block.GetHash()), true)
		if err != nil {
			return err
		}
	}

	return nil
}
```

위 함수는 Blockchain에 Block을 끼우는 AddBlock함수이다. Block이 들어오면 LastBlock을 이용하여 지금 들어온 Block이 LastBlock 다음에 올 수 있는지를 IsPrev를 통해 검사하고 KeyValueDB에 hash를 키로 serialized를 value로 저장한다. 함수를 보면 Block, Transaction interface에만 의존하고 있기 때문에 어떠한 Custom Block이 들어오더라도 Interface만 구현되어 있으면 저장 할 수 있다. 



## Validator

블록의 검증을 위해서 Validator를 사용하는데 Validator또한 위와 마찬가지로 공통 Interface를 정의하고 다양한 Validator를 사용할 수 있도록 하고자 한다. MerkleTree, PatriciaTree 등이 기본으로 제공될 Validator이다. 아직 Validator파트는 구현 중에 있으며 **Contribution**환영 합니다.



이번 포스팅에서는 it-chain-Engine에서 사용하는 Blockchain 라이브러리에 대해서 알아보았다. 내용을 요약하자면, Yggdrasill은 Block, Transaction의 interface를 통해 누구나 자신만의 Block, Transaction을 Blockchain형태로 저장하고 검증할 수 있다. 

Github주소: https://github.com/it-chain/yggdrasill