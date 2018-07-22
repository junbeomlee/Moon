## Micro service architecture - (2/3)

이번 포스팅에서 설명할 내용은 Micro service architecture(MSA)에서 사용되는 패턴중 Event Sourcing에 대해서 알아볼 예정입니다. 그리고 it-chain에서 사용하는 Midgard(이벤트 소싱 라이브러리)에 대해서 간단 예제로 소개해드리겠습니다.



## Event driven Microservices

오늘날 기업 업무 시스템은 수많은 변화게 빠르게 대처해야합니다. MSA 방식으로 구현된 서비스는 별도의 저장소를 사용하는 경우가 많고 필요에 따라 분산된 데이터의 값들을 일치시켜야 하는 경우도 있습니다. 일부 비즈니스 트랜잭션은 여러 서비스에 걸쳐 있으므로 서비스간의 데이터 일광성을 보장하는 메커니즘이 필요합니다. 

예를 들어, 고객이 신용 한도가 있는 전자 상거래 상점을 만들고 있다고 가정했을때, 응용 프로그램은 새 주문이 고객의 신용 한도를 초과하지 않도록 해야 합니다. Orders와 Customers가 다른 데이터베이스에 있기 때문에 응용 프로그램은 단순히 로컬 ACID 트랜잭션을 사용할 수 없습니다. 

Event driven을 사용한다면 각 서비스는 데이터를 업데이트 할 때마다 이벤트를 게시합니다. 다른 서비스들은 이 이벤트를 구독합니다. 그렇게 되면 다른 서비스의 특정 데이터의 상태가 변했을 경우 이벤트를 통해 해당하는 데이터를 일치 시킬 수 있습니다.



## Event sourcing

이벤트 소싱은 Event driven microservice에서 많이 사용되는 패턴으로 도메인 모델에서 발생하는 모든 이벤트를 기록하는 데이터 저장 기법입니다. 도메인 모델의 현재 상태를 데이터베이스에 저장하는 대신 이벤트의 순서를 저장한 다음에 이벤트를 순서대로 실행시켜 도메인 모델의 현재 상태로 복구할 수 있습니다. 



- 정규 데이터 구조의 단순함

  도메인 모델의 정규 데이터인 도메인 이벤트는 단순한 구조로 저장됩니다. 때문에 도메인 이벤트를 저장할 대상으로 관계형 데이터베이스, 문서 데이터베이스, 키-값 저장소, BLOB 저장소, 또는 [Event Store](https://geteventstore.com/)와 같이 이벤트 소싱에 특화된 저장소 등 다양한 데이터베이스를 고려할 수 있고 분산 저장소에 적합해 규모 확장도 쉽습니다. 그리고 도메인 이벤트는 수정되거나 삭제되지 않으며 오직 추가만 되기 때문에 기존 데이터에 접근하기 위한 경쟁이 발생하지 않습니다.

- 신뢰할 수 있는 시스템 기록 확보

  도메인 모델의 모든 변경 내역은 도메인 이벤트로 기록되기 때문에 특정 시점까지의 이벤트를 재생하면 해당 시점의 도메인 모델이 복원됩니다. 시스템에 오류가 발생하면 프로그래머는 오류가 발생한 시점의 도메인 모델을 복원해 오류를 분석할 수 있습니다.

- 메세지 중심 아키텍처에 적합

  비동기 메시지 전달을 중심으로 한 설계는 반응형 시스템의 기반입니다. 이미 설명된 것처럼 도메인 이벤트는 일급 데이터로 저장되기 때문에 이를 통해 **‘최소 일 회 배달(at-least-once delivery)’**을 구현하기 용이합니다. 도메인 모델에서 발생한 이벤트는 안정적으로 관찰될 수 있으며 반드시 발행됨을 보장할 수 있습니다. 또한 도메인 이벤트가 담긴 메시지는 높은 설명력을 가집니다.



### Event

Command로 부터 수행되어지는 비즈니스 로직의 결과로 aggregate의 상태 변화를 나타냅니다. UserCreatedEvent, UserNameUpdatedEvent등으로 나타내 질 수 있습니다. 



### Aggregate

어그리게이트는 도메인 모델을 나타냅니다.



```Go
//aggregate
type UserAggregate struct {
	Name string
	midgard.AggregateModel
}

//Build aggregate from event
func (u *UserAggregate) On(event midgard.Event) error {

	switch v := event.(type) {

	case *UserCreated:
		u.ID = v.ID

	case *UserNameUpdated:
		u.Name = v.Name

	default:
		return errors.New(fmt.Sprintf("unhandled event [%s]", v))
	}

	return nil
}

//event
type UserCreated struct {
	midgard.EventModel
}

//event
type UserNameUpdated struct {
	Name string
	midgard.EventModel
}
```



### Repository

이벤트를 영구 저장소에 저장하고 검색하는 데이터 엑세스 계층을 제공합니다.

```Go
type Repository struct {
	store     EventStore
	publisher EventPublisher
}

func NewRepo(store EventStore, publisher EventPublisher) *Repository {
	return &Repository{
		store:     store,
		publisher: publisher,
	}
}

//Load aggregate by id and replay saved event
func (r *Repository) Load(aggregate Aggregate, aggregateID string) error {

	if aggregateID == "" {
		return ErrInvaildAggregateID
	}

	if aggregate == nil {
		return ErrNilAggregate
	}

	events, err := r.store.Load(aggregateID)

	if err != nil {
		return err
	}

	for _, event := range events {
		err = aggregate.On(event)

		if err != nil {
			return errors.New("fail to ")
		}
	}

	return nil
}

// save events to aggregate and publish event
// todo 보상 event 추가
func (r *Repository) Save(aggregateID string, events ...Event) error {

	if len(events) == 0 {
		return errors.New("no events to save")
	}

	for _, event := range events {
		if event.GetType() == "" {
			return errors.New("all event need type")
		}
	}

	err := r.store.Save(aggregateID, events...)

	if err != nil {
		return err
	}

	if r.publisher != nil {

		for _, event := range events {
			//Todo type implicit problem
			err := r.publisher.Publish("Event", event.GetType(), event)
			if err != nil {
				return errors.New("need roll back")
			}
		}
	}

	return nil
}
```

Repository는 2가지의 기능을 수행합니다. aggregate의 ID를 key 값으로 event들을 저장하는 기능과 aggregate의 id로 이벤트들을 찾고 aggregate에 각 이벤트들을 모두 적용하여 최신상태의 aggregate를 load하는 기능입니다. 또한 event저장시에 event bus를 통해 event를 publish해서 해당 event를 subsrcibe한 다른 서비스들이 이벤트를 받을 수 있도록 합니다.



### Command

Aggregate의 어떤 상태 변화를 이끄는 요청입니다.





## Example

```Go
//aggregate
type UserAggregate struct {
	Name string
	midgard.EventModel
}

func (u *UserAggregate) On(event midgard.Event) error {

	switch v := event.(type) {

	case *UserCreated:
		u.ID = v.ID

	case *UserUpdated:
		u.Name = v.Name

	default:
		return errors.New(fmt.Sprintf("unhandled event [%s]", v))
	}

	return nil
}

//event
type UserCreated struct {
	midgard.EventModel
}

//event
type UserUpdated struct {
	Name string
	midgard.EventModel
}

func TestNewRepository(t *testing.T) {

	path := "test"
	defer os.RemoveAll(path)

	store := leveldb.NewEventStore(path, leveldb.NewSerializer(UserCreated{}, UserUpdated{}))
	r := midgard.NewRepo(store, nil)

	aggregateID := "123"

	err := r.Save(aggregateID, UserCreated{
		EventModel: midgard.EventModel{
			ID:   aggregateID,
			Type: "User",
		},
	})

	assert.NoError(t, err)

	err = r.Save(aggregateID, UserUpdated{
		EventModel: midgard.EventModel{
			ID:   aggregateID,
			Type: "User",
		},
		Name: "jun",
	})

	assert.NoError(t, err)

	err = r.Save(aggregateID, UserUpdated{
		EventModel: midgard.EventModel{
			ID:   aggregateID,
			Type: "User",
		},
		Name: "jun2",
	})

	assert.NoError(t, err)

	user := &UserAggregate{}

	err = r.Load(user, aggregateID)
	assert.NoError(t, err)

	assert.Equal(t, user.ID, aggregateID)
	assert.Equal(t, user.Name, "jun2")

	fmt.Println(user)
}
```

이 예제는 Event를 저장하고 다시 복원하여 현재 상태로 aggregate를 불러오는 예제입니다. 이 예제를 보면 여러가지 이상한점을 볼 수 있는데 우리가 일반적으로 생각하는 List조회나 조건문 검색등을 수행할 수 가 없다는 점입니다. 이러한 문제점들을 해결하기 위해 CQRS란 패턴이 보통 event sourcing과 함께 사용됩니다. CQRS는 생성,삭제, 업데이트와 조회를 분리시키는 패턴입니다. 다음 포스팅에서 CQRS에 대해서 좀더 자세히 설명 드리겠습니다!