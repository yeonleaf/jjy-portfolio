# 월간보고 자동화 애플리케이션 개발
⚠️ 아래에 첨부된 모든 코드는 예시용으로 재작성되었습니다.

<br>

## GIF / 스크린샷
![demo](https://github.com/user-attachments/assets/f1691738-a8aa-4c0d-8c98-08a8939faa76)

<br>

## 개요
SK C&C BTV 운영 월간보고용 자동화 애플리케이션
- 수행 기간 : 1.5개월
- 수행 인원 : 1인
  
<br>

## 기술
- 언어: Python 3.12
- GUI 프레임워크 : pyside6
- PPTX 작성 : python-pptx

<br>

## 주요 기능
- 명시한 년월, 시스템의 월간 보고용 PPTX 파일을 생성
- (optional) 엑셀 XLSX 이슈리스트 생성
- (optional) gpt o1-mini와 연동하여 insight 생성
- (optional) 커스텀 기능 제공
	- 이슈리스트 조회 시 커스텀 JQL 사용 지원
 	- 엑셀 M/D 계산
  	- 일자 계산
  	- PPT 단락 추가 방식
  	- PPT 컬럼 별 디폴트 텍스트/bold/색상

<br>

## 핵심 기여
1️⃣ GUI 기반 자동화로 사용자 경험 개선
- 기존 수작업으로 진행되던 월간 보고서 작성 및 정리 프로세스를 pyside6 기반의 GUI 애플리케이션으로 자동화하여 비개발자도 쉽게 사용할 수 있도록 구현.

2️⃣ Worker Thread 활용한 비즈니스 로직 분리 – GUI 블로킹 방지
- 보고서 생성 과정이 파일 I/O 처리 등 무거운 로직을 포함하여, 메인 UI 스레드만 사용시 GUI의 응답 지연(Freeze) 문제가 발생.
- 이를 해결하기 위해 QThread를 활용하여 백그라운드에서 작업을 수행, GUI의 응답성을 유지.
- QThread에서 발생하는 로그를 메인 UI로 안전하게 전달하기 위해 Thread-Safe한 로깅 핸들러를 별도로 구현. (Signal 기반의 UI 갱신 방식)

3️⃣ Pydantic과 YAML 기반 설정 파일 검증
- 설정 파일을 구성할 때 상속을 통해 일관된 프로세스를 구축하고, Pydantic 모델을 통해 유효성 검증을 자동화하여 오류 발생 가능성을 최소화.

4️⃣ 로깅 시스템과 실시간 UI 로그 뷰어
- Python 로깅 모듈을 활용해 파일과 UI 창 두 곳 모두에 로그를 출력, 예기치 않은 오류나 작업 현황을 실시간으로 파악 가능. 

5️⃣ 테스트코드 280개 작성으로 핵심 로직에 대해 90% 커버리지를 달성
- 실제 PPT 파일을 활용한 통합 테스트 수행으로 자동 생성된 보고서의 정합성 및 양식 검증.

<br>

## 문제 및 해결 방법

### REST API 호출 시 429 에러 발생 및 Rate Limit 대응
#### 📖 배경
```
HTTP/1.1 429 Too Many Requests
```
- 사내에서 사용하는 Jira 서버에 Personal Access Token으로 API를 호출하던 도중, 서버 측에서 해당 토큰을 차단.
- 일괄적으로 429 (Too Many Requests) 상태를 반환하여, 실제로는 **Token이 차단됨** 인지 **Rate Limit 초과**인지 구분이 불가.
- 만에 하나 실제로 Rate Limit가 걸려 발생한 429일 수도 있으므로, 재시도 로직과 대기(Backoff) 로직을 구현하여 보완.

#### ✅ 해결 방법
1. 기본 대기 시간을 1초로 설정

- 일반적으로 1초는 긴 편이지만, 관리자용 내부 앱에서는 성능보다 Rate Limit 초과로 고객사에 불편을 주지 않는 것이 더 중요.
- 단, 서버에서 준 Rate Limit 관련 헤더 정보가 존재하면 1초 미만 간격으로도 추가 요청을 시도할 수 있게 함.

2. 429 응답 시, 헤더 정보(X-RateLimit-Remaining, Retry-After)를 참조한 Backoff 로직 적용
```python
@staticmethod
def avoid_rate_limit(response, default_sleep=1, max_retry_after=60):
    rate_limit_remaining = response.headers.get("X-RateLimit-Remaining")
    retry_after = response.headers.get("Retry-After")
    
    if rate_limit_remaining is not None:
        # 남은 호출 횟수가 충분치 않으면 기본 1초 대기
        if int(rate_limit_remaining) <= 3:
            time.sleep(default_sleep)
    elif retry_after is not None:
        # 서버가 명시적으로 대기 시간을 알려준 경우
        wait_time = int(retry_after)
        if wait_time > max_retry_after:
            raise ApiException("대기 시간이 너무 깁니다.")
        time.sleep(wait_time if wait_time > 0 else default_sleep)
    else:
        # 헤더 정보가 불충분한 경우라도 기본 대기
        time.sleep(default_sleep)
```
- Retry-After가 지나치게 큰 값(예: 60초 초과)이면 즉시 예외를 발생시켜 무한 대기를 방지.

3. 재시도(Exponential Backoff) 로직과 결합

- 429, 5xx 상태코드에 대해 지수적으로 대기 시간을 증가시키며 재시도.
- 재시도 횟수가 max_retries를 초과하면 예외를 발생시켜 무한 재시도를 막음.

```python
@staticmethod
def fetch_with_retry(method, url, max_retries=5):
    retries = 0
    delay = 1
    while retries < max_retries:
        response = requests.request(method=method, url=url)
        if response.status_code in [429, 500, 502, 503]:
            Fetch.avoid_rate_limit(response, delay)
            retries += 1
            delay = min(delay * 2, 32)  # 최대 32초까지 증가
        else:
            response.raise_for_status()
            return response
    raise ApiException("재시도 횟수 초과")
```
4. Pytest 기반 자동화 테스트
- 429 상태코드 시나리오, Retry-After 헤더 유무, 5xx 에러, 재시도 횟수 초과 상황 등을 각각 모의(Mock)하고, 실제로 설정한 재시도 로직이 의도대로 동작하는지 검증.

<br>

### PyQt 기반 멀티스레드 환경에서 발생하는 세그먼트 폴트 (Segmentation Fault)

#### 📖 배경
```
zsh: segmentation fault  python3 -m src.main
```
- PySide6 기반의 GUI 애플리케이션에서, QThread(Worker Thread)를 사용하여 비즈니스 로직을 수행함.
- 실시간 로그를 UI에 표시하기 위해 `logger().warning(...)`을 Worker Thread 내부에서 직접 호출함.
- 실행 중 간헐적으로 segmentation fault 발생 → Python 프로세스가 비정상 종료됨.

#### 🔍 해결 과정
Python의 logging 모듈은 C 라이브러리 레벨에서 동작하는 경우 충돌 가능성이 존재
- Worker Thread가 logging을 호출하면 내부적으로 C에서 실행되는 코드와 PyQt 이벤트 루프 간의 충돌이 발생할 수 있음.
- 외부 유틸이 로깅을 호출하여 Worker Thread에서 실행되는 경우에도 같은 문제 발생

#### ✅ 해결 방법
1. Worker Thread에서 직접 logging을 호출하지 않음 → 세그먼트 폴트 발생 가능성을 원천 차단.
- Worker Thread에서는 logging을 직접 호출하는 대신, PyQt Signal을 사용하여 Main Thread로 로그를 전달하도록 수정.

```python
class WorkerThread(QThread):
    log_signal = Signal(str)  # 로그 메시지를 전달할 PyQt Signal

    def run(self):
        self.log_signal.emit("작업 시작")  # Worker Thread에서 Main Thread로 로그 전송
        try:
            self.process()
        except Exception as e:
            self.log_signal.emit(f"에러 발생: {str(e)}")
    
    def process(self):
        # 기존 logger() 호출 대신 log_signal 사용
        self.log_signal.emit("API 요청을 시작합니다...")
```
- WorkerThread에서 `log_signal.emit("로그 메시지")`을 호출하면, Main Thread에서 이를 받아 UI에 출력 + 실제 logger() 호출을 수행.

```python
class MainApplication(QMainWindow):
    def __init__(self):
        super().__init__()
        self.log_text_edit = QTextEdit()
        self.setCentralWidget(self.log_text_edit)

        self.worker = WorkerThread()
        self.worker.log_signal.connect(self.on_worker_log)  # Signal 연결

    def on_worker_log(self, log_msg: str):
        self.log_text_edit.append(log_msg)  # UI 업데이트
        logger().info(log_msg)  # 실제 로깅 수행
```

2. 외부 유틸 클래스(Fetch)에서 콜백 패턴을 적용하여 Worker Thread 내 logging 호출 제거
- 콜백 함수(on_log)를 추가하여 로그를 Worker Thread로 넘기고, Worker Thread가 PyQt Signal을 통해 Main Thread로 전달.
```python
class Fetch:
    @staticmethod
    def fetch_with_retry(method: str, url: str, on_log: Callable = None):
        retries = 0
        while retries < 5:
            response = requests.get(url)
            if response.status_code == 200:
                if retries > 1 and on_log:  # on_log 콜백 사용
                    on_log(f"재시도 성공: {retries}회")
                return response
            else:
                retries += 1
                if on_log:
                    on_log(f"재시도 중... ({retries}회)")

# WorkerThread에서 Fetch를 호출할 때 on_log로 log_signal을 연결
class WorkerThread(QThread):
    log_signal = Signal(str)

    def run(self):
        Fetch.fetch_with_retry(
            method="GET",
            url="https://example.com",
            on_log=lambda msg: self.log_signal.emit(msg)  # 로그를 WorkerThread의 Signal로 전달
        )
```

<br>

### PySide6 기반 macOS 애플리케이션 배포 문제 해결
#### 📖 배경
PySide6를 사용하여 macOS용 .app 번들을 배포했으나, Finder에서 더블클릭 시 실행되지 않는 문제가 발생.

터미널에서 ./main을 직접 실행하면 정상적으로 동작했지만, Finder 실행에서는 아무런 로그도 출력되지 않고 종료.

#### 🔍 해결 과정
1. macOS 보안 정책 분석
- `codesign --force --deep --sign` 을 통해 코드 서명을 적용했음에도 해결되지 않음
- `sudo spctl --master-disable`로 Gatekeeper를 완전히 비활성화했으나 동일한 문제 발생

2. 라이브러리 로딩 문제 분석
- DYLD_PRINT_LIBRARIES=1로 실행 로그를 확인하려 했으나 아무 출력이 없었음
- otool -L main을 사용해 의존 라이브러리 확인 결과, @executable_path/Python이 포함되어 있었음
- 터미널에서 unix 실행 파일을 실행할 때와 Finder에서 실행할 때 Python 경로 변수에 차이가 있어 발생하는 문제로 추정함 

#### ✅ 해결 방법
대체 실행 방식 채택
- .app 외부에서 run.command 스크립트를 실행하는 방식으로 변경


<br>

## 구조
```plaintext
SKCC-BTV-MONTHLY
│── resources/              # 프로젝트에서 사용하는 리소스 파일 (예: 아이콘, 설정 파일)
│── src/                    # 소스 코드 디렉터리
│   ├── gui/                # GUI 관련 모듈
│   │   ├── core/           # GUI의 핵심 로직
│   │   │   ├── config_core.py       # 설정 관련 로직
│   │   │   ├── worker_core.py       # 백그라운드 작업 처리
│   │   ├── models/         # 데이터 모델 정의
│   │   │   ├── metadata_model.py    # 설정(슬라이드 메타데이터) 관련 모델
│   │   │   ├── settings_model.py    # 설정 관련 모델
│   ├── __init__.py
│   ├── date_utils.py       # 날짜 관련 유틸리티
│   ├── excel_utils.py      # 엑셀 처리 관련 유틸리티
│   ├── exception.py        # 예외 처리 모듈
│   ├── fetch.py            # 데이터 수집 관련 모듈
│   ├── logging_utils.py    # 로깅 관련 유틸리티
│   ├── main.py             # 실행 진입점 (Main 스크립트)
│   ├── messages.py         # 메시지 처리 모듈
│   ├── monthly.py          # 월간 보고서 생성 로직
│   ├── openai_utils.py     # OpenAI API 연동 모듈 (예: 자동화된 텍스트 생성)
│   ├── ppt_utils.py        # PPT 파일 처리 관련 유틸리티
│   ├── pysidedeploy.spec   # PySide 배포 관련 스펙 파일
│   ├── rc_resources.py     # 리소스 파일 관련 코드
│   ├── resources.qrc       # Qt 리소스 파일
│── tests/                  # 테스트 코드 디렉터리
```

