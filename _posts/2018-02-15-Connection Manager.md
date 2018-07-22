## It-chain - ConnectionManager

It-chain의 각각의 Node들이 어떻게 같은 네트워크의 다른 Node들과 데이터를 주고 받으며, Connection을 유지, 생성, 관리, 종료하는지에 대해서 설명하고자 한다.



### Overall View

<img src="https://www.dropbox.com/s/bulsasqqytllceh/connection_overall_architecture.png?dl=1">



- ConnectionManager

ConnectionManager는 Connection들을 관리한다. Connection을 생성, 관리, 종료한다. ConnectionManager를 통해 다른 Peer들과 Connection이 연결되면 Connection을 유지한채로 연결된 Connection을 통해 데이터를 송신, 수신한다. Connection을 연결시겨 놓는 이유는 합의알고리즘과 Transaction을 송,수신의 비용을 줄이기 위함이다.

- Connection

Connection은 Peer간의 통신의 추상화이며 구현체는 Grpc Bi-Stream으로 구현되어 있다. Grpc Bi-Stream은 TCP/IP통신 처럼 Connection을 유지한채 데이터를 송,수신 할 수 있다. 각 Peer마다 하나의 Connection으로 유지되며 Connection은 clientConnection, serverConnection의 2가지 중 하나이다. Connection은 내부적으로 송신, 수신 파트를 각각 고루틴을 통해 비동기적으로 처리한다.



### Connection Implementation details

![connection_class_diagram](https://www.dropbox.com/s/cl95vw83phbx4pw/connection_class_diagram.png?dl=1)

Connection Class의 중요한 메소드와 속성들을 나타낸 Class Diagram이다. Connection 은 크게 2개의 종류로 분리된다. Connection에서 clientStream와 serverStream은 서로 동시에 존재 할 수 없다. Connection이 clientStream을 가지고 있으면 ClientConnection, serverStream을 가지고 있으면 ServerConnection으로 간주한다. 나머지 모든 로직은 동일하며 단순히 어느 Stream을 사용할지가 차이점이다. 

- ClientConnection

Peer가 grpc.ClientConn객체를 사용해 다른 Peer와 Connection 을 맺게 되면 ClientConnection이 생성된다.

- ServerConnection

Peer가 다른 Peer로 부터 Stream 요청을 받게 되는 순간 Connection이 생성된다. 그 순간 serverStream을 인자로 받게 되는데 그 serverStream을 가진 Connection이 ServerConnection이다.



### ClientConnection Sequence Diagram

![clientConnection_sequence_diagram](https://www.dropbox.com/s/21kq7u470no1crh/clientConnection_sequence_diagram.png?dl=1)

새로운 Connection연결을 위해 ConnectionManager에게 ip와 connectionID를 전달하면, ConnectionManager는 grpcClient를 사용하여 Stream을 통해 연결을 확인한다. 연결이 될 경우 StreamClient객체를 전달받는다. Connection객체를 생성하고 StreamClient를 와 기타 여러 정보들을 Connection객체에 저장후 Connection객체를 ConnectionManager의 ConnectionMap에 저장한다.

### ServerConnection Sequence Diagram

![serverConnection_sequence_diagram](https://www.dropbox.com/s/82oshcqt7mbj0on/serverConnection_sequence_diagram.png?dl=1)

다른 Peer로 부터 Connection연결 요청이 들어오면 StreamServer를 받게 된다. 나에게 Connection을 요청한 Peer가 Valid한 Peer인지를 확인하기 위해 Peer정보를 요청하고 알맞은 Peer정보가 오게 되면 기타 여러 정보들을 Connection객체에 저장후 Connection객체를 ConnectionManager의 ConnectionMap에 저장한다. 그 후에 Connection 생성 이벤트를 전송한다.