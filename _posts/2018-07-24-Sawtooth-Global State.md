# Global State

이번 포스팅에서는 Sawtooth의 `Global State`에 대해서 알아보겠습니다. Sawtooth의 기본적인 내용은 https://junbeomlee.github.io/Sawtooth/에서 내용을 확인할 수 있습니다!

Sawtooth의 smallback transaction family를 대상으로 설명드리겠습니다. transaction familiy란 client, state 그리고 transaction processor를 일컫는 말이고, 여기서 client는 transaction을 발생시키고, state는 transaction에 의해서 발생한 결과를 나타내며, transaction processor는 transaction을 받아 비지니스 로직을 처리한 후 state로 저장합니다. 

Sawtooth에 smallback transaction family(은행의 역할을 하는 smartcontract의 일종)를 예시로 설명드리겠습니다.

[small back transaction processor 전체 코드: https://github.com/hyperledger/sawtooth-core/blob/master/families/smallbank/smallbank_go/src/sawtooth_smallbank/handler/handler.go]

```go
func applyCreateAccount(createAccountData *smallbank_pb2.SmallbankTransactionPayload_CreateAccountTransactionData, context *processor.Context) error {
	account, err := loadAccount(createAccountData.CustomerId, context)
	if err != nil {
		return err
	}

	if account != nil {
		return &processor.InvalidTransactionError{Msg: "Account already exists"}
	}

	if createAccountData.CustomerName == "" {
		return &processor.InvalidTransactionError{Msg: "Customer Name must be set"}
	}

	new_account := &smallbank_pb2.Account{
		CustomerId:      createAccountData.CustomerId,
		CustomerName:    createAccountData.CustomerName,
		SavingsBalance:  createAccountData.InitialSavingsBalance,
		CheckingBalance: createAccountData.InitialCheckingBalance,
	}

	saveAccount(new_account, context)

	return nil
}

func saveAccount(account *smallbank_pb2.Account, context *processor.Context) error {
	address := namespace + hexdigest(fmt.Sprint(account.CustomerId))[:64]
	data, err := proto.Marshal(account)
	if err != nil {
		return &processor.InternalError{Msg: fmt.Sprint("Failed to serialize Account:", err)}
	}

	addresses, err := context.SetState(map[string][]byte{
		address: data,
	})
	if err != nil {
		return err
	}

	if len(addresses) == 0 {
		return &processor.InternalError{Msg: "No addresses in set response"}
	}
	return nil
}
```

위는 smallback에서 계좌를 생성하는 함수입니다. 함수를 보시면 `createAccountData`에서 계좌에 필요한 정보를 추출한뒤에 계좌를 만들고 `addresses, err := context.SetState(map[string][]byte{ address: data, })`다음과 같은 함수를 사용해 저장하는 것을 볼 수 있습니다. 블록에 저장되는 것은 transaction이지만 transaction에는 어떤 transaction processor에 어떠한 함수의 호출 정보만을 가지고 있기 때문에(예를 들어 A가 B에게 100달러 송금), 현재 누가 얼마의 돈을 가지고 있는정보를 알기는 어렵습니다. 따라서 현재 상태를 얻기 위해서는 제일 처음의 transaction부터 시작하여 현재까지의 모든 transaction을 실행시켜야하고 그렇게 하지 않기 위해 항상 global state에 현재의 모든 state들을 저장하여 최신 상태를 유지하게 됩니다. 즉 global state는 transaction의 순차적인 실행결과의 최신 반영본이며, 블록체인에 저장되는 transaction을 순서대로 실행시키면 매번 동일한 global state로 복구할 수 있습니다. transaction processor는 validator와 소켓으로 연결되어 있고 context의 setState, getState를 호출하는 경우 validator가 관리하는 `global state` 바로 저장 혹은 조회합니다.



자 그러면 다시 본론으로 돌아와서 sawtooth가 이 `global state`를 어떻게 관리 및 저장하는지 살펴보겠습니다. Sawtooth는 각 validator에서 하나의 Radix Merkle Tree를 활용해 모든 transaction families의 state를 관리합니다. 각 `state`는 transaction familiy의 작성자가 지정한 namespace에 따라 구분되며 transaction processor간에 공유될 수 있습니다.



## Radix Merkle Tree

Sawtooth의 `state`는 `map[string][]byte` 형태로 merkle tree의 노드로 저장되며 string은 address, []byte에는 transaction familiy가 정의한 struct가 serialize되어서 저장됩니다. `state`가 새로 생성 될때마다 merkle tree에 leaf node로 추가하고, state가 변경 될 때마다 merkle tree의 hash를 다시 계산해 root hash까지 변경합니다. Block을 저장할때 현재 merkle tree의 roothash를 같이 저장해 나중에 block으로부터 `global state` 를 복구할때 복구한 merkle tree의 root hash값과 비교하여 잘 복구했는지를 확인할 수 있습니다.



![](https://sawtooth.hyperledger.org/docs/core/releases/1.0/_images/state_radix.svg)

## Radix Address

Radix merkle tree의 node들은 유니크한 address로 구별되어 집니다.

![](https://sawtooth.hyperledger.org/docs/core/releases/1.0/_images/state_address_format.svg)



3bytes의 prefix와 transaction family의 작성자가 지정한 namespace + unique id(32 bytes)로 구성됩니다. 

```go
address := namespace + hexdigest(fmt.Sprint(account.CustomerId))[:64]
```