# DB Migration Script 개발 회고

본 문서는 MariaDB에서 PostgreSQL로의 DB Migration 작업에 대한 회고록입니다.

서버 스펙 :
- macOS Sonoma 14.2.1
- MariaDB v10.6
- PostgreSQL v15.5
- Python v3.11.7
- pip v23.3.1
- Node.js v21.5.0
- npm v10.2.4

## 서론

운영 중인 서비스의 개편을 위해 여러가지 방법을 고려하던 중 MedusaJs라는 쇼핑몰 빌드용 Open Source를 발견하게 되어 해당 프로젝트를 Fork하여 개발에 도전하게 되었습니다.

MedusaJs는 기본적으로 PostgreSQL을 사용하고 있었기 때문에 기존 서비스의 DB를 PostgreSQL로 Migration하는 작업이 필요했고 이를 위해 다음과 같은 방법을 고려하게 되었습니다.

1. 기존 데이터를 CSV 또는 JSON 파일로 Export 후 MedusaJs Migration & Seed 기능 사용
2. 기존 데이터를 Query로 Export 후 가공하여 PostgreSQL에 Import
3. Python으로 Script를 작성하여 기존 데이터를 PostgreSQL에 Import

개편 예정 서비스의 경우 기존 데이터를 변환하여 마이그레이션을 진행해야했으며 지속적으로 운영중인 사이트였기에 최대한 빠르게 마이그레이션을 진행해야했습니다.

따라서 1번 방법은 데이터의 양이 많을 경우 시간이 오래 걸릴 것이라고 판단하였고 2번 방법은 기존 데이터의 구조가 PostgreSQL의 구조와 맞지 않아 가공하는데 시간이 오래 걸릴 것이라고 판단하여 3번 방법을 선택하게 되었습니다.

## Python Script 작성

Migration Script의 경우 이후 다시 사용할 수 있도록 재사용성을 고려하여 작성하였습니다.

다음은 Migration Script의 구조입니다.

```
├── migrations
│   ├── __init__.py
│   └── domain-2023-12-31-000000.py
├── .env
├── .gitignore
├── migration.py
├── requirements.txt
└── README.md
```

각 파일의 역할은 다음과 같습니다.

- migrations : 각 도메인에 대한 Migration Script가 위치하는 디렉토리
- .env : DB 접속 정보를 설정하는 파일
- .gitignore : Git에 업로드하지 않을 파일을 설정하는 파일
- migration.py : 공통 Migration Script를 실행하는 파일
- requirements.txt : Migration Script 실행에 필요한 Python 라이브러리를 정의하는 파일
- README.md : Migration Script에 대한 설명을 작성하는 파일

각 파일의 예시는 [Python Migration Script](https://github.com/dhrtn1006/migration-script/tree/main/migration.python)에서 확인하실 수 있습니다.

## MedusaJs의 암호화 이슈

MedusaJs의 경우 암호화 모듈로 Node.js의 scrypt-kdf를 채택하고 있었는데 이 모듈과 동일한 암호화 방식을 사용하기 위해 Python의 scrypt 모듈을 사용하였습니다.

하지만 MedusaJs의 scrypt-kdf 모듈과 Python의 scrypt 모듈은 동일한 암호화 방식을 사용하고 있음에도 불구하고 암호화 결과가 다르게 나타나는 이슈가 발생하였습니다.

여러가지 방법을 시도해보았지만 결과적으로 실패하게 되었고 결국 기존 Python으로 작성된 Migration Script를 Node.js로 변환하고 Node.js의 scrypt-kdf 모듈을 사용하여 암호화 이슈를 해결하는 것으로 방향을 변경하게 되었습니다.

## Node.js Script 작성

암호화 이슈를 해결하기 위해 Python으로 작성된 Migration Script를 Node.js로 변환하여 작성하게 되었습니다.

다음은 Node.js Migration Script의 구조입니다.

```
├── migrations
│   └── domain-2023-12-31-000000.js
├── node_modules
├── .env
├── .gitignore
├── migration.js
├── package.json
└── README.md
```

각 파일의 역할은 다음과 같습니다.

- migrations : 각 도메인에 대한 Migration Script가 위치하는 디렉토리
- node_modules : Node.js Script 실행에 필요한 모듈이 위치하는 디렉토리
- .env : DB 접속 정보를 설정하는 파일
- .gitignore : Git에 업로드하지 않을 파일을 설정하는 파일
- migration.js : 공통 Migration Script를 실행하는 파일
- package.json : Node.js Script 실행에 필요한 모듈을 정의하는 파일
- README.md : Migration Script에 대한 설명을 작성하는 파일

각 파일의 예시는 [Node Migration Script](https://github.com/dhrtn1006/migration-script/tree/main/migration.node)에서 확인하실 수 있습니다.

## 마치며

이번 Migration 작업을 통해 Node.js의 사용에 조금 더 익숙해질 수 있었고 지금까지 사용해보지 못했던 Python 또한 사용해볼 수 있었습니다.

데이터를 처리하는 작업이라는 점에서 막연히 Python을 사용하면 좋을 것이라고 생각했지만 Node.js로도 생각보다 빠르게 처리할 수 있었고 다만 아쉬운 점은 암호화에 대한 이해도가 높지 않아 암호화 이슈를 해결하지 못하고 scrypt-kdf 모듈을 사용하게 되었다는 점입니다.