# 아주인

[서비스 링크](https://www.ajou.in)

## 1. 기능

- 사용자가 정보를 효율적으로 접할 수 있도록, 50개 이상의 학교 웹사이트에 분산되어 있던 공지사항을 한 곳에서 모아보는 기능 제공.
- 사용자가 중요한 공지사항에 리마인더를 설정하면, 원하는 날짜와 시간에 메일로 해당 공지사항 리마인드를 전송해주는 기능 제공.
- 공지사항 저장을 위한 북마크 기능 제공.
- 공지사항 목록에서 바로 볼 수 있는 세 줄 요약 제공.
- (Legacy version) 아주대학교 도메인 지식으로 학습된 챗봇

[레거시버전 깃허브 링크](https://github.com/man-moon/ajouin-be-legacy.git)

## 2. 프론트엔드
[프론트엔드 링크](https://github.com/man-moon/ajouin-fe.git)


## 3. 백엔드

#### 애플리케이션 목록









- [회원 서비스](https://github.com/man-moon/member-service.git)  
    메일 리마인더, 북마크, 회원관리
- [공지사항 조회 서비스](https://github.com/man-moon/notice-view.git)  
    공지사항 데이터 제공

#### 설계 목표

- 서비스 분리 및 장애 대응 강화  
복잡한 데이터 파이프라인 내 장애에 신속하게 대응하기 위해, 기능별로 서비스를 분리하여 운영. 이를 통해 한 서비스의 수정이나 장애가 전체 시스템에 영향을 미치지 않도록 설계.

- 운영 서버 가동률 99.9%  
라이브 서비스와 공지사항 데이터 파이프라인을 명확히 분리하여 수정이 빈번한 기능을 분리. 각 서비스별로 독립적인 빌드/테스트/배포 파이프라인을 구축함으로써, 서비스 중단 없이 안정적인 운영 환경을 유지.

- 데이터 파이프라인 안정성 향상  
OCR API, OpenAI API, 이미지 저장 등 긴 처리시간이 소요되는 파이프라인을 개별의 독립된 서비스로 분리하고, 메시지 큐를 도입하여 비동기적으로 처리함으로써 전체 파이프라인의 안정성을 향상.

- DB 성능 최적화 및 확장성 확보  
대량의 Write 작업과 라이브 서버에서 Read 작업이 동시에 발생하는 경우, 사용자에게 응답이 지연되는 상황 발생. 이 문제를 해결하기 위해 Write와 Read용 DB를 분리하는 일종의 CQRS 패턴을 적용하고, Read 빈도가 높은 운영 서버에는 조회 성능에 우수한 Document DB를 도입. Write와 Read용 DB의 싱크를 맞추기 위해 CDC를 적용하여 실시간으로 데이터 동기화 보장.

- 유연한 코드 설계와 다양한 HTML 대응  
교내 공지사항 사이트의 HTML 양식이 비일관적인 문제를 해결하기 위해, 팩토리 패턴 기반의 구조를 도입. 이를 통해 10종 이상의 다양한 HTML 양식에 대해 동적으로 파싱 로직을 교체하며, 변화에 유연하게 대응할 수 있도록 설계.


### 3-1. 데이터 파이프라인 백엔드

#### 아키텍처
![데이터 파이프라인 아키텍처](https://private-user-images.githubusercontent.com/88218891/423135390-50c015a6-0bca-4e2a-b26b-ec08880635bd.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDIwNTY2NzgsIm5iZiI6MTc0MjA1NjM3OCwicGF0aCI6Ii84ODIxODg5MS80MjMxMzUzOTAtNTBjMDE1YTYtMGJjYS00ZTJhLWIyNmItZWMwODg4MDYzNWJkLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNTAzMTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjUwMzE1VDE2MzI1OFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTZiMDVjMTRkZDc1OWNhODRlOGU2ZjFlY2FmMjNmYjE0ODM1NDNmY2MyMmNiZGJhN2ZmMGMwOTNhY2MxMGFhOGMmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.sF7GwUdtCdhvr2bQFE1sQ8yfM1YJIdrR4NA_sXJ5LVc)

- [공지사항 목록 스크래퍼(Notice list Scraper)](https://github.com/man-moon/notice-scraper.git)  
    10분 주기로 50개 이상의 교내 홈페이지에서 스크래핑 할 공지사항을 필터링하여 수집. 공지사항 리스트를 notice-req 메시지 큐에 전달

- [OCR 서비스(OCR Service)](https://github.com/man-moon/ocr-service.git)  
    이미지 내 텍스트 추출 (Google Vision API 활용). ocr-req 메시지 수신 → OCR 수행 후 ocr-res 메시지 송신

- [요약 서비스(Summary Service)](https://github.com/man-moon/summary-service.git)  
    공지사항 본문(+ OCR 결과) 요약 (OpenAI API 활용)

- [공지사항 데이터 프로세싱 서비스(Notice Date Processing Service)](https://github.com/man-moon/notice-content-scraper.git)  
    공지사항 전처리 및 DB 저장을 오케스트레이션.
    1. 공지사항 목록(notice-req) 수신
    1. OCR, 요약 서비스에 작업을 요청 (ocr-req, summary-req)
    1. 응답(ocr-res, summary-res) 종합 후 DB에 최종 저장

마이크로서비스와 메시지 큐로 서비스 간 결합도를 낮게 유지하여, 특정 서비스 장애 발생 시 다른 서비스에 영향을 최소화하는 효과.  
OCR, 요약 등 외부 API를 사용하여 시간이 많이 걸리는 작업을 메시지 큐 기반 비동기로 분산 처리.

#### CQRS와 CDC
![CDC](https://private-user-images.githubusercontent.com/88218891/423136258-650bf6bd-3de8-44ac-88f7-5e076075af1d.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDIwNTY4NzIsIm5iZiI6MTc0MjA1NjU3MiwicGF0aCI6Ii84ODIxODg5MS80MjMxMzYyNTgtNjUwYmY2YmQtM2RlOC00NGFjLTg4ZjctNWUwNzYwNzVhZjFkLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNTAzMTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjUwMzE1VDE2MzYxMlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTk2Zjg3NjczMGM2NmY3YjU4MWNjYTljMWQ2YTYzMjEzNDMyYTBkMzRjMzI2MTA5YTc3YWU3NjNlODJlYjY2N2MmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.Qau49M_tzHx96aWhcWlzUc9SyI6QRJtcXdSmuUT56v8)

1. **CQRS**  
    쓰기(Write) 작업은 트랜잭션 무결성이 중요한 Maria DB에서 수행하고, 읽기(Read) 작업은 빠른 조회가 가능한 MongoDB에서 처리하도록 분리. 대량의 쓰기·읽기 요청이 동시에 발생해도 각각의 DB가 전문화된 역할을 수행할 수 있어, 성능 저하와 DB 락 문제를 효과적으로 해결.

1. **CDC**  
    Maria DB의 bin log를 Debezium으로 구독(subscribe)하여, DB에 발생하는 변경사항(INSERT, UPDATE, DELETE)을 실시간 이벤트로 Kafka 브로커에 전달.
    Kafka Broker는 수신한 이벤트를 Debezium Sink Connector가 이를 MongoDB에 반영.
    데이터 변경 지연을 최소화하고, 일관성과 신뢰성을 보장하는 시스템 구축.

### 3-2. API 서버 백엔드

#### 아키텍처

![운영서버 아키텍처](https://private-user-images.githubusercontent.com/88218891/423134816-0e7c2fa3-6062-40d4-86b5-29b26b995074.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDIwNTY1NDgsIm5iZiI6MTc0MjA1NjI0OCwicGF0aCI6Ii84ODIxODg5MS80MjMxMzQ4MTYtMGU3YzJmYTMtNjA2Mi00MGQ0LTg2YjUtMjliMjZiOTk1MDc0LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNTAzMTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjUwMzE1VDE2MzA0OFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTY1ZDM4MzYxMzI3ZGMyMWRmYmYzNTE2ODcyYjA4MGMzZmU4Y2ZjYTgyNDk3YWJlMzc4N2Q1MmJjNzk0OTkyNzImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.rTI_mwKRQbgNuVOOglcAjjg-F9X9kB8niUUNiXERp7A)

운영 서버는 API Gateway를 중심으로, 외부 요청을 Notice View Service와 Member Service로 분산 처리하는 구조를 채택. 각 서비스는 메시지 큐를 통해 다른 서비스와 통신.  
마이크로서비스가 늘어나도 확장성을 유지하며, 서비스 간 의존성을 낮출 수 있도록 설계.

### 3-3. 챗봇 백엔드(Legacy, Not available now)

#### 아키텍처

![챗봇 아키텍처](https://private-user-images.githubusercontent.com/88218891/423192064-121fd24a-cf25-4fdd-873f-652fd225b967.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDIxMDkxODEsIm5iZiI6MTc0MjEwODg4MSwicGF0aCI6Ii84ODIxODg5MS80MjMxOTIwNjQtMTIxZmQyNGEtY2YyNS00ZmRkLTg3M2YtNjUyZmQyMjViOTY3LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNTAzMTYlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjUwMzE2VDA3MDgwMVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWJlMjJhMzg1MDJkMGU3ZjcxNjgzNWE1M2JiMDAyOTJiYWY5YTQ0YTRhYjNiYjBjNzA0YjczYzUzN2M5MGQ1NmYmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.JX7g_6b4rIHPj5kRnbV8cx7HK1KsyoXhYUeFtQ9pQBs)

챗봇은 학교 공지사항, 학사 정보, 위키 데이터(사용자 참여 데이터), 교내 웹사이트 크롤링 데이터 등을 활용하여 사용자의 질문에 실시간으로 답변. 기본적으로는 앞서 언급한 데이터로 학습된 교내 도메인 지식을 활용하는 내부 LLM 서버를 통해 정확한 응답을 제공. 대기열이 3 이상으로 증가하는 등 트래픽이 증가할 경우에는 OpenAI의 GPT-4 API로 전환. 이를 통해 서비스 안정성과 응답 속도를 동시에 확보하며 비용 역시 최소화 가능.

GPT-4 API 사용 시에는 Vector DB(PostgreSQL)와 LangChain을 통해 관련 도메인 지식을 함께 제공하여, 질문 맥락을 더욱 정확히 파악하고 최적의 답변을 생성. 예를 들어, "학교 주변 한식 음식점을 추천해주세요."나 "도서관 운영시간이 어떻게 되나요?" 등의 질문에 대해, 학교 LLM 서버는 즉시 응답하고, 트래픽이 증가된 상황에는 GPT-4 API가 자동으로 요청을 분산 처리.

로드 밸런서를 통해 요청을 수집한 뒤, 대기열 처리 워커가 요청 상태를 모니터링하고, 필요 시 내부 LLM 서버와 GPT-4 API를 동적으로 선택하도록 설계.

### 4. 로깅

여러 서비스로 나눠진 상태에서 오류가 발생하는 경우, 로그를 보기가 쉽지 않음. 특히 데이터 파이프라인의 경우, 한 데이터에서 문제가 발생하면 해당 데이터는 연쇄적으로 문제가 생기는 경우가 있음. 이에 각 데이터별로 추적이 가능하도록 해야함. -> 메시지 큐에 들어가는 데이터에 고유한 id를 붙여, 해당 id로 로깅. 문제 발생 시, 각 서비스의 로그를 한 곳에 취합하여, 어떤 데이터에 문제가 생겼는지 id로 필터링하여 문제 파악 가능.

분산된 마이크로서비스 환경에서 오류 추적은 중요한 문제. 특히, 데이터 파이프라인에서는 단일 데이터의 오류가 연쇄적인 문제를 야기할 수 있으므로, 각 데이터별로 추적할 수 있는 체계 구축이 필요.  
1. 고유 ID 기반 로깅
    메시지 큐에 전달되는 각 데이터에 고유한 ID를 부여하고, 모든 서비스에서 해당 ID를 사용하여 로깅.
    이를 통해 특정 데이터에서 오류가 발생했을 때, ID를 기준으로 모든 서비스의 로그를 빠르게 조회하고 문제 지점을 파악.

1. 중앙 집중 로그 수집
    여러 서비스에서 발생하는 로그를 한 곳에 취합하여 모니터링 및 분석을 용이하게 함.
    각 로그를 ID로 필터링하여, 어떤 데이터에서 문제가 시작되었는지 파악.

### 5. 테스트 환경 및 Config 주입

![환경설정 아키텍처](https://private-user-images.githubusercontent.com/88218891/423140510-55fbfbb3-125e-4cac-81d4-cd881474a2b7.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDIwNTc5NDcsIm5iZiI6MTc0MjA1NzY0NywicGF0aCI6Ii84ODIxODg5MS80MjMxNDA1MTAtNTVmYmZiYjMtMTI1ZS00Y2FjLTgxZDQtY2Q4ODE0NzRhMmI3LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNTAzMTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjUwMzE1VDE2NTQwN1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWY1MTU1OWI5YjFjMDRjNDE5MDlhNDlkMDYwNzBmZDQyZWE4ZWQwMjQ3YmJjN2UxZjc3Mjk2MDQ1ZDRlNjA4YmUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.tY2Y__93U42JMvhcGQAKj_lnHS5Ew9hN7Kuw1zG0Hhw)

테스트 환경과 운영 환경을 분리하여, 테스트 통과 시 자동 배포가 이루어지도록 구성. 이를 위해 RabbitMQ(Spring Cloud Bus)와 Config Service를 연동하고, 각 환경별 설정 파일을 GitHub 저장소에서 관리. 테스트 환경에서는 MQ와 DB를 별도로 구성해 운영 리소스와의 충돌을 방지하고, CI/CD 파이프라인에서 모든 테스트가 성공해야만 운영 환경에 배포되도록 하여 안정성을 높임. 또한 Config Service를 통해 각 마이크로서비스가 실행 시점에 필요한 설정을 동적으로 주입받게 함으로써, 코드 수정 없이도 환경 전환이 가능하도록 설계.

### 6. CI/CD

![CI/CD](https://private-user-images.githubusercontent.com/88218891/423144804-10cda196-81b9-4997-a7d3-36d87d372220.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDIwNTkyMjEsIm5iZiI6MTc0MjA1ODkyMSwicGF0aCI6Ii84ODIxODg5MS80MjMxNDQ4MDQtMTBjZGExOTYtODFiOS00OTk3LWE3ZDMtMzZkODdkMzcyMjIwLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNTAzMTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjUwMzE1VDE3MTUyMVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWI4N2JhZGU4MzQ3ODc2N2FiMTQ2ODdhMzRlNjMyYzUwMDMyYWRhYmJlOGQ0ZTA3MmQ3YjYyNmMwNjYwMWEzNzAmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.LfT01pmZHBkgJmAdf6wvO1QvrYso57GcSLyAsMDW5VY)

Jenkins를 통해 GitHub에 코드가 푸시되는 순간 자동으로 빌드와 테스트가 수행되고, 통과시 배포되는 CI/CD 파이프라인. 각 서비스(Notice View, Member, OCR, Summary, Notice List Scraper, Notice Data Processing)는 독립적인 파이프라인을 사용.  
배포 환경은 Docker 컨테이너 기반. Jenkins에서 SSH 연결을 통해 컨테이너를 배포함으로써 운영 서버로의 반영 과정을 자동화.

### 7. 운영서버 인프라(Oracle Cloud)

#### Notice view Service
- CPU VM.Standard.A1.Flex / 1 OCPU(1 core 1 thread)  
- memory 8GB  
- network 1Gbps  

#### Member Service
- CPU VM.Standard.A1.Flex / 1 OCPU(1 core 1 thread)  
- memory 4GB  
- network 1Gbps  

#### NoticeDB
- CPU VM.Standard.A1.Flex / 1 OCPU(1 core 1 thread)  
- memory 8GB  
- network 1Gbps  

#### MemberDB
- CPU VM.Standard.A1.Flex / 1 OCPU(1 core 1 thread)  
- memory 4GB  
- network 1Gbps  

#### API Gateway
- CPU VM.Standard.A1.Flex / 1 OCPU(1 core 1 thread)  
- memory 2GB  
- network 1Gbps  
