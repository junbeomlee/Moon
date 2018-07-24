

# Script

이번 포스팅에서는 Bitcoin에서 사용자들간의 송금(거래, Transaction)을 유효화 시키는 script에 대해서 소개해 드리겠습니다. 이번 포스팅에서는 Bitcoin에 대한 기본적인 이해를 바탕으로 하기 때문에 처음보는 분이 계시면 조금 어려운 내용일 될 수도 있습니다.

모든 bitcoin은 `locking script`라는 것으로 잠겨져 있고, 이 잠긴 `locking script`를 풀수있는 `unlocking script`를 만들수 있는 자만이 해당 bitcoin을 소비할 수 있습니다. 좀더 자세한 설명을 위해 실제 bitcoin의 transaction을 보면서 설명해드리겠습니다. 아래는 bitcoin의 실제 transaction의 json format입니다. 해당 transaction은 1개의 input과 2개의 out을 가지고 있습니다. 이는 해당 bitcoin을 2명에게 송금했다고 보시면 됩니다. inputs에는 script라고 써있는 부분이 2개 있습니다. input의 prev_out의 script가 `locking script` input의 script가 `unlocking script`이며, out의 script는 `locking script` 입니다.

```
{
   "ver":1,
   "inputs":[
      {
         "sequence":4294967294,
         "witness":"",
         "prev_out":{
            "spent":true,
            "tx_index":269632701,
            "type":0,
            "addr":"1GLctvTi81GDYZF5F6nif2MdbxUnAGHATZ",
            "value":4920000,
            "n":5,
            "script":"76a914a83fc0fb6edecb5289ef3e333520e4235bdf8a8788ac"
         },
         "script":"473044022036be6403aeb4e0e6fd54720b328d9d81bea32fb79684da0288743668fb5ef3ee02202023a71ef7217061fb9b4f35a05143de71447032e5a35b39c3d14b3210bad10b0121032725846bb7bc2e47b7b5a50670d77c8268f4d7f3243bdcf1b22174a67faaf528"
      }
   ],
   "weight":900,
   "block_height":477230,
   "relayed_by":"178.79.179.49",
   "out":[
      {
         "spent":true,
         "tx_index":270018614,
         "type":0,
         "addr":"1AG24pctoCpEMfSvEKfUZqaGYnz8ey8BfF",
         "value":3744000,
         "n":0,
         "script":"76a914659042e01e864e2f29641ea3a213c51a956d33c788ac"
      },
      {
         "spent":true,
         "tx_index":270018614,
         "type":0,
         "addr":"18Gj3unkApi17yh24JF6WwhN635SfABFMj",
         "value":1018920,
         "n":1,
         "script":"76a9144fc238bcda3f884ff6ce8d9feeb89b50dfd3da8888ac"
      }
   ],
   "lock_time":477228,
   "size":225,
   "double_spend":false,
   "time":1500839662,
   "tx_index":270018614,
   "vin_sz":1,
   "hash":"b657e22827039461a9493ede7bdf55b01579254c1630b0bfc9185ec564fc05ab",
   "vout_sz":2
}
```

`prev_out` 의  `locking script(76a914a83fc0fb6edecb5289ef3e333520e4235bdf8a8788ac)` 를 `unlocking script(473044022036be6403aeb4e0e6fd54720b328d9d81bea32fb79684da0288743668fb5ef3ee02202023a71ef 7217061fb9b4f35a05143de71447032e5a35b39c3d14b3210bad10b0121032725846bb7bc2e47b7b5a50670d77c826 8f4d7f3243bdcf1b22174a67faaf528)` 로 해제하고 2개의 out을 만들어내어 각각을 `locking script`로 잠궜습니다. 즉 transaction이란 나만 풀수있는 bitcoin을 `unlocking`하고 송금받는 사람이 풀수 있도록 `locking` 하는 것입니다. locking script와 unlocking script를 함께 연산해서 사용하였을 때 결과가 'TRUE'값이 나오면 unlocking script가 유효하다는 것이고 해당 bitcoin을 소비할 수 있습니다.



## 스크립트는 어떻게 동작할까?

locking, unlocking script는 스택이라는 데이터 구조를 사용하여 연산을 수행합니다. 스택은 한쪽 끝에서만 자료를 넣거나 뺄 수 있는 선형 구조(LIFO - Last In First Out)으로 되어있습니다. 자료를 넣는 것을 Push라고하고 반대로 넣어둔 자료를 꺼내는 것을 팝(pop)이라고 하는데, 이 때 꺼내지는 자료를 가장 최근에 보관한 자료부터 나오게 됩니다. (https://ko.wikipedia.org/wiki/%EC%8A%A4%ED%83%9D) 스택을 사용하기전에 우선 전처리를 통해 script를 Operator와 Data로 변환해야합니다.

 `unlocking script` 와 `locking script` 를 concat한 다음에

```
473044022036be6403aeb4e0e6fd54720b328d9d81bea32fb79684da0288743668fb5ef3ee02202023a71ef7217061fb9b4f35a05143de71447032e5a35b39c3d14b3210bad10b0121032725846bb7bc2e47b7b5a50670d77c8268f4d7f3243bdcf1b22174a67faaf52876a914659042e01e864e2f29641ea3a213c51a956d33c788acd
```

 앞에서 부터 byte단위로 읽으면서 Operator와 Data로 변환합니다. byte단위로 읽으면서 해당 값이 Opcode인지 아니면 Data를 나타내는 Code인지를 확인하며 변환홥니다. 변환 규칙과 Opcode는 https://en.bitcoin.it/wiki/Script 에서 확인 할 수 있습이다.

```
unlocking script: 3044022036be6403aeb4e0e6fd54720b328d9d81bea32fb79684da0288743668fb5ef3ee02202023a71ef7217061fb9b4f35a05143de71447032e5a35b39c3d14b3210bad10b01 032725846bb7bc2e47b7b5a50670d77c8268f4d7f3243bdcf1b22174a67faaf528
```

```
unlocking script: SIG PUBKEY
```

```
locking script: 76a914659042e01e864e2f29641ea3a213c51a956d33c788acd
```

```
locking scrit: OP_DUP OP_HASH160 a83fc0fb6edecb5289ef3e333520e4235bdf8a87(PUBKEY_HASH) OP_EQUALVERIFY OP_CHECKSIG
```

최종 형태는 `SIG PUBKEY OP_DUP OP_HASH160 PUBKEY_HASH OP_EQUALVERIFY OP_CHECKSIG`  입니다.

이제 스택을 활용하여 연산을 수행합니다.




| Stack                                | Script                                                       | Description                                                  |
| :------------------------------------: | :------------------------------------------------------------: | :------------------------------------------------------------: |
| `SIG`                                | ` PUBKEY OP_DUP OP_HASH160 PUBKEY_HASH OP_EQUALVERIFY OP_CHECKSIG` | `SIG`를 stack에 푸쉬한다. (OPCode)가 아니면 모두 stack push한다. |
| `SIG PUBKEY`                         | `OP_DUP OP_HASH160 PUBKEY_HASH OP_EQUALVERIFY OP_CHECKSIG`   | `PUBKEY` 를 stack에 푸쉬한다.                                |
| `SIG PUBKEY PUBKEY`                  | `OP_HASH160 PUBKEY_HASH OP_EQUALVERIFY OP_CHECKSIG`          | `OP_DUP` 는 stack에서 pop한후에 복사해서 둘다 push한다.      |
| `SIG PUBKEY PUBKEY_HASH`             | `PUBKEY_HASH OP_EQUALVERIFY OP_CHECKSIG`                     | `OP_HASH160` 는 stack에서 pop한 후 hash160를 수행한후 push 한다. `PUBKEY` 를 hash160하면 `PUBKEY_HASH`가 나온다. |
| `SIG PUBKEY PUBKEY_HASH PUBKEY_HASH` | `OP_EQUALVERIFY OP_CHECKSIG`                                 | `PUBKEY_HASH` 를 stack에 푸쉬한다.                           |
| `SIG PUBKEY`                         | ` OP_CHECKSIG`                                               | `OP_EQUALVERIFY` stack에서 2개를 pop개 같은지 비교한다. 같으면 아무것도수 행하지 않는다. |
| `TRUE`                               |                                                              | ` OP_CHECKSIG` 는 `SIG` 가 `PUBKEY` 에 대응하는 `PRIKEY` 로 서명됬는지 확인하고 맞으면 `TRUE` 푸쉬한다. |




모든 Script operation을 수행 한 후에 마지막에 stack에 남은 OPCode가 `TRUE` 일 경우 해당 bitcoin은 `unlocking script`로 해제 가능함을 의미합니다.



## Example Code

위의 operation을 go코드로 작성해 보았습니다. https://github.com/junbeomlee/vm

```go
package vm

import (
	"encoding/hex"
	"errors"
	"fmt"
)

const NONE ScriptType = "NONE"
const P2PK ScriptType = "P2PK"
const P2SH ScriptType = "P2SH"

type ScriptType string

func Run(lockingScript string, unlockingScript string, txHash []byte) (Stack, error) {

	scriptType := CheckScriptType(lockingScript)

	if scriptType == NONE {
		return Stack{}, errors.New("invalid script type")
	}

	if scriptType == P2SH {
		fmt.Println("need to do p2sh")
	}

	fmt.Printf("Script Type is [%s] \n", scriptType)

	stack := NewStack()

	ls := ParseScript(lockingScript)
	us := ParseScript(unlockingScript)

	script := append(ls, us...)

	for _, h := range script {

		opCode, ok := h.(Opcode)

		if ok {
			opCode.Do(&stack, txHash)
			continue
		}

		stack.Push(h)
	}

	return stack, nil
}
```

위 코드는 locking script와 unlocking script를 받아 hex로 변환 후에 byte단위로 읽으면서 Opcode와 data로 parsing을 수행하고 for문을 돌며서 Opcode면 해당 operation을 수행, data일 경우 stack에 push하고, stack을 return한다. 이때 stack의 가장 위의 element가 TRUE일경우 unlocking script로 locking script가 해제 가능한 것입니다.

```go

package vm

import (
	"bytes"
	"crypto/sha256"
	"errors"

	"github.com/junbeomlee/vm/ecdsa"
	"golang.org/x/crypto/ripemd160"
)

// Constant
const OP_PUSHDATA uint8 = 0x4c
const OP_TRUE uint8 = 0x51

// Stack
const OP_DUP uint8 = 0x76

// Bitwise logic
const OP_EQUAL uint8 = 0x87
const OP_EQUAL_VERIFY uint8 = 0x88

// Crypto
const OP_CHECK_SIG uint8 = 0xac
const OP_HASH_160 uint8 = 0xa9
const OP_CHECKMULTI_SIG uint8 = 0xae

// can convert data to []uint8 type
type Hexable interface {
	Hex() []uint8
}

type Data struct {
	Body []uint8
}

func (d Data) Hex() []uint8 {
	return d.Body
}

// do something with stack
type Operator interface {
	Do(stack *Stack, txHash []byte) error
}

// opcode can do something with stack and convert data to []uint8 type
type Opcode interface {
	Hexable
	Operator
}

var Opcodes = make(map[uint8]Opcode, 0)

// init all opcodes
func init() {
	Opcodes[PushDataOp{}.Hex()[0]] = PushDataOp{}
	Opcodes[DupOp{}.Hex()[0]] = DupOp{}
	Opcodes[EqualOp{}.Hex()[0]] = EqualOp{}
	Opcodes[EqualVerifyOp{}.Hex()[0]] = EqualVerifyOp{}
	Opcodes[CheckSigOp{}.Hex()[0]] = CheckSigOp{}
	Opcodes[Hash160Op{}.Hex()[0]] = Hash160Op{}
	Opcodes[CheckMultiSigOp{}.Hex()[0]] = CheckMultiSigOp{}
}

// The variables below refer to the bitcoin opcode [https://en.bitcoin.it/wiki/Script]

// Constant

// desc : The next byte contains the number of bytes to be pushed onto the stack.
type PushDataOp struct {
}

func (PushDataOp) Hex() []uint8 {
	return []uint8{OP_PUSHDATA}
}

func (PushDataOp) Do(stack *Stack, txHash []byte) error {
	//do nothing
	return nil
}

type DupOp struct{}

func (DupOp) Hex() []uint8 {
	return []uint8{OP_DUP}
}

// pop first element and push twice
func (DupOp) Do(stack *Stack, txHash []byte) error {

	h, err := stack.Pop()

	if err != nil {
		return err
	}

	stack.Push(h)
	stack.Push(h)

	return nil
}

// Bitwise logic
type EqualOp struct{}

func (o EqualOp) Hex() []uint8 {
	return []uint8{OP_EQUAL}
}

func (o EqualOp) Do(stack *Stack, txHash []byte) error {

	h1, err := stack.Pop()

	if err != nil {
		return err
	}

	h2, err := stack.Pop()

	if err != nil {
		return err
	}

	if !bytes.Equal(h1.Hex(), h2.Hex()) {
		return errors.New("not equal")
	}

	stack.Push(Data{Body: []uint8{OP_TRUE}})

	return nil
}

type EqualVerifyOp struct{}

func (o EqualVerifyOp) Hex() []uint8 {
	return []uint8{OP_EQUAL_VERIFY}
}

func (o EqualVerifyOp) Do(stack *Stack, txHash []byte) error {

	h1, err := stack.Pop()

	if err != nil {
		return err
	}

	h2, err := stack.Pop()

	if err != nil {
		return err
	}

	if !bytes.Equal(h1.Hex(), h2.Hex()) {
		return errors.New("not equal")
	}

	return nil
}

// Crypto
//
type CheckSigOp struct {
}

func (CheckSigOp) Hex() []uint8 {
	return []uint8{OP_CHECK_SIG}
}

func (CheckSigOp) Do(stack *Stack, txHash []byte) error {

	p, err := stack.Pop()

	if err != nil {
		return err
	}

	pubKey, err := ecdsa.Decode(p.Hex())

	if err != nil {
		return err
	}

	sig, err := stack.Pop()

	if err != nil {
		return err
	}

	valid, err := ecdsa.Verify(pubKey, sig.Hex(), txHash)

	if err != nil {
		return err
	}

	if !valid {
		stack.Push(Data{Body: []uint8{OP_TRUE}})
		return nil
	}

	stack.Push(Data{Body: []uint8{OP_TRUE}})

	return nil
}

// OP_HASH_160
type Hash160Op struct {
}

func (Hash160Op) Hex() []uint8 {
	return []uint8{OP_HASH_160}
}

func (Hash160Op) Do(stack *Stack, txHash []byte) error {

	h1, err := stack.Pop()

	if err != nil {
		return err
	}

	s := sha256.New()
	s.Write(h1.Hex())
	bs := s.Sum(nil)

	r := ripemd160.New()
	r.Write(bs)
	hashed := r.Sum(nil)

	stack.Push(Data{Body: hashed})

	return nil
}

//OP_CHECK_MULTI_SIG_256
type CheckMultiSigOp struct {
}

func (CheckMultiSigOp) Hex() []uint8 {
	return []uint8{OP_CHECKMULTI_SIG}
}

func (CheckMultiSigOp) Do(stack *Stack, txHash []byte) error {
	panic("not implemented")
	return nil
}

func GetOpCode(u uint8) Opcode {
	return Opcodes[u]
}
```

위 코드는 해당 OPcode에 대한 operation들을 구현한 코드 입니다. bitcoin script에 정의된 opcode는 더 많지만 그중에 중요한 몇가지 opcode에 대해서만 구현하였습니다.