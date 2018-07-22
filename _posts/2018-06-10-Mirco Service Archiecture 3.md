## Micro service architecture - (3/3)

이번 포스팅에서 설명할 내용은 Micro service architecture(MSA)에서 사용되는 패턴중 CQRS(Command and Query Responsibility Segregation)에 대해서 알아볼 예정입니다. 그리고 it-chain에서 사용되는 간단한 예제를 보여드리겠습니다.



## CQRS (Command and Query Responsibility Segregation) 란?

CQRS는 명령과 쿼리의 역할을 구분하는 것이다. 전통적인 CRUD아키텍처를 사용하여 어플리케이션을 개발하다 보면 자연스레 Domain의 복잡도가 증가하게 됩니다. 따라서 비지니스 로직과 관련된 Domain Model과 데이터 조회의 작업을 분리하여 Domain의 복잡도를 줄이는 것입니다.

CQRS를 이해하려면 Command와 query라는 용어를 정의할 필요가 있습니다. Command란 시스템의 상태를 변경하는 작업을 의미하고 query는 시스템의 상태를 반환하는 작업을 의미합니다. 정리해서 Command란 시스템의 Domain 객체들의 상태를 변화시키는 요청(Create, Update, Delete)이고 Query는 Command에 의해 변경된 Domain 객체들의 상태를 조회(Read)하는 것입니다.

CQRS의 핵심은 단순히 CUD와 R을 분리하는 것이기 때문에, CQRS를 구현하는 구현체의 모습은 모두 제각각으로 표현될 수 있습니다. 



![mono and micro](https://ravendb.net/RavenFS/GetImage?filePath=/docs/articles/articles/cqrs-and-event-sourcing-made-easy-with-ravendb/cqrs1.jpg)



그림은 간단한 CQRS패턴을 적용한 구조를 보여줍니다. Command Model은 Command를 받아서 Domain객체를 활용해 비지니스 로직을 처리하고 결과를 DB에 저장합니다. 저장된 결과를 조회하기 위해서는 Query를 통해 조회 할 수 있습니다.



## CQRS + Event Sourcing

![mono and micro](https://cdn-images-1.medium.com/max/1600/1*oCdpBemlRxTZxrK6XD0l8w.jpeg)



위 그림은 CQRS와 Event Sourcing이 같이 사용되는 시스템을 보여주고 있습니다. Command Side에서 Command Handler가 Command(요청)을 받으면 Aggregate를 이용하여 Business logic을 처리하고 발생된 Event를 Repository(Event Repository)에 저장합니다. 그러면 Event는 Repository에 의해 EventDB에 저장되고, EventBus에 의해 전파됩니다.

Query Side에서는 관심있는 Domain객체의 변화 event를 등록해놓았다가 event가 발생하면 Query의 입장에서 알맞은 형태로 Mysql, mongodb등에 저장하게 됩니다. 이를 Projection이라고 부릅니다. 다시말해 Projection은 event의 stream을 조회 서비스에 알맞은 형태로 가공하여 저장하는 것을 의미합니다. 

그림을 보면 하나의 Event도 여러개의 Projection에 의해 가공되는 것을 볼 수 있는데 이는 각 조회 서비스의 관심에 따라서 다른 형태로 저장될 수 있기 때문입니다. 예를 들어 유저생성 이벤트가 발생되었다고 하면, 일반적인 조회 서비스에서는 이벤트를 유저의 형태로 변환하여 우리가 보통 저장하는 table형태로 Mysql에 저장합니다. 이벤트 발생 내역을 추적하기 위해 개발된 이벤트 내역 조회 서비스에서는 대용량 이벤트를 빠르게 조회할 수 있도록 이벤트 그대로 Mongodb, elastic에 저장합니다. 이런식으로 Query, Command 모델을 분리하여 우리가 원하는 데이터 조회를 목적에 맞도록 쉽게 구현할 수 있습니다.



## Example

**Command Side**

```Go
package api

import (
	"log"
	"time"

	"github.com/it-chain/it-chain-Engine/txpool"
	"github.com/it-chain/midgard"
)

type TransactionApi struct {
	eventRepository *midgard.Repository
	publisherId     string
}

func NewTransactionApi(eventRepository *midgard.Repository, publisherId string) TransactionApi {
	return TransactionApi{
		publisherId:     publisherId,
		eventRepository: eventRepository,
	}
}

func (t TransactionApi) CreateTransaction(txCreateCommand txpool.TxCreateCommand) {

	events := make([]midgard.Event, 0)

	if txCreateCommand.GetID() == "" {
		log.Println("need id")
		return
	}

	id := txCreateCommand.GetID()
	timeStamp := time.Now()
	hash := txpool.CalTxHash(txCreateCommand.TxData, t.publisherId, txpool.TransactionId(id), timeStamp)

	events = append(events, txpool.TxCreatedEvent{
		EventModel: midgard.EventModel{
			ID:   id,
			Type: "Transaction",
		},
		PublishPeerId: t.publisherId,
		TxStatus:      txpool.VALID,
		TxHash:        hash,
		TimeStamp:     timeStamp,
		TxData:        txCreateCommand.TxData,
	})

	err := t.eventRepository.Save(id, events...)

	if err != nil {
		log.Println(err.Error())
	}
}
```

위 예제는 it-chain에서 transaction생성 요청을 처리하는 함수입니다. 일반적인 api service와는 다르게 transaction을 직접 repository를 통해 저장하지 않고 TxCreateCommand의 요청을 확인하고 그 요청으로 부터 TxCreatedEvent를 생성한 뒤에 TxCreatedEvent를 저장하고 있습니다.



**Query Side**

```Go
package messaging

import (
	"log"

	"github.com/it-chain/it-chain-Engine/txpool"
)

type TxEventHandler struct {
	txRepository     txpool.TransactionRepository
	leaderRepository txpool.LeaderRepository
}

//add tx to txrepository
func (t TxEventHandler) HandleTxCreatedEvent(txCreatedEvent txpool.TxCreatedEvent) {

	txID := txCreatedEvent.ID

	if txID == "" {
		return
	}

	tx := txCreatedEvent.GetTransaction()
	err := t.txRepository.Save(tx)

	if err != nil {
		log.Println(err.Error())
	}
}
```

위 예제는 Command Side에서 생성한 TxCreatedEvent를 받아서 txRepository에 저장하는 부분입니다. TxCreatedEvent로 부터 Transaction을 생성한 후에, 그 Transaction을 Repository에 저장합니다. 나중에 조회는 이 TxRepository를 이용하여 조회됩니다.



CQRS패턴을 마지막으로 5월 15일 부터 MSA에 관해서 포스팅을 진행하였습니다. 앞에서 포스팅한 아키텍처와 패턴이 it-chain Engine에 왜 어떻게 적용되었고 어떻게 작동되고 있는지 구체적으로 설명드리겠습니다. 

(6월 12일 아침에 석사 논문 디펜스를 앞두고 있는데, 벌써 석사를 졸업할 시기가 되었다니 참 시간이 빠르네요.. 디펜스 잘 성공하고 다음 포스팅에서 찾아뵙겠습니다...)