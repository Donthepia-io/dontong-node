# dontong-node

### Docker 설치 & 기본 세팅

```bash
# Docker 설치 스크립트를 다운로드합니다.
$ curl -fsSL https://get.docker.com -o get-docker.sh

# 다운로드한 스크립트를 실행하여 Docker를 설치합니다.
$ sudo sh get-docker.sh

# Docker Compose 설치를 위한 스크립트를 다운로드합니다. 이때, 시스템의 OS와 아키텍처에 맞는 버전이 자동으로 선택됩니다.
$ sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 다운로드한 Docker Compose 실행 파일에 실행 권한을 부여합니다.
$ sudo chmod +x /usr/local/bin/docker-compose

# 현재 사용자를 'docker' 그룹에 추가합니다. 이렇게 하면 sudo 없이 Docker 명령을 실행할 수 있습니다.
$ sudo usermod -aG docker $USER

# 변경된 그룹 설정을 적용하기 위해 새로운 그룹 세션을 시작합니다.
$ newgrp docker

# 'node'라는 이름의 디렉토리를 생성하고 해당 디렉토리로 이동합니다.
$ mkdir node && cd node

# 'password' 파일을 생성하고, 사용자의 입력을 받아 해당 파일에 저장합니다.
$ echo '비밀번호 입력' > password

# 부모 디렉토리(상위 디렉토리)로 이동합니다.
$ cd ..
```

### 검증자 계정 생성 파일 작성

docker-compose-geth-new-account.yml

```yml
version: '3'
services:
  init:
    image: ethereum/client-go
    command: account new --password="/root/.ethereum/password"
    volumes:
      - ./node:/root/.ethereum
```

### 검증자 계정 생성

생성 후 지갑 주소 저장

```bash
$ docker-compose -f docker-compose-geth-new-account.yml up
```

### genesis.json 파일 복사

node 디렉토리 안에서 파일 생성

```json
{
	"config": {
		"chainId": 8888,
		"homesteadBlock": 0,
		"eip150Block": 0,
		"eip150Hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
		"eip155Block": 0,
		"eip158Block": 0,
		"byzantiumBlock": 0,
		"constantinopleBlock": 0,
		"petersburgBlock": 0,
		"istanbulBlock": 0,
		"clique": {
			"period": 5,
			"epoch": 30000
		}
	},
	"nonce": "0x0",
	"timestamp": "0x634457ee",
	"extraData": "0x00000000000000000000000000000000000000000000000000000000000000008A62E918e1Bd3fb90Fc7dd62A57b756d002793450000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
	"gasLimit": "0x47b760",
	"difficulty": "0x1",
	"mixHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
	"coinbase": "0x0000000000000000000000000000000000000000",
	"alloc": {
		"8A62E918e1Bd3fb90Fc7dd62A57b756d00279345": {
			"balance": "0x200000000000000000000000000000000000000000000000000000000000000"
		}
	},
	"number": "0x0",
	"gasUsed": "0x0",
	"parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
	"baseFeePerGas": null
}
```

### 네트워크 초기화 파일 생성

docker-compose-geth-init.yml

```yml
version: '3'
services:
  init:
    image: ethereum/client-go
    command: init /root/.ethereum/genesis.json
    volumes:
      - ./node:/root/.ethereum
```

### 네트워크 초기화

```bash
$ docker-compose -f docker-compose-geth-init.yml up
```

### geth 실행 파일 생성

docker-compose.yml

- EC2_IP = geth를 실행할 ec2의 ip

- VALIDATOR_ADDRESS = 생성한 검증자 계정 주소 (0x는 제거하고 입력)

```yml
version: '3.8'
services:
  node:
    image: ethereum/client-go:v1.13.11
    container_name: node
    command: --datadir="/root/.ethereum"
      --syncmode="full"
      --gcmode="archive"
      --port=30303
      --http
      --http.addr="0.0.0.0"
      --http.port=8501
      --http.corsdomain="*"
      --http.api="personal,miner,eth,net,txpool,debug,clique"
      --http.vhosts="*"
      --ws
      --ws.addr="0.0.0.0"
      --ws.port=8502
      --ws.api="personal,miner,eth,net,txpool,debug,clique"
      --networkid=8888
      --nat="extip:[EC2_IP]"
      --verbosity=5
      --unlock="[VALIDATOR_ADDRESS]"
      --password=/root/.ethereum/password
      --miner.gasprice=0
      --miner.etherbase="[VALIDATOR_ADDRESS]"
      --allow-insecure-unlock
      --nodiscover
      --mine
    volumes:
      - ./node:/root/.ethereum:rw
    ports:
      - '30303:30303'
      - '8501:8501'
      - '8502:8502'
    networks:
      - poa-network

networks:
  poa-network:
    driver: bridge
```

### geth 실행

```bash
$ docker-compose up -d
```

### geth 접속

```bash
$ docker exec -it [DOCKER_CONTAINER_ID] /bin/sh

$ cd ~/.ethereum

$ sudo geth attach node/geth.ipc

```

### geth 연결

```bash
$ admin.addPeer("enode://24265b4d295a9449a614e977c54ea5d74bf3767f9b45fc1655b406a2940e36250219ef7a3536e0d47becce8ef14e3b0cb9a607f646a2046676ec0d1df7e936e8@43.203.80.242:30303?discport=0")
```

### geth 연결 확인

```bash
admin.peers
```
