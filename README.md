# Transactions-in-Hyperledger-Fabric

■ 참조 사이트 : https://medium.com/@kctheservant/transactions-in-hyperledger-fabric-50e068dda8a9

< Transactions in Hyperledger Fabric >
The ledger is composed of two parts: a blockchain and a world state database
트랜젝션은 blockchain을 구성하는 중요한 요소이며, 원장의 증거를 관찰하기 위한 요소입니다.

Simple하게 아래 항목들에 대해서 다루어 봅니다.

- a simplified lifecycle of a transaction
- how to read a block from the ledger
- Channel Genesis Block
- installation and instantiation of Fabcar chaincode(샘플코드)
- what is inside a transaction
- transactions after invoking chaincode functions
- various actors (client, peer and orderer) in the transaction and block creation


1. Lifecycle of a Transaction (사이트 그림참조)
1.1 Client 지정된 endorsing peer에 transaction proposal 전달
1.2 peer는 chaincode 실행
1.3 client에게 proposal response 전달(peer가 수행한 결과값이 있음 endorsement)
1.4 client는 transaction proposal, proposal response, endorsements 정보를 Orderer에게 전달.
1.5 Orderer는 transaction를 수신, 검증 후 블럭을 작성 후 모든 peer에게 브로드캐스팅함.
1.6 피어노드들은 블록의 유효성 검사 후 월ㅈ드 상태를 업데이트 및 유효블록으로 커밋함.

2. Basic Network and Fabcar Chaincode

2.1 Basic Network  : fabric-samples/basic-network 사용
- 하나의 organization(Org1), 하나의 Peer node(peer0.org1.example.com), 하나의 order node(orderer.example.com)
- 채널 정보 :  mychannel 생성
- 암호 자료와 채널 아티팩트는 자동으로 생성되므로 여기서는 Skip됨

2.2 Fabcar 체인 코드 : fabric-samples/chaincode/fabcar... 사용
내용 : 간단한 데이터베이스 기록 자동차 세부 사항 및 소유권
- chaincode instantiation : 아무작업을 않함.
- Invoke initLedger() : 10개의 데모용 자동차 레코드를 기록
- CAR# 색인(0~9), 자동차의 세부사항은 JSON형식으로 배치 및 문자열화되어 원장에 기록
- 체인 코드를 호출 API : changeCarOwner(), queryCar() 등

3. Basic Network 실행 흐름 (참조 사이트 The Demo Flow 그림)

3.1 Run basic-network/start.sh 
- join the peer0.org1 to mychannel
- Channel genesis block (Block #0)
3.2 Install Fabcar chaincode. 
- No block
3.3 stantiate Fabcar chaincode in mychannel
- Block #1 생성
3.4 voke Fabcar function initLedger()
- Block #2 생성
3.4 Invoke Fabcar function changeCarOwner()
- Block #3 생성
3.5 Invoke Fabcar function queryCar()
- Block #4 생성

4. 원장에서 블록을 읽는 방법
4.1 원장에서 blockchain information 가져오는 명령어
- blockchain information : 블록 체인의 높이, 현재 블록 및 이전 블록의 블록 해시 정보
$ docker exec [container] peer channel getinfo -c [channel_name]

4.2 원장에서 특정 block 가져오는 명령어
- [channel_name]_[block_number].block 파일로 저장됨. (예 : mychannel_0.block), 이 파일은 읽을수 없으므로 컨버팅이 필요함.
$ docker exec [container] peer channel fetch [block_number] -c [channel_name]

4.2.1 블록파일 컨버팅 방법
- localhost로 copy
$ docker cp [container]:[path of .block file] .

- 블록파일을 JSON 포맷으로 변환
$ ../bin/configtxgen -inspectBlock [.block file] > [.block.JSON]

5. Channel Genesis Block (참조사이트 그림참조)

- genesis.block 파일은 Channel Genesis Block 이 아님.
- 각 채널에는 Orderer의 genesis.block 파일에서 생성 된 고유한 Genesis Block이 있습니다.
- mychannel 용 Channel Genesis Block 생성 명령어 (basic-network/start.sh)
$ docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@org1.example.com/msp" peer0.org1.example.com peer channel create -o orderer.example.com:7050 -c mychannel -f /etc/hyperledger/configtx/channel.tx

- peer0.org1.example.com을 mychannel에 Join
$ docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@org1.example.com/msp" peer0.org1.example.com peer channel join -b mychannel.block

5.1 Channel Genesis Block: Block #0 내용
- 블록 : 헤더, 데이터, 메타 데이터 세 부분 구성
- 블록 헤더 : prev_hash 값 저장
- 블록 데이터 : 트랜잭션 정보 보관
- 블록 메타 데이터 : 해당 블록의 정보

6. Install and Instantiate Fabcar Chaincode
6.1 Chaincode Installation
- 체인 코드를 피어 노드에 패키징하는 것입니다. 따라서이 명령 이후에 새로운 블록이 생성되지 않습니다.
$ docker exec cli peer chaincode install -n fabcar -v 1.0 -p github.com/fabcar/go

- mychannel 원장의 높이를 확인하면 여전히 1입니다. 즉 원장에 한 블록 (블록 # 0)만 있습니다.
$ docker exec peer0.org1.example.com peer channel getinfo -c mychannel

6.2 Chaincode Instantiation
- 체인 코드 인스턴스화에서 endorser에 의해 실행되는 Init () 함수 입니다. Fabcar 체인 코드 Init()는 비어 있으며 인수가 필요하지 않습니다.
$ docker exec cli peer chaincode instantiate -o orderer.example.com:7050 -C mychannel -n fabcar -v 1.0 -c '{"Args":[]}' -P "OR ('Org1MSP.member')"

- 이제 높이가 2이므로 블록 체인에 새 블록 (블록 # 1)이 추가되었습니다.
$ docker exec peer0.org1.example.com peer channel getinfo -c mychannel

6.3 Blockchain Structure (그림참조)
- 블록 # 1의 헤더에서 우리는 블록 # 0의 해시 인 previous_hash (8B2cTc…)를 볼 수 있습니다
- 현재 블록 해시(xx5uLY…)는 블록 # 1에서 찾을 수 없습니다. 블록 # 2의 previous_hash에만 표시됩니다.

7. Transaction (그림참조)
- 블록 데이터 내부에는 트랜잭션 존재
- 트랜잭션 3개 부분으로 구성 (거래 제안서, 보증, 제안 응답)

7.1 거래 제안서 : 클라이언트는이 제안서를 승인하는 피어 노드에 전달
- 체인 코드는 lscc또는 Lifecycle System Chaincode
- 모든 데이터는 base64로 인코딩
- 디코딩 된 인수 목록 예제 : [“deploy”, “mychannel”, “fabcar 1.0”, “Org1MSP”, “escc”, “vscc”]
- escc 및 vscc는 각각 Endorsing System Chaincode 및 Validation System Chaincode

7.2 제안서 응답 및 보증(승인)
- 보증(승인) : 승인 피어 노드는 자체 ID 및 서명으로 반환
- 제안 응답 : 피어 노드 승인 후 요청 된 체인 코드를 RWSet 형식으로 실행 한 결과에 대한 응답코드
- proposal_response_payload : Read Write Set 결과값
- endorsements : endorser ID (peer0.org1.example.com), endorser signature

8. Invoke Various Chaincode Functions (예제 : Fabcar, 그림참조 )
-  세 가지 함수를 호출 예제를 통한 블록 생성

8.1 Invoke initLedger()
- initLedger() 10개의 자동차 레코드를 원장에 기록
$ docker exec cli peer chaincode invoke -o orderer.example.com:7050 -C mychannel -n fabcar -c '{"function":"initLedger","Args":[]}'

- 블록정보 : 현재 높이는 3, 블록 # 2가 생성
$ docker exec peer0.org1.example.com peer channel getinfo -c mychannel

8.1.1 Transaction Proposal 내용
- 체인 코드는 fabcar, 하나의 인수가 있으며 디코딩 된 결과는 initLedger
8.1.2 Proposal Response 내용
- endorsements 필드 (ID, signature)
- initLedger() writes 10 car records on the ledger and return nothing (response is 200)
- RWSet reads[] : (read nothing in ledger)
- RWSet writes : 원장에 10개의 자동차 기록
- CAR0의 decodeing value : {“make”:”Toyota”,”model”:”Prius”,”colour”:”blue”,”owner”:”Tomoko”}

8.2 Invoke changeCarOwner()
- 자동차의 소유자를 변경하는 함수
- 필요한 입력 파라미터는 자동차 ID와 새 소유자 이름
$ docker exec cli peer chaincode invoke -C mychannel -n fabcar -c '{"Args":["changeCarOwner", "CAR0", "KC"]}'

- 블록정보 : 현재 높이는 4, 블록 #3 생성

8.2.1 Transaction Proposal 내용
- 체인 코드는 fabcar
- 3개의 인수 [“changeCarOwner”, “CAR0”, “KC”]

8.2.1 Proposal Response 내용 (RWSet 중심내용)
- RWSet reads : Block #2 Transaction #0 내용이 있음
- RWSet writes : CAR0을 새로운 값으로 씁니다. 디코딩 된 값은 {“make”:”Toyota”,”model”:”Prius”,”colour”:”blue”,”owner”:”KC”}

8.3 Invoke queryCar()
- 원장에 기록된 특정 Car 정보를 불러오는 함수
- 이번에 는 원장 의 CAR0 레코드 와 함께 응답이 리턴됨
$ docker exec cli peer chaincode invoke -C mychannel -n fabcar -c '{"Args":["queryCar", "CAR0"]}'

- 블록정보 : 현재 높이는 5, 블록 #4 생성

8.3.1 Proposal Response 내용
- 디코드 된 메시지 : {“colour”:”blue”,”make”:”Toyota”,”model”:”Prius”,”owner”:”KC”}

9. 누가 블록에서 무엇을했는지? (그림참조)
- 패브릭 네트워크에서 다양한 구성원들(클라이언트, 피어 및 주문자)의 트랜잭션 및 블록을 처리하는 것 살펴본다.
- 블록 #3 예시

9.1 Client (CLI, with user Admin)
- Client는 transaction (including the transaction proposal, endorsements and proposal response)을 생성하고, 서명 한 후 Orderer에게 제출
- creator.id_bytes 디코더 내용 : Admin@org1.example.com 인증서

9.2 Endorsing Peer
- 제안서 응답 내에서 승인에 대한 보증인 및 서명 내용
- endorser 디코드 내용 : Org1MSP and the certificate of peer0.org1.example.com

9.3 Orderer
- 블록을 생성하고 메타 데이터에서 생성자와 서명을 확인 할수 있음.
- decoded result : OrdererMSP 정보 및 the certificate of orderer.example.com

요약
- 전체 사이클 : client(proposal 생성) -> endorsers(sign response) -> Clinet -> Oderer(블럭에 Proposal 및 response 생성) -> all peer (commit block & world state DB에 RWSet 수행)
