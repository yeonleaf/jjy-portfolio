# monthly-portfolio

## 프로젝트 개요
SK C&C BTV 운영 월간보고용 자동화 스크립트

<br>

## 🔧 사용 기술
- 언어: Python 3.12
- GUI 프레임워크 : pyside6
- PPTX 작성 : python-pptx

<br>

## 프로젝트 구조
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

<br>

## 🚀 주요 기능
- 명시한 년월, 시스템의 월간 보고용 PPTX 파일을 생성
- (optional) 엑셀 XLSX 이슈리스트 생성
- (optional) gpt o1-mini와 연동하여 insight 생성
- (optional) 커스텀 기능 제공 (이슈리스트 조회, 엑셀 M/D 계산, 일자 계산, PPT 단락 추가 방식, PPT 컬럼 별 디폴트 텍스트/bold/색상)

<br>

## 문제 및 해결 방법
### PyQt 기반 멀티스레드 환경에서 발생하는 세그먼트 폴트 (Segmentation Fault)

#### 배경
```
zsh: segmentation fault  python3 -m src.main
```
- PyQt5 및 PySide6 기반의 GUI 애플리케이션에서, QThread(Worker Thread)를 사용하여 REST API 요청, 데이터 정제, PPT/Excel 파일 생성 등의 작업을 수행함.
- 실시간 로그를 UI에 표시하기 위해 logger().warning(...)을 Worker Thread 내부에서 직접 호출함.
- 실행 중 간헐적으로 segmentation fault 발생 → Python 프로세스가 비정상 종료됨.
- 특히, faulthandler를 이용한 디버깅 결과, logging 모듈과 관련된 C 레벨 동작에서 충돌이 발생하는 것이 확인됨.

#### 🔍 원인 분석
1.	Python의 logging 모듈은 GIL을 사용하지만, C 라이브러리 레벨에서 동작하는 경우 충돌 가능성이 존재
- Worker Thread가 PyQt의 QThread에서 실행될 때, 로깅을 호출하면 내부적으로 C에서 실행되는 코드와 PyQt 이벤트 루프 간의 충돌이 발생할 수 있음.
- 특히 macOS + PySide6 + Python 3.13 환경에서는 이러한 충돌이 더 빈번하게 발생하는 것으로 확인됨.
2. Worker Thread에서 직접 logging을 호출하면서 UI 업데이트까지 시도하는 경우, UI 스레드와 충돌 가능성 증가
- GUI 업데이트는 반드시 **Main Thread(UI Thread)**에서 이루어져야 하는데, Worker Thread에서 직접 로그를 남기면서 UI도 변경하려 하면 불안정성이 증가할 수 있음.
3. 외부 라이브러리(Fetch 클래스)에서 logger().warning(...)을 호출하여 Worker Thread에서 실행되는 경우에도 같은 문제 발생
- Fetch 클래스는 REST API 호출과 응답 처리 기능을 담당하는 유틸리티 클래스이며, logger().warning(...)을 사용하여 로그를 남겼음.
- 이 코드가 Worker Thread 내에서 실행되면서 logging 모듈과 PyQt의 이벤트 루프가 충돌하여 세그먼트 폴트 발생.

✅ 해결 방법

1. Worker Thread에서 직접 logger().warning(...)을 호출하지 않음 > 세그먼트 폴트 발생 가능성을 원천 차단.
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
- WorkerThread에서 log_signal.emit("로그 메시지")을 호출하면, Main Thread에서 이를 받아 UI에 출력 + 실제 logger() 호출을 수행.

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
- 기존에는 Fetch 클래스 내부에서 logger().warning(...)을 직접 호출 → Worker Thread에서 실행될 경우 세그먼트 폴트 발생.
- 해결 방법: 콜백 함수(on_log)를 추가하여 로그를 Worker Thread로 넘기고, Worker Thread가 PyQt Signal을 통해 Main Thread로 전달.
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
#### 📌 결론 및 주요 교훈
	1.	Python의 logging 모듈은 C 레벨에서 실행될 때 멀티스레드 환경에서 충돌할 수 있음.
	2.	Worker Thread에서 직접 logging을 호출하면 예상치 못한 세그먼트 폴트가 발생할 가능성이 있음.
	3.	Worker Thread → PyQt Signal → Main Thread 구조를 활용하면, UI 업데이트와 logging을 안전하게 처리할 수 있음.
	4.	외부 유틸리티 클래스(Fetch)는 직접 logging을 호출하지 않고, 콜백을 활용하여 로그를 Worker Thread로 넘긴 후 PyQt Signal을 통해 Main Thread에서 로깅을 수행하는 방식이 안정적임.
	5.	멀티스레드 환경에서는 UI 업데이트는 Main Thread에서, Heavy 연산은 Worker Thread에서, logging은 Main Thread에서 처리하는 것이 원칙.

<br>

## 소개용 PPT

<br>

## GIF / 스크린샷
