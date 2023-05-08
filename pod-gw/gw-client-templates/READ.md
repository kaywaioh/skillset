## Client Site의 pod를 구축하는 방법을 소개하는 것이다.
## 이미 배포된 vpn client pod로 트래픽이 흐를 수 있게 한다.

### Preq - change here라고 주석달아놓은 곳들을 바꾼다. 사실 배포한 후에 해도된다.
1. nat-conf파일의 숫자로 고정 ip가 할당되므로 원하는 ip를 설정한다.
2. client_init.sh파일에 gw-pod의 pod IP를 설정한다.
3. settings.sh에 VXLAN의 cidr을 설정한다. pod/service등의 대역과 곂치지 않게 한다.

### Deploy
1. configmap 3개와 deployment를 배포한다.
    > nat.conf와 settings.sh는 /conf 안에 생성되며, client_init.sh는 /bin안에 생성된다.
    > pod의 privileged가 true가 아니면 client_init.sh를 수행할 수 없다.(권한 부족)
2. nat-conf파일의 new2-pod-gateway-client를 해당 pod의 hostname으로 변경필요
3. 배포한 pod에서 client_init.sh를 실행한다.
    - 최종적으로 Gateway ready and reachable 이라는 메세지가 나오는지 본다.
    - 라우팅 테이블을 확인한다.