AWS Read Replica RDS와 GCP BigQuery 연동 작업에 대한 회고입니다.

### 서론

기존 운영중인 서비스의 경우 AWS를 통해 클라우드 인프라를 구축하여 운영하였고 GTM을 통해 데이터를 수집, GA4로 모니터링 및 데이터를 분석하고 있었습니다.

서비스의 규모가 증가함에 따라 더욱 정교한 데이터 분석이 필요하게 되었고 데이터 분석을 위해 GA4와 BigQuery를 연동하였으나 기존에 운영중인 서비스의 데이터를 분석하기 위해 AWS RDS와 GCP BigQuery간의 실시간 데이터 연동이 필요하게 되었습니다.

다음은 AWS RDS와 GCP BigQuery를 연동하는 방법입니다.

1. AWS DMS 서비스를 통해 GCP Cloud SQL로 데이터를 복제하는 방법
2. GCP Datastream 서비스를 통해 CDC 파이프라인을 구축하는 방법

두 가지 방법 모두 가능하나 1번 방법의 경우 DMS와 Cloud SQL 두 가지 서비스를 사용해야 하고, 2번 방법의 경우 Datastream 서비스만을 사용하면 되기 때문에 비용과 복잡도 면에서 2번 방법을 선택하였습니다.


### AWS RDS Binary Log 설정

파라미터 그룹 수정

| 매개변수 | 값 |
| :------: | :------: |
| binlog_format | ROW |
| read_only | 1 |
| net_read_timeout | 3600 |
| net_write_timeout | 3600 |
| wait_timeout | 86400 |

백업 보존기간을 7로 변경하여 로그 보존 기간을 설정합니다.

RDS 인스턴스 재부팅


### AWS RDS 연결 설정

Datastream과 RDS 연동을 위해 다음과 같은 방법을 사용합니다.

1. VPC 피어링
2. IP 허용 목록 설정
3. SSH 터널링

VPC 피어링의 경우 GCP와 AWS의 VPC를 연결하는 방법으로, 두 클라우드의 네트워크를 연결하여 데이터를 전송할 수 있도록 합니다.

다만 VPC 피어링을 사용하려면 GCP에 VPC 내트워크를 생성하고 IAM 권한을 설정해야 하기 때문에 복잡도가 높습니다.

IP 허용 목록 설정의 경우 Datastream에서 제공하는 IP들을 RDS 보안 그룹에 추가하여 접근을 허용하는 방법입니다.

이 방법의 경우 복잡도가 낮지만 Datastream에서 제공하는 IP가 변경될 수 있고 보안상의 이유로 사용하지 않았습니다.

SSH 터널링의 경우 RDS의 서브넷이 소속된 VPC 내부에 Public EC2 인스턴스를 생성하고, EC2 인스턴스를 통해 RDS에 접근하는 방법입니다.

현재 운영중인 서비스의 경우 이미 Bastion 서버가 존재하고 있어 SSH 터널링을 사용하여 RDS에 접근하였습니다.


### GCP Datastream 설정

하단의 [참조 링크](https://velog.io/@minbrok/Datastream%EC%9D%84-%EC%82%AC%EC%9A%A9%ED%95%9C-Amazon-RDS-to-BigQuery-CDC-%ED%8C%8C%EC%9D%B4%ED%94%84%EB%9D%BC%EC%9D%B8-%EA%B5%AC%EC%B6%95)를 참고하여 Datastream을 설정합니다.


### 마치며

AWS RDS와 GCP BigQuery를 연동하는 방법에 대해 알아보았습니다.

두 클라우드 간의 연동은 복잡도가 높고 보안상의 이유로 어려움이 있지만, Datastream을 통해 CDC 파이프라인을 구축하면 복잡도를 낮출 수 있고, 실시간으로 데이터를 전송할 수 있습니다.

또한, Datastream을 통해 데이터를 전송하면 BigQuery에서 데이터를 분석하고 시각화할 수 있으며, 데이터를 분석하여 서비스의 성능을 개선할 수 있습니다.

데이터 분석을 통해 서비스의 성능을 개선하고 사용자들에게 더 나은 서비스를 제공할 수 있도록 노력하겠습니다.


### 참조
https://velog.io/@minbrok/Datastream%EC%9D%84-%EC%82%AC%EC%9A%A9%ED%95%9C-Amazon-RDS-to-BigQuery-CDC-%ED%8C%8C%EC%9D%B4%ED%94%84%EB%9D%BC%EC%9D%B8-%EA%B5%AC%EC%B6%95