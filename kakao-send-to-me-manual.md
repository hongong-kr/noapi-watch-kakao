# 공개 API를 제공하지 않는 홈페이지를 효과적으로 탐지하고 실시간으로 카카오톡 알림까지 받는 절차(AI에게 주는 자료)

> 이 파일은 사람이 읽고 따라 하는 설명 문서가 아니다. 이 파일을 개인 AI 에이전트에게 그대로 건네면, 에이전트가 아래 순서대로 실행한다. 에이전트는 사람이 직접 해야 하는 일만 작동자(사람)에게 확인받고, 나머지는 스스로 실행·검증한 뒤 결과를 보고한다. 작동자(사람)는 이 파일을 처음부터 읽을 필요가 없다. 에이전트가 묻는 것만 답하면 된다.
>
> **작동자(사람)**는 작업을 하는 AI를 다루는 사람이다. 이 지시서에서 사람을 가리킬 때는 이 말로만 부른다. 콘솔 로그인·클릭·동의, 키·코드 복사, 폰 확인, 최종 승인은 작동자(사람) 몫이고, 그 밖의 실행은 에이전트 몫이다.

- **목표:** 카카오톡 '나와의 채팅'에 메시지를 보내는 기능을 동작 상태로 만든 뒤, 그 기능을 알림 통로로 써서 공공 홈페이지에서 원하는 부분만 골라 바뀌었는지 살펴보고, 새 소식이 생겼을 때만 작동자(사람)에게 알리는 흐름까지 끝까지 연결한다.
- **범위(기본):** '나에게 보내기' 구축 + 경량 감시 엔진(Track 1) 연결. Docker 배포(Track 2)와 확장 서비스(친구에게 보내기·알림톡·브랜드메시지·챗봇)는 작동자(사람)가 명시적으로 고른 경우에만 별도 단계로 진행한다.
- **기준일:** 2026-11-03, 카카오디벨로퍼스 공식 문서 대조 완료. 콘솔 화면·단가는 바뀔 수 있으니 에이전트는 실행 당일 공식 문서를 한 번 더 대조하고, 경로가 다르면 공식 문서 경로를 따른다.
- **핵심 규격:** 텍스트 본문 최대 200자(190자 단위 분할), 액세스 토큰 6시간, 리프레시 토큰 2개월, 나에게 보내기 비용 무료(쿼터 범위 내).

## 0. 이 파일을 개인 AI에게 건네면 무엇이 좋아지는가

에이전트는 이 절을 작동자(사람)에게 설명할 때 그대로 쓴다. 과장하지 마라. 확인되지 않은 효과는 "미확인"이라고 밝혀라.

**건네기 전:** AI는 물어볼 때만 답한다. 공공 웹페이지는 작동자(사람)가 직접 들어가서 새로고침하며 확인해야 하고, 전체 페이지를 눈으로 훑다 보니 배너·카운터 같은 잡다한 변동에 시간을 쓰고, 정작 새 의안·새 공고가 올라온 시점은 놓치기 쉽다. 알림도 없다.

**건네고 난 뒤, 달라지는 것:**

1. **탐지 능력이 바뀐다.** AI가 정해진 주기마다 정해진 페이지만 확인한다. 페이지 전체를 비교하는 것이 아니라, 표·문단·상태값처럼 원하는 부분만 골라 본다. CSS 선택자·XPath·정규식·키워드 필터로 범위를 좁히므로, 관련 없는 변동은 알림으로 올라오지 않는다.
2. **새 소식만 카톡으로 온다.** 변동이 없으면 아무 메시지도 보내지 않는다(침묵 규칙). 새 항목이 생겼을 때만 '나와의 채팅'으로 제목·출처·링크가 온다. 작동자(사람)는 카톡만 확인하면 된다. 다만 '나에게 보내기' 메시지의 푸시 알림 울림 여부는 공식 문서로 확인된 사항이 아니므로, 실제 폰에서 확인한 결과 그대로만 말한다.
3. **근거가 남는다.** 직전 스냅샷과 새 항목을 비교해 무엇이 새로 생겼는지 남긴다. 나중에 "언제, 무엇이, 어디에 올라왔나"를 스냅샷 이력으로 확인할 수 있다.
4. **작동자(사람) 손이 덜 간다.** 한 번 구축해 두면, 토큰 갱신·분할 발송·주기 실행은 에이전트와 스크립트가 맡는다. 작동자(사람)는 바뀔 때만 답하고, 토큰이 완전히 만료됐을 때만 다시 인가 코드를 준다.

**이 흐름의 출처와 한계:** 감시 아이디어는 2026-11-03 슬랙 잡담 채널의 GitHub 유용 도구 토론에서 Instinct가 `changedetection.io`를 소개한 데서 나왔다. 농그니는 API가 없는 공공 페이지의 선택자 기반 노이즈 제거를, Muse는 의안·감사자료의 이력 보존과 중복 방지를, 에이미는 동적 페이지 포착과 사생활 보호 관점을 보탰다. 이 지시서는 그 논의를 경량 엔진으로 먼저 구현하고, Docker 배포는 선택지로 둔 것이다. 모든 공공 페이지에 바로 통하는 만능 감시가 아니다. 페이지 구조가 바뀌면 파서를 다시 맞춰야 하고, 로그인 뒤 페이지나 접근이 막힌 페이지는 이 흐름으로 감시하지 마라.

## 1. 에이전트의 역할과 완료 기준

에이전트는 구축 주체다. 작동자(사람)는 에이전트가 할 수 없는 행위(콘솔 로그인·클릭·동의·코드 복사·폰 확인·최종 승인)만 수행한다. 에이전트가 "됐다고 생각한다"는 완료가 아니다. 아래가 모두 확인돼야 완료다.

**완료 기준 (Done):**

- [ ] `config.json`, `send_memo.py`, `tokens.json` 3개 파일이 작업 폴더에 생성됐다.
- [ ] `send_memo.py`가 컴파일되고, 200자·250자 분할 테스트를 통과했다.
- [ ] 작동자(사람) 폰의 '나와의 채팅'에 테스트 메시지가 실제로 도착했고, 작동자(사람)가 직접 확인했다.
- [ ] 리프레시 토큰 만료일과 재인가 절차가 인수인계 보고서에 적혔다.
- [ ] 감시 대상·키워드·주기가 작동자(사람) 승인을 받아 `targets.json`에 확정됐고, 초기 스냅샷이 저장됐다.
- [ ] 시뮬레이션(`--simulate`)으로 신규 항목 1건이 실제로 카톡으로 발송되는 것이 확인됐다. 무변동 실행에서는 메시지가 오지 않음이 확인됐다.
- [ ] 재설정 방법(8절)이 인수인계 보고서에 적혔다.

## 2. 절대 규칙

1. **게이트를 건너뛰지 마라.** 각 단계의 확인 사항이 답을 받기 전에는 다음 단계로 가지 마라. 값을 추측하거나 빈 값으로 진행하지 마라.
2. **질문은 묶어서 한 번에.** 작동자(사람)에게 물을 때는 해당 게이트의 질문을 한 메시지로 묶어 보내라. 한 번에 하나씩 묻지 마라. 기본값이 있는 항목은 "기본값으로 진행해도 되는지"를 함께 물어라.
3. **키·토큰·시크릿·인가 코드를 채팅·로그·보고서에 그대로 출력하지 마라.** 마스킹해서 보고하고, 파일에는 권한 600으로만 저장한다. 공유 문서·채팅방·코드 저장소에 올리지 마라.
4. **콘솔 조작은 작동자(사람) 몫이다.** 에이전트는 콘솔에 로그인하거나 클릭을 대신한다고 주장하지 마라. 정확한 경로·입력값·확인 방법을 주고, 작동자(사람)가 한 일을 값으로 받아 검증만 한다.
5. **에러는 코드부터.** 실패하면 에러 코드를 확인하고 6절 대응표대로 조치하라. 같은 요청을 무작정 반복하지 마라.
6. **사실은 공식 문서 기준.** 이 지시서와 공식 문서가 다르면 실행 당일의 공식 문서를 따르고, 달라진 점을 보고서에 적어라. 단가·쿼터·콘솔 경로는 지어내지 마라.
7. **사전 제안·승인제를 지켜라.** 좋은 도구를 알았다고 임의로 설치하거나 자원을 쓰지 마라. 감시 대상 추가, Docker 배포, 주기 변경, 확장 서비스처럼 새로 시작하는 일은 효용과 자원 영향(무과금 여부·부하·알림량)을 카드 형태로 먼저 제안하고, 작동자(사람) 승인을 받은 뒤에만 실행한다.
8. **무변동이면 침묵하라.** 감시에서 변동이 없으면 어떤 메시지도 보내지 마라. "변동 없음" 보고를 매번 보내지 마라.
9. **예시는 예시로 다뤄라.** 4절의 감시 대상·키워드는 예시다. 작동자(사람) 확인 없이 그대로 운용 설정으로 굳히지 마라.

## 3. 작동자(사람) 확인 사항 일람

아래는 전체 확인 항목이다. 에이전트는 0단계에서 OP-01·OP-04·OP-06·OP-07·OP-10만 먼저 한 번에 묻고, 나머지는 각 게이트에서 받을 때 묻는다. 작동자(사람)는 모르는 항목이 있으면 "모른다"고 답하고, 에이전트는 그 항목만 따로 안내한다. 기본값이 있는 항목은 작동자(사람)가 "기본값으로 진행"이라고 한 마디로 답할 수 있다.

| ID | 확인 사항 | 왜 필요한지 | 작동자(사람)가 주는 답 | 기본값 / 대안 |
|---|---|---|---|---|
| OP-01 | 카카오디벨로퍼스 앱 | REST API 키의 주인. 기존 앱을 쓸지 새로 만들지 에이전트는 알 수 없음 | 기존 앱 이름, 또는 "새로 만들겠다" | 없음. 반드시 답 필요 |
| OP-02 | REST API 키 | 인가 URL·토큰 요청의 `client_id` | 콘솔에서 복사한 키 문자열 | 없음. 1단계 후 전달 |
| OP-03 | Client Secret 처리 방침 | 기본 '사용함'이면 매 토큰 요청에 Secret 필요. 빼먹으면 KOE010 | "사용 안 함으로 바꿨다" 또는 Secret 코드 전달 | **사용 안 함으로 전환 권장.** 바꾸기 싫으면 Secret 코드를 에이전트에게만 전달 |
| OP-04 | 리다이렉트 URI·링크 도메인 | 요청값과 등록값이 끝 슬래시까지 같아야 함(KOE006). 메시지 링크는 등록 도메인만 가능 | "localhost 기본값" 또는 쓸 도메인 | 리다이렉트 URI `https://localhost`, 링크 도메인 없음(없으면 `link` 제외) |
| OP-05 | `talk_message` 동의 설정 | 없으면 메시지 전송 권한 토큰을 받을 수 없음 | "선택 동의(또는 이용 중 동의)로 설정했다" | 없음. 1단계 후 확인 |
| OP-06 | 작업 폴더·Python 환경 | 파일들을 어디에 만들고 어떤 Python으로 실행할지 | 폴더 경로, 또는 "기본 폴더로 해줘" | 현재 폴더에 생성, Python 3 표준 라이브러리만 사용(설치 불필요) |
| OP-07 | 확장 서비스 범위 선택 | 확장 서비스는 심사·채널·비용이 따로 붙음. 기본 범위 밖은 함부로 시작하지 않음 | "나에게 보내기만" 또는 원하는 확장 서비스명 | **나에게 보내기만** |

| ID | 확인 사항 | 왜 필요한지 | 작동자(사람)가 주는 답 | 기본값 / 대안 |
|---|---|---|---|---|
| OP-08 | 인가 코드 | 본인 로그인·동의 후 주소창에 뜨는 1회용 코드. 수 분 내 만료 | `?code=` 뒤 문자열 전체 | 없음. 2단계에서 즉시 전달 |
| OP-09 | 테스트 발송 확인 | 폰에서 실제 도착·알림 여부를 사람만 확인 가능 | "도착했다 / 알림이 왔다(또는 안 왔다)" | 테스트 문구: "카카오톡 나에게 보내기 테스트 메시지입니다!" |
| OP-10 | 감시 방식 선택 | 경량 엔진만으로 할지, Docker 배포까지 할지, 둘 다 할지 정해야 함 | "경량 엔진만" 또는 "Docker까지" 또는 "둘 다" | **경량 엔진만** |
| OP-11 | 감시 대상·키워드·필터 | 4절 예시를 그대로 쓸지, 빼고 더할지를 정해야 함 | "예시 그대로" 또는 바꿀 대상·키워드 목록 | **예시 그대로 쓰지 않음.** 4절을 보여주고 답을 받아 확정 |
| OP-12 | 감시 주기 | 너무 짧으면 대상 서버와 폰에 부담, 너무 길면 늦게 안다 | "2시간마다" 또는 원하는 주기 | **2시간마다(홀수 시각 정각)** |
| OP-13 | 알림 채널 | 카톡 외에 슬랙 웹훅 등을 함께 쓸지 정해야 함 | "카톡만" 또는 추가할 채널과 웹훅 정보 | **카톡만** |
| OP-14 | 초기 스냅샷 알림 여부 | 처음 저장할 때 기존 항목을 알림으로 쏠지 말지 정해야 함 | "처음에는 알림 없이 저장" 또는 "기존 항목도 알려줘" | **알림 없이 저장하고 종료** |
| OP-15 | Docker 실행 환경 | Track 2를 고른 경우에만 필요. Docker가 있는 서버·PC가 어디인지 | 서버·PC 위치와 접속 방법 | 없음. Track 2를 고른 경우에만 묻는다 |

**에이전트 주의:** OP-02·OP-03·OP-08은 비밀 정보다. 받자마자 파일에만 쓰고, 이후 대화에서는 마스킹(앞 4자 + `...`)으로만 언급하라. 작동자(사람)가 키를 채팅에 붙여 넣으면 "파일에 저장했고 채팅 기록에서는 지워달라"고 안내하라. OP-13의 웹훅 주소도 비밀 정보와 같이 다뤄라.

## 4. 예시로 들어 있는 감시 설정 — 그대로 쓰지 말고 물어보고 바꿔라

이 절은 SOP에 예시로 들어 있던 설정이다. **이 지시서를 건네받은 모든 사용자에게 그대로 적용되는 고정값이 아니다.** 에이전트는 6단계에서 아래 표를 작동자(사람)에게 그대로 보여주고, "이대로 할까요, 무엇을 빼고 더할까요"라고 물은 뒤 답을 받아 `targets.json`을 확정한다. 답이 없으면 감시 단계를 시작하지 마라.

**예시 감시 대상 5종:**

| ID | 예시 이름 | 예시 URL |
|---|---|---|
| `smc_reception` | 서울시의회 교통위원회 접수의안 | https://www.smc.seoul.kr/comt/information.do?menuId=015004001 |
| `smc_review` | 서울시의회 교통위원회 검토보고서 | https://www.smc.seoul.kr/comt/information.do?menuId=015004003 |
| `smc_report` | 서울시의회 교통위원회 업무보고자료 | https://www.smc.seoul.kr/board/BoardList.do?boardTypeId=98&menuId=015005002 |
| `iris_testbed` | 범부처통합연구지원시스템(IRIS) 테스트베드 R&D 공고 | https://www.iris.go.kr/contents/retrieveBsnsAncmListView.do |
| `ntis_testbed` | 국가과학기술지식정보서비스(NTIS) 테스트베드 R&D 공고 | https://www.ntis.go.kr/rndgate/eg/un/ra/mng.do |

**예시 필터 — 이 지시서가 예시적으로 탐지하게 한 기준이다:**

- 행안부·국토부 소관 공고는 키워드와 관계없이 전수 수집한다. 판별어: 행정안전부·국토교통부·행안부·국토부.
- 그 밖의 공고는 테스트베드 결합 조건을 만족할 때만 수집한다. 테스트베드어(실증·테스트베드·시범·수요조사·현장실증·공공실증·검증·오픈랩 중 하나)와 분야어(모빌리티·자율주행·교통·도로·주차·시설·안전·방재·스마트시티·스마트도시·인프라·지하·AX·AI·로봇·도시·공공·디지털트윈·C-ITS 중 하나)를 둘 다 포함하거나, 직접 판별어(AX실증·모빌리티·자율주행·스마트시티·교통실증·도로실증·시설안전 중 하나)를 포함할 때 수집한다.
- 의안·보고서 대상은 목록의 새 항목(ID 기준 차집합)을 새 소식으로 본다. 이미 본 항목의 사소한 문구 변동은 새 소식으로 알리지 않는다.

**예시 설정의 옛 표기 바로잡음:** SOP 원본에 있던 옛 분할 단위 표기는 이 지시서에서 쓰지 마라. 알림 발송은 이 지시서의 규격(본문 최대 200자, 190자 단위 분할)을 따른다. 4단계 스크립트의 분할 로직과 같다.

## 5. 실행 절차 — 카톡 구축부터 감시 연결, 탐지 알림까지

전체 흐름은 한 줄로 이렇다. **접수 → 콘솔·인가·토큰 → 스크립트·테스트 발송 → 감시 방식 승인 → 경량 엔진 구축 → (선택) Docker 배포 → 시뮬레이션 검증 → 인수인계 → 재설정.** 앞 단계 게이트를 통과하지 못하면 뒤로 가지 마라.

### 0단계: 접수 — 에이전트가 먼저 묻고, 작동자(사람)는 한 번에 답한다

1. 에이전트는 OP-01·OP-04·OP-06·OP-07·OP-10을 기본값과 함께 한 메시지로 묻는다. (OP-02·OP-03·OP-05·OP-08은 1·2단계 때 받는다. 미리 묻지 마라.)
2. 작동자(사람) 답을 받아 실행 계획을 확정한다. "기본값으로 진행"이면 OP-04는 localhost·링크 없음, OP-06은 현재 폴더, OP-07은 나에게 보내기만, OP-10은 경량 엔진만으로 확정한다.
3. OP-01이 "새로 만들겠다"면, 에이전트는 카카오디벨로퍼스에서 **내 애플리케이션**을 새로 만드는 일만 작동자(사람)에게 맡기고, 앱이 만들어진 뒤 1단계로 간다.

**게이트 G0:** OP-01·OP-04·OP-06·OP-07·OP-10 확정 전에는 1단계 안내를 시작하지 마라.

### 1단계: 콘솔 설정 — 작동자(사람)가 클릭하고, 에이전트가 값으로 검증한다

에이전트는 아래 순서대로 경로와 입력값을 작동자(사람)에게 준다. 작동자(사람)는 각 항목을 마친 뒤 결과 값(키·설정 상태)만 에이전트에게 전달한다. 에이전트는 콘솔 화면을 직접 보지 못하므로, 전달받은 값의 형식과 누락만 검증한다.

1. **REST API 키 확인** — 경로: `[앱] > [플랫폼 키] > [REST API 키]`. 표시된 키를 복사해 에이전트에게 전달한다. 에이전트는 빈 값·공백 포함 여부를 검증하고 `config.json` 재료로 보관한다. 이 키가 2·3단계의 `client_id`다.
2. **웹 도메인 등록(링크를 쓸 때만)** — 경로: `[앱] > [제품 링크 관리] > [웹 도메인]`. OP-04에서 도메인을 줬으면 등록한다. 도메인이 없으면 이 항목은 건너뛰고, 메시지 `link`를 빼는 것으로 확정한다. `https://localhost`는 인가 코드 받기용이지 메시지 링크용이 아니다.
3. **카카오 로그인 사용 설정 + 리다이렉트 URI 등록** — `[카카오 로그인] > [일반]`에서 사용 설정을 ON으로 켠다. 리다이렉트 URI는 `[앱] > [플랫폼 키] > [REST API 키] > [카카오 로그인 리다이렉트 URI]`에 `https://localhost`(또는 OP-04 값)를 등록하고 저장한다. 예전 경로인 `[카카오 로그인] > [일반]`에서 등록하는 것이 아니다. 요청 `redirect_uri`와 등록값은 프로토콜·도메인·경로·끝 슬래시까지 완전히 같아야 한다.
4. **Client Secret 처리** — 경로: `[앱] > [플랫폼 키] > [REST API 키] > [클라이언트 시크릿]`. REST API 키는 기본 Client Secret '사용함' 상태다. OP-03 방침대로 **[사용 안 함]으로 바꾸고 저장**하거나, 켜 둔 채로 Secret 코드를 에이전트에게 전달한다. 에이전트는 방침을 `config.json`의 `client_secret` 값(빈 문자열 또는 코드)에 반영한다.
5. **동의항목 설정** — 경로: `[카카오 로그인] > [동의항목]`. **카카오톡 메시지 전송(`talk_message`)** 항목을 '선택 동의' 또는 '이용 중 동의'로 설정한다.

**게이트 G1:** REST API 키 수신 + 리다이렉트 URI 저장 확인 + OP-03 방침 확정 + `talk_message` 설정 확인. 4가지가 모두 확인돼야 2단계로 간다. 하나라도 빠지면 빠진 항목만 다시 물어라.

### 2단계: 인가 코드 발급 — 에이전트가 URL을 만들고, 작동자(사람)가 코드만 복사한다

1. 에이전트는 OP-02 키와 OP-04 리다이렉트 URI로 아래 URL을 만들어 작동자(사람)에게 준다. `{REST_API_KEY}` 자리는 에이전트가 직접 채워서 완성된 URL 하나만 준다. 작동자(사람)가 키를 조립하지 않게 하라.

```text
https://kauth.kakao.com/oauth/authorize?client_id={REST_API_KEY}&redirect_uri=https://localhost&response_type=code&scope=talk_message
```

2. 작동자(사람)는 브라우저에서 접속해 로그인하고 [동의하고 계속하기]를 누른다. 빈 화면("페이지를 열 수 없음" 등)이 뜨는 것이 정상이다. 주소창의 `https://localhost/?code=...`에서 `code=` 뒤 문자열 전체를 복사해 에이전트에게 즉시 전달한다.
3. 에이전트는 코드가 비어 있지 않은지, `code=` 접두사나 공백이 섞이지 않았는지 검증한다. 인가 코드는 수 분 안에 만료되므로, 받자마자 3단계를 실행한다. 만료가 의심되면 2단계를 다시 하라고 안내한다(작동자(사람)가 URL 재접속 → 코드 재복사).

**게이트 G2:** 인가 코드 수신 즉시 3단계로 간다. 코드를 받고 다른 작업을 끼워 넣지 마라.

### 3단계: 토큰 교환 — 에이전트가 실행하고, 작동자(사람)는 기다린다

1. 에이전트는 인가 코드를 아래 요청으로 토큰과 교환한다. Client Secret '사용함'이면 `client_secret` 파라미터를 포함하고, '사용 안 함'이면 포함하지 않는다.

```bash
curl -s -X POST "https://kauth.kakao.com/oauth/token" \
  -H "Content-Type: application/x-www-form-urlencoded;charset=utf-8" \
  -d "grant_type=authorization_code" \
  -d "client_id={REST_API_KEY}" \
  -d "redirect_uri=https://localhost" \
  -d "code={복사한_인가코드}"
```

2. 성공 응답 JSON에 `access_token`, `refresh_token`, `scope`가 있는지 검증한다. `scope`에 `talk_message`가 없으면 1단계 5번 동의 설정부터 다시 확인하라고 작동자(사람)에게 알린다.
3. 응답 값을 `tokens.json`으로 저장한다(권한 600). 토큰 전문을 채팅에 출력하지 마라. `expires_in` 21599초 ≈ 액세스 토큰 6시간, `refresh_token_expires_in` 5183999초 ≈ 리프레시 토큰 2개월임을 인수인계 보고서에 적는다.
4. 갱신 시 새 리프레시 토큰이 돌아오면 반드시 새 값으로 덮어쓴다. 리프레시 토큰까지 만료되면 2단계부터 다시 해야 한다.

**게이트 G3:** `tokens.json` 저장 + `scope` 확인이 끝나야 4단계로 간다.

### 4단계: 스크립트 생성·테스트 — 에이전트가 파일을 만들고 끝까지 검증한다

1. 에이전트는 부록 A 그대로 `config.json`과 `send_memo.py`를 작업 폴더(OP-06)에 쓴다. 작동자(사람)에게 코드를 복사·붙여넣기 하라고 하지 마라. `config.json`의 `rest_api_key`·`client_secret`·`link_url`은 확정된 OP-02·OP-03·OP-04 값으로 채운다.
2. 스크립트 동작(에이전트가 보장해야 할 것): 190자 단위 분할(문단 기준, 긴 문단은 안에서 강제 절단), 401 발생 시 리프레시 토큰으로 자동 갱신 후 해당 조각 재전송, `link`는 등록 도메인이 있을 때만 포함. Python 3 표준 라이브러리만 쓴다.
3. 에이전트는 실행 전에 컴파일과 분할 테스트를 돌린다. 200자는 190+10, 250자는 190+60으로 나뉘고, 다시 합치면 원본과 같아야 한다. 실패하면 발송 테스트를 하지 마라.
4. 테스트 발송은 OP-09 문구로 1통만 보낸다. 에이전트는 API 응답을 확인하고, 작동자(사람)에게 폰의 '나와의 채팅' 도착 여부와 푸시 알림 여부를 물어 확인받는다. 푸시 알림 여부는 공식 문서로 확인된 사항이 아니므로, 작동자(사람) 확인 결과를 그대로 보고서에 적는다(추측하지 마라).

**게이트 G4:** 작동자(사람)의 "폰에서 도착 확인" 답을 받아야 카톡 구축이 완료다. API 성공 응답만으로는 완료라고 보고하지 마라.

### 5단계: 카톡 중간 보고 — 감시로 넘어가기 전에 작동자(사람)에게 알린다

에이전트는 카톡 구축이 끝났음을 한 보고서로 알리고, 감시는 아직 시작하지 않았다고 밝힌다. 토큰·키 전문은 포함하지 마라.

- 생성 파일 목록과 경로(`config.json`, `send_memo.py`, `tokens.json`), 실행 방법: `python3 send_memo.py "보낼 메시지"` 또는 `python3 send_memo.py < message.txt`
- 분할 규칙(190자 단위)과 자동 갱신 동작 요약
- 리프레시 토큰 만료일(2개월)과 만료 시 조치: 만료 전 갱신, 만료 후 2단계부터 재인가
- 테스트 발송 확인 결과(도착 여부·푸시 알림 여부 — 작동자(사람) 확인 그대로)
- 달라진 점이 있으면 실행 당일 공식 문서 대조 결과

### 6단계: 감시 방식 결정·승인 — 제안 카드부터, 승인 뒤에만 시작한다

1. 에이전트는 4절 예시 표와 OP-11·OP-12·OP-13·OP-14를 한 메시지로 보여주고, "예시 그대로 할지, 무엇을 바꿀지"를 묻는다. 이때 감시 방식(OP-10)의 자원 영향도 함께 적는다. 경량 엔진은 외부 패키지 없이 Python 표준 라이브러리만으로 돌고, Docker 배포는 서버·PC 자원과 포트 5000을 쓴다.
2. 작동자(사람) 답을 받아 감시 대상·키워드·주기·알림 채널·초기 스냅샷 방침을 확정한다. 답이 "예시 그대로"라도, 에이전트는 확정 표를 다시 보여주고 "이대로 시작해도 되겠습니까"라는 승인을 받는다.
3. OP-10이 "Docker까지" 또는 "둘 다"면, 8단계 배포도 같은 승인 안에 포함되는지 따로 확인받는다. 포함되지 않았으면 7단계까지만 진행한다.

**게이트 G5:** 감시 대상·키워드·주기·알림 채널 확정 + 작동자(사람) 승인 전에는 감시 파일을 만들지 마라.

### 7단계: Track 1 경량 엔진 구축 — 카톡 발송에 감시를 연결한다

1. 에이전트는 부록 B 그대로 디렉터리·`targets.json`·`monitor.py`·스냅샷 저장소를 만든다. `targets.json`은 6단계 확정값으로만 채운다. 예시 URL·키워드를 확정 없이 복사해 넣지 마라.
2. 감시 로직은 이 순서를 지킨다. ① 대상 페이지를 가져온다. ② 대상별 파서로 목록을 뽑는다. ③ 필터(4절 확정값)를 적용한다. ④ 직전 스냅샷의 ID 집합과 비교해 새 항목만 가려낸다. ⑤ 새 항목이 있을 때만 카톡으로 알린다. ⑥ 현재 목록을 새 스냅샷으로 저장한다.
3. 알림 문구는 한 건씩 짧게 만든다. 제목·출처·링크를 담고, 본문이 200자를 넘으면 190자 단위로 나눠 순차 발송한다. 분할은 `send_memo.py`와 같은 규칙을 쓴다. 여러 건이 한 번에 생기면 건수를 먼저 알리고, 건별로 나눠 보낸다. 링크는 대상 페이지 주소를 그대로 쓴다.
4. 초기 실행은 OP-14 방침대로 한다. 기본값은 알림 없이 현재 목록을 스냅샷으로 저장하고 종료하는 것이다. 스냅샷 파일이 없으면 "새 소식"으로 오인해 기존 항목을 한꺼번에 알리지 마라.
5. 정기 실행은 OP-12 주기로 등록한다. 기본값은 2시간마다 홀수 시각 정각(01:00, 03:00, …, 23:00)이다. 실행 환경의 스케줄 등록 방법은 환경에 맞는 방식을 쓰고, 등록한 뒤 다음 실행 시각을 보고서에 적는다.

**게이트 G6:** 초기 스냅샷 저장 + 무변동 실행 1회(메시지 없음) 확인이 끝나야 9단계 검증으로 간다.

### 8단계: Track 2 Docker 배포 — OP-10에서 고른 경우에만, 승인 뒤에 진행한다

OP-10이 "경량 엔진만"이면 이 단계는 열지 마라.

1. 에이전트는 OP-15 환경(Docker가 있는 서버·PC)을 확인받고, 부록 C의 `docker-compose.yml`을 배치한다. 공식 이미지 `changedetection.io`와 Playwright용 크롬 컨테이너를 함께 띄우고, 포트는 5000, 시간대는 Asia/Seoul로 둔다.
2. 배포 명령은 `docker compose up -d`다. 뜬 뒤에는 `http://localhost:5000`(서버 IP:5000)으로 접속해 대시보드를 확인하도록 작동자(사람)에게 안내한다.
3. 감시 설정은 공식 UI에서 한다. 감시할 사이트를 추가한 뒤 `Edit` > `Filters & Triggers`에서 CSS·XPath 필터로 원하는 부분만 골라 감시할 범위를 정하고, `Edit` > `Notifications`에서 알림 URL을 넣는다. 슬랙 웹훅을 쓰면 `slack://` 형식을, 카톡으로 보내려면 카카오 릴레이 웹훅 URL을 쓴다. 감시 주기는 공공기관 서버 부담을 고려해 2시간~하루 1회를 권한다.
4. Track 1과 Track 2가 같은 대상을 중복 감시하지 않게 한다. 둘 다 쓰는 경우, 대상을 나눠 담당을 정하고 중복 알림이 오지 않는지 확인한다. 중복이 생기면 한쪽을 끄거나 대상을 나눈다.

**게이트 G7:** 대시보드 접속 확인 + 필터·알림 설정 확인 + 중복 감시 분리 확인이 끝나야 검증으로 간다.

### 9단계: 검증 — 시뮬레이션으로 탐지·알림·침묵을 끝까지 확인한다

1. **무변동 확인:** `python3 scripts/changedetection/monitor.py`를 돌린다. 모든 대상이 무변동이면 메시지가 오지 않아야 한다. 메시지가 오면 스냅샷 비교 로직부터 다시 본다.
2. **신규 시뮬레이션:** `python3 scripts/changedetection/monitor.py --simulate`로 스냅샷에서 최신 항목 1개를 임시로 제외해 신규 등록 상황을 만든 뒤, 다시 실행해 1건이 카톡으로 실제로 오는지 확인한다. 작동자(사람)에게 폰 도착 여부를 확인받는다.
3. **카톡 단독 테스트:** `python3 scripts/kakao/send_message.py "테스트 메시지입니다."`로 발송 통로만 따로 확인한다. 이 경로는 부록 A 스크립트와 같은 토큰·분할 규칙을 쓴다.
4. 실패하면 6절 대응표대로 원인을 좁히고, 같은 요청을 반복하지 마라.

**게이트 G8:** 무변동 침묵 + 시뮬레이션 1건 발송 + 작동자(사람) 폰 확인이 모두 끝나야 완료 보고를 한다.

### 10단계: 최종 인수인계 — 에이전트가 보고서로 끝낸다

- 카톡 파일·감시 파일 목록과 경로, 정기 실행 주기와 다음 실행 시각
- 확정된 감시 대상·키워드·필터 표(6단계 승인본 그대로)
- 스냅샷 위치와 무변동 침묵 규칙
- 리프레시 토큰 만료일과 재인가 절차
- 재설정 방법(8절 요약)과, 바꿀 때 에이전트에게 물어볼 문장 예시
- Track 2를 쓴 경우 대시보드 주소와 중복 분리 내역

### 11단계: 확장 서비스 — OP-07에서 고른 경우에만, 별도 게이트로 진행한다

OP-07이 "나에게 보내기만"이면 이 단계는 열지 마라. 확장 서비스를 고른 경우, 에이전트는 부록 D의 조건·비용을 먼저 작동자(사람)에게 확인받고, 필요한 심사·채널·템플릿 신청은 작동자(사람) 승인을 받은 뒤에 진행한다. 에이전트는 작동자(사람) 승인 없이 비즈앱 전환·심사 신청·채널 개설을 시작하지 마라.

## 6. 오류 대응표

에이전트는 실패 시 아래 표대로 원인부터 좁힌다. 표에 없는 에러는 원문(`code`·`msg`)을 마스킹 없이(토큰 제외) 작동자(사람)에게 보여주고, 다음 확인 항목을 하나만 물어라.

| 증상 | 먼저 확인할 것 | 에이전트 조치 / 작동자(사람)에게 물을 것 |
|---|---|---|
| KOE006 | 등록값과 요청값의 `redirect_uri` 불일치 | 1단계 3번 등록값과 요청값을 끝 슬래시까지 대조. 다르면 작동자(사람)에게 재등록·재저장을 요청하고 2단계부터 다시 |
| KOE010 | Secret '사용함'인데 `client_secret` 누락·불일치 | 요청에 Secret 포함 여부를 확인. 포함했으면 코드 최신값 확인, 아니면 [사용 안 함] 전환 후 재요청을 작동자(사람)에게 확인 |
| 본문이 200자 초과 시 잘림·말줄임 | 텍스트 템플릿 한도 200자. 에러 없이 잘릴 수 있음 | 분할 테스트를 다시 돌리고, 발송 전 조각 수를 작동자(사람)에게 미리 알림 |
| 401 | 액세스 토큰 만료(6시간) | 스크립트 자동 갱신으로 처리. 갱신도 실패하면 리프레시 만료로 보고 2단계 재인가를 안내 |
| 링크 포함 시 발송 실패 | `link` 도메인 미등록 | 도메인 등록 또는 `link` 제외 후 테스트 발송으로 원인 분리. `localhost`는 링크용이 아님을 안내 |
| 감시에서 매번 알림이 옴 | 스냅샷 비교·필터 범위 문제 | ID 비교인지, 필터가 너무 넓은지 확인. 스냅샷이 매번 갱신되는지 보고, 무변동 실행으로 재확인 |
| 감시에서 새 소식을 놓침 | 파서·페이지 구조 변경 | 대상 페이지를 직접 확인해 목록 구조가 바뀌었는지 보고, 파서를 고친 뒤 시뮬레이션으로 재검증 |
| Docker 대시보드가 안 뜸 | 컨테이너 상태·포트 충돌 | 컨테이너 실행 상태와 5000번 포트 사용 여부를 확인하고, 충돌 시 포트·배포 위치를 작동자(사람)에게 확인 |

## 7. 검증 체크리스트

에이전트는 완료 보고 전에 아래를 직접 실행하고 통과 여부를 적는다. 실행하지 않은 항목은 "미확인"이라고 적는다.

- [ ] `config.json`에 빈 `rest_api_key`가 없고, `client_secret`이 OP-03 방침과 일치한다.
- [ ] `tokens.json`의 `scope`에 `talk_message`가 포함된다.
- [ ] `send_memo.py` 컴파일 통과, 200자·250자 분할 테스트 통과.
- [ ] 테스트 메시지 1통 발송, API 응답 성공, 작동자(사람) 폰 도착 확인 완료.
- [ ] `config.json`·`tokens.json` 권한 600, 키·토큰 전문이 대화·보고서에 노출되지 않았다.
- [ ] `targets.json`이 6단계 승인본과 일치하고, 초기 스냅샷이 저장됐다.
- [ ] 무변동 실행에서 메시지가 오지 않고, 시뮬레이션에서 1건이 카톡으로 도착했다.
- [ ] 정기 실행 주기와 다음 실행 시각이 보고서에 적혔다.
- [ ] Track 2를 쓴 경우 중복 감시 분리가 확인됐다.

## 8. 받는 사람이 바꾸는 방법 — 재설정

이 지시서를 건네받은 사용자는, 구축이 끝난 뒤에도 에이전트에게 말 한 마디로 설정을 바꿀 수 있다. 에이전트는 아래 순서로만 재설정한다. 작동자(사람)가 바꾸라고 한 값을 추측으로 채우지 마라.

**에이전트가 먼저 물어보는 것:** 재설정을 맡기면, 에이전트는 바꾸기 전에 "지금 설정은 이렇습니다. 무엇을 어떻게 바꾸겠습니까?"라고 현재 표(대상·키워드·주기·알림 채널)를 보여준다. 사용자가 답하면, 에이전트는 변경 전·후 표를 보여주고 승인을 받는다. 승인 뒤에야 파일을 고친다.

**바꿀 수 있는 것 — 물어보는 말 예시:**

- 감시 대상 바꾸기: "감시 대상에서 IRIS를 빼줘." / "○○ 공고 페이지를 새로 추가해줘. 주소는 …야." / "시의회 업무보고자료만 남기고 나머지는 꺼줘."
- 키워드·필터 바꾸기: "키워드에 '○○'를 더해줘." / "행안부 전수 수집을 끄고, 키워드 조건으로만 모아줘." / "이 단어가 들어간 공고는 빼줘."
- 주기 바꾸기: "감시 주기를 4시간마다로 바꿔줘." / "하루 1번, 아침에만 확인해줘."
- 알림 채널 바꾸기: "알림을 카톡 말고 슬랙으로도 보내줘." / "카톡 알림은 끄고 슬랙만으로 바꿔줘."
- 알림 문구·건수 바꾸기: "알림은 제목만 짧게 보내줘." / "한 번에 여러 건이 오면 건수만 먼저 알려줘."

**재설정 절차(에이전트가 지킬 것):**

1. 사용자의 말을 그대로 받아, 바꿀 항목·바꾸기 전 값·바꾼 뒤 값을 표로 정리해 보여주고 승인을 받는다. 대상을 새로 추가하면 URL과 파서 종류도 함께 확인받는다.
2. 승인 뒤 `targets.json`과 필터(부록 B 로직)를 고치고, 필요하면 스냅샷을 새로 저장한다. 대상을 뺄 때는 해당 스냅샷 파일도 함께 정리할지 물어본다.
3. 주기를 바꾸면 정기 실행 등록을 다시 하고, 다음 실행 시각을 알린다. 알림 채널을 바꾸면 새 채널로 테스트 발송 1건을 보내고 도착 확인을 받는다.
4. 바꾼 뒤에는 무변동 실행 1회와 시뮬레이션 1회로 다시 검증하고, 결과를 "바뀐 설정 표 + 검증 결과"로 보고한다. 검증 없이 "바꿨다"고만 보고하지 마라.

## 부록 A. 에이전트가 생성할 파일 — 카카오톡 발송

에이전트는 아래 파일을 작동자(사람)에게 "복사해 달라"고 하지 않고 직접 쓴다. `link_url`은 등록 도메인이 없으면 빈 문자열로 둔다.

`config.json`:

```json
{
  "rest_api_key": "REST_API_KEY",
  "client_secret": "",
  "link_url": "https://제품-링크-관리에-등록한-도메인"
}
```

`send_memo.py` (아래 전체를 하나의 파일로 쓴다):

```python
#!/usr/bin/env python3
"""카카오톡 '나에게 보내기' 발송 스크립트
- 액세스 토큰 만료(401) 시 리프레시 토큰으로 자동 갱신 후 재전송
- 텍스트 템플릿 한도 200자에 맞춰 190자 단위로 분할
  (문단 하나가 190자를 넘어도 안에서 다시 자름)
사용법:
  python3 send_memo.py "보낼 메시지"
  python3 send_memo.py < message.txt
"""
import json
import os
import sys
import time
import urllib.error
import urllib.parse
import urllib.request

SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
CONFIG_FILE = os.path.join(SCRIPT_DIR, "config.json")
TOKENS_FILE = os.path.join(SCRIPT_DIR, "tokens.json")

TOKEN_URL = "https://kauth.kakao.com/oauth/token"
SEND_URL = "https://kapi.kakao.com/v2/api/talk/memo/default/send"
MAX_LEN = 190  # 공식 한도 200자에서 여유를 둔 분할 단위
```

```python
def load_json(path):
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)


def save_json(path, data):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)


def refresh_tokens():
    config = load_json(CONFIG_FILE)
    tokens = load_json(TOKENS_FILE)
    params = {
        "grant_type": "refresh_token",
        "client_id": config["rest_api_key"],
        "refresh_token": tokens["refresh_token"],
    }
    if config.get("client_secret"):
        params["client_secret"] = config["client_secret"]
    data = urllib.parse.urlencode(params).encode("utf-8")
    req = urllib.request.Request(
        TOKEN_URL,
        data=data,
        headers={"Content-Type": "application/x-www-form-urlencoded;charset=utf-8"},
    )
    try:
        with urllib.request.urlopen(req) as resp:
            new_tokens = json.loads(resp.read().decode("utf-8"))
    except Exception as e:
        print(f"Token refresh failed: {e}", file=sys.stderr)
        return tokens
    if "access_token" in new_tokens:
        tokens["access_token"] = new_tokens["access_token"]
    if "refresh_token" in new_tokens:
        tokens["refresh_token"] = new_tokens["refresh_token"]
    save_json(TOKENS_FILE, tokens)
    return tokens
```

```python
def _send_single(text, access_token, link_url):
    template = {"object_type": "text", "text": text, "button_title": "확인"}
    # link는 등록된 도메인이 있을 때만 포함한다.
    # 등록된 도메인이 없으면 link를 빼고 테스트 발송으로 확인한다.
    if link_url:
        template["link"] = {"web_url": link_url, "mobile_web_url": link_url}
    body = urllib.parse.urlencode(
        {"template_object": json.dumps(template, ensure_ascii=False)}
    ).encode("utf-8")
    req = urllib.request.Request(
        SEND_URL,
        data=body,
        headers={
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/x-www-form-urlencoded;charset=utf-8",
        },
    )
    try:
        with urllib.request.urlopen(req) as resp:
            return json.loads(resp.read().decode("utf-8"))
    except urllib.error.HTTPError as e:
        res_body = e.read().decode("utf-8")
        try:
            parsed = json.loads(res_body) if res_body else {}
        except json.JSONDecodeError:
            parsed = {"raw": res_body}
        return {"error": e.code, "body": parsed}
```

```python
def split_text_cleanly(text, max_len=MAX_LEN):
    """문단(\n\n) 단위로 나누되, 긴 문단은 안에서 강제 절단한다."""
    if len(text) <= max_len:
        return [text]
    chunks = []
    current = ""
    for paragraph in text.split("\n\n"):
        pieces = (
            [paragraph[i : i + max_len] for i in range(0, len(paragraph), max_len)]
            if len(paragraph) > max_len
            else [paragraph]
        )
        for piece in pieces:
            if not current:
                current = piece
            elif len(current) + 2 + len(piece) <= max_len:
                current += "\n\n" + piece
            else:
                chunks.append(current)
                current = piece
    if current:
        chunks.append(current)
    return chunks


def send_memo(text):
    config = load_json(CONFIG_FILE)
    tokens = load_json(TOKENS_FILE)
    link_url = (config.get("link_url") or "").strip()
    chunks = split_text_cleanly(text)
    results = []
    for i, chunk in enumerate(chunks):
        res = _send_single(chunk, tokens["access_token"], link_url)
        if res.get("error") == 401 or res.get("code") in (-401, 401):
            tokens = refresh_tokens()
            res = _send_single(chunk, tokens["access_token"], link_url)
        results.append(res)
        if len(chunks) > 1 and i < len(chunks) - 1:
            time.sleep(0.5)
    return results


if __name__ == "__main__":
    if len(sys.argv) > 1:
        message = " ".join(sys.argv[1:])
    else:
        message = sys.stdin.read().strip()
    if not message:
        message = "카카오톡 나에게 보내기 테스트 메시지입니다!"
    print(json.dumps(send_memo(message), ensure_ascii=False, indent=2))
```

## 부록 B. 에이전트가 생성할 파일 — Track 1 경량 감시 엔진

외부 패키지 설치 없이 Python 3 표준 라이브러리(`urllib`, `re`, `json`)만으로 구동한다.

**디렉터리 구성:**

```text
workspace/
├── scripts/
│   ├── changedetection/
│   │   ├── monitor.py           # 핵심 감시 및 파싱 엔진 (엔트리포인트)
│   │   ├── targets.json         # 감시 대상 목록 설정 (URL, 파서 타입, 알림 설정)
│   │   └── snapshots/           # 타깃별 직전 베이스라인 JSON 스냅샷 저장소
│   │       ├── smc_reception.json
│   │       ├── smc_review.json
│   │       ├── smc_report.json
│   │       ├── iris_testbed.json
│   │       └── ntis_testbed.json
│   └── kakao/
│       ├── send_message.py      # 카카오톡 나와의 채팅방 토큰 갱신 & 분할 발송 모듈
│       ├── config.json          # REST API 키 설정
│       └── tokens.json          # 액세스/리프레시 토큰 저장소
└── scheduled-actions/
    └── 🏛️-시의회-교통위원회-및-국가r&d-웹-감시/
        └── action.md            # 2시간 정기 감시 스케줄 정의서
```

스냅샷 파일 이름은 6단계에서 확정한 대상 ID를 따른다. 대상이 바뀌면 스냅샷 이름도 함께 바꾼다.

**`targets.json` (예시 형태 — 6단계 확정값으로 채운다):**

```json
[
  {
    "id": "smc_reception",
    "name": "서울시의회 교통위원회 접수의안",
    "url": "https://www.smc.seoul.kr/comt/information.do?menuId=015004001",
    "parser": "smc_bills",
    "notify_kakao": true,
    "enabled": true
  },
  {
    "id": "smc_review",
    "name": "서울시의회 교통위원회 검토보고서",
    "url": "https://www.smc.seoul.kr/comt/information.do?menuId=015004003",
    "parser": "smc_bills",
    "notify_kakao": true,
    "enabled": true
  },
  {
    "id": "smc_report",
    "name": "서울시의회 교통위원회 업무보고자료",
    "url": "https://www.smc.seoul.kr/board/BoardList.do?boardTypeId=98&menuId=015005002",
    "parser": "smc_reports",
    "notify_kakao": true,
    "enabled": true
  },
  {
    "id": "iris_testbed",
    "name": "범부처통합연구지원시스템(IRIS) 테스트베드 R&D 공고",
    "url": "https://www.iris.go.kr/contents/retrieveBsnsAncmListView.do",
    "parser": "iris_testbed",
    "notify_kakao": true,
    "enabled": true
  },
  {
    "id": "ntis_testbed",
    "name": "국가과학기술지식정보서비스(NTIS) 테스트베드 R&D 공고",
    "url": "https://www.ntis.go.kr/rndgate/eg/un/ra/mng.do",
    "parser": "ntis_testbed",
    "notify_kakao": true,
    "enabled": true
  }
]
```

**핵심 필터 로직 (`monitor.py`에 넣는다 — 키워드는 6단계 확정값으로 바꾼다):**

```python
def is_priority_ministry(ministry_text):
    """행정안전부·국토교통부 소관 여부 확인 (키워드 불문 전수 수집 — 예시 방침)"""
    m = (ministry_text or "").lower()
    return any(k in m for k in ["행정안전부", "국토교통부", "행안부", "국토부"])

def check_testbed_match(title):
    """테스트베드 연계 가능성 판별 (예시 결합 조건)"""
    title_lower = title.lower()
    kw_testbed = ["실증", "테스트베드", "testbed", "시범", "수요조사", "현장실증", "공공실증", "검증", "오픈랩"]
    kw_domain = [
        "모빌리티", "자율주행", "교통", "도로", "주차", "시설", "안전", "방재", "스마트시티",
        "스마트도시", "인프라", "지하", "ax", "ai", "로봇", "도시", "공공", "디지털트윈", "c-its"
    ]
    has_tb = any(k in title_lower for k in kw_testbed)
    has_dom = any(k in title_lower for k in kw_domain)
    direct = any(k in title_lower for k in ["ax실증", "모빌리티", "자율주행", "스마트시티", "교통실증", "도로실증", "시설안전"])
    return (has_tb and has_dom) or direct
```

**스냅샷 비교 규칙:** 파일이 없으면 최신 목록을 저장하고 알림 없이 종료한다. ID 기준 차집합(`new_items = [it for it in current if it['id'] not in prev_ids]`)으로 새 항목만 가린다. 변동 0건이면 어떤 메시지도 보내지 않고 종료한다.

**카카오 연동 규칙:** `send_message.py`는 부록 A와 같은 토큰 갱신·190자 분할 규칙을 쓴다. 401을 만나면 리프레시 토큰으로 갱신한 뒤 해당 조각을 다시 보낸다.

**스케줄러 등록 (`action.md`):** 실행 환경의 스케줄 등록 방식을 따르고, 기본 주기는 매 2시간 간격 홀수 시각 정각(01:00, 03:00, …, 23:00)이다. 등록 뒤 다음 실행 시각을 보고서에 적는다.

## 부록 C. Track 2 Docker 배포 파일

경로: `references/changedetection/docker-compose.yml`

```yaml
version: '3.8'

services:
  # 1. changedetection.io 메인 웹 서버
  changedetection:
    image: ghcr.io/dgtlmoon/changedetection.io:latest
    container_name: changedetection
    hostname: changedetection
    volumes:
      - changedetection-data:/datastore
    environment:
      # Playwright 크롬 브라우저 컨테이너 연동 (동적 렌더링/SPA 지원)
      - PLAYWRIGHT_DRIVER_URL=ws://playwright-chrome:3000
      # 기본 타임존 설정 (한국 표준시)
      - TZ=Asia/Seoul
    ports:
      - "5000:5000"
    restart: unless-stopped
    depends_on:
      - playwright-chrome

  # 2. 자바스크립트 동적 로딩(Playwright) 전용 헤드리스 크롬 컨테이너
  playwright-chrome:
    image: dgtlmoon/sockpuppetbrowser:latest
    container_name: playwright-chrome
    hostname: playwright-chrome
    environment:
      - SCREEN_WIDTH=1920
      - SCREEN_HEIGHT=1020
      - SCREEN_DEPTH=16
      - MAX_CONCURRENT_CHROME_PROCESSES=10
    restart: unless-stopped

volumes:
  changedetection-data:
```

배포는 `docker compose up -d`로 하고, 접속은 `http://localhost:5000`이다.

## 부록 D. 확장 서비스 선택 가이드 (11단계 전용)

에이전트는 이 부록을 기본 구축에 섞지 마라. OP-07에서 작동자(사람)가 고른 서비스만 다루고, 고르기 전에는 조건·비용을 먼저 보여주고 승인을 받는다. 아래 금액은 모두 부가세(VAT) 별도 기준이다. 알림톡·브랜드메시지는 카카오가 직접 파는 상품이 아니라 공식 딜러사(발송 대행사)를 통해 계약하는 구조라, 딜러사·발송 물량·플랜에 따라 단가가 달라진다. 견적을 받을 때는 "성공 건수 기준인지, 실패 시 문자 대체발송 비용이 포함되는지"를 먼저 확인하라.

- **본인만 보면 된다 → 나에게 보내기.** 받는 사람: 로그인한 본인 1명. 시작 조건 없음, 별도 심사 없음. 비용 무료. 단, 일간·월간 쿼터(제공량)가 적용된다. 무료 제공 쿼터 범위와 현재 사용량은 콘솔의 통계·쿼터 화면에서 확인한다.
- **친구·동료 몇 명과 주고받는다 → 친구에게 보내기.** 받는 사람: 같은 앱에 로그인·동의한 카카오톡 친구. 피커 또는 친구 목록 조회로 받은 `uuid`로 지정하며, **1회 최대 5명**까지 보낼 수 있다. 시작 조건: **비즈앱 전환 + 추가 기능 심사** 필요. 발송자와 수신자 모두 `friends`와 `talk_message` 동의 필요. 심사 전에는 앱 기준 일 30건까지만 테스트 발송 가능. 비용 무료. 단, 심사 승인 후에도 일간·월간 쿼터가 적용된다(앱 기준 일 3만 건, 발신자당·수신자당 일 100건, 같은 발신자·수신자 쌍은 일 20건 등). 주의: 메시지 API는 **사용자 간 소셜(상호작용) 기능** 용도다. 서비스가 사용자에게 일방적으로 알림을 보내는 용도로 신청하면 심사에서 반려될 수 있다. 일방 알림이 목적이면 알림톡을 검토하라. 엔드포인트: `POST https://kapi.kakao.com/v1/api/talk/friends/message/default/send`

- **모르는 많은 사람에게 공식 안내를 보낸다 → 알림톡 (카카오 비즈니스).** 받는 사람: 전화번호를 아는 불특정 다수. 카카오톡 친구 추가 여부와 관계없이 보낼 수 있다. 시작 조건: **비즈니스 채널 개설 + 템플릿 사전 승인** 필요. 정보성 메시지(주문·예약·배송·안내 등)만 보낼 수 있고, 광고성 내용은 보낼 수 없다. 비용: 공식 정산단가 **8원/건**, 딜러사 판매가 **6.5~13원대/건**. VAT 별도, **성공 건수 기준** 과금. 단순 계산 예시: 1만 건이 전부 성공하면 공식 단가 기준으로 8만 원(VAT 별도). 실패 시 대체발송 비용: SMS 약 15원/건, LMS 25~40원/건 수준으로 별도 과금된다.
- **홍보·소식을 알린다 → 브랜드메시지 (구 친구톡).** **친구톡은 2025년 12월 31일에 종료**됐고, **2026년 1월 1일부터 브랜드메시지**로 전환됐다. 예전 '친구톡' 안내를 보고 준비하면 안 된다. 받는 사람: 카카오톡 채널을 추가한 이용자. 광고성·홍보 메시지를 보낼 수 있다. 시작 조건: 비즈니스 채널 필요. 알림톡과 달리 템플릿 사전 심사는 필요 없는 구조이지만, 광고성 정보 수신 동의 등 발송 규칙을 지켜야 한다. 비용: 텍스트형 약 **20~24원/건**, 자유형 **100원/건**. VAT 별도. 크레딧(선불)·후불, 플랜·딜러사에 따라 달라진다. 이미지·형식에 따라 단가가 달라질 수 있으니, 쓸 형식을 정한 뒤 딜러사 견적으로 확정하라.
- **이용자가 물으면 답한다 → 챗봇 (챗봇 관리자센터).** 지금 이름은 **챗봇 관리자센터**다. 예전 이름인 '카카오 i 오픈빌더'로 검색하면 같은 도구를 가리킨다. 받는 사람·쓰는 곳: 카카오톡 채널을 방문한 이용자와 대화한다. **개인 카톡 계정이나 단체 채팅방에서는 쓸 수 없고, 채널 전용**이다. 채널을 개설한 뒤 챗봇을 만들고 배포한다. 비용: 일반 상품 **무료**. 챗봇 대화 자체(일반 상품)는 무료다. 다만 스킬 서버 등 외부 시스템 연동 비용은 별도다. 실시간 데이터를 조회해 답하는 봇을 만들면, 그 백엔드(서버·AI 사용료 등) 비용이 따로 든다. 단방향 알림이 아니라, 이용자가 먼저 말을 걸어야 대화를 시작한다.

## 부록 E. 확인한 공식 문서

에이전트는 실행 당일 아래 문서를 대조한다. 바로잡은 핵심은 텍스트 한도 200자, 리다이렉트 URI 등록 위치, 메시지 링크 도메인 주의, 친구톡 종료·브랜드메시지 전환이다.

- [카카오톡 메시지 · REST API — 나에게/친구에게 발송 엔드포인트와 요구 사항](https://developers.kakao.com/docs/ko/kakaotalk-message/rest-api)
- [메시지 템플릿 · 기본 템플릿 — 텍스트 템플릿 최대 200자](https://developers.kakao.com/docs/ko/message-template/default)
- [앱 설정 · 앱 — 플랫폼 키, 리다이렉트 URI, 클라이언트 시크릿 기본 활성화](https://developers.kakao.com/docs/ko/app-setting/app)
- [카카오톡 메시지 · 이해하기 — 친구 발송 심사·쿼터·소셜 용도 안내](https://developers.kakao.com/docs/ko/kakaotalk-message/common)
- [이용 안내 · 쿼터 — 일간·월간 무료 제공 쿼터](https://developers.kakao.com/docs/ko/getting-started/quota)
- [카카오 로그인 · 에러 코드 — KOE006·KOE010 원인](https://developers.kakao.com/docs/ko/kakaologin/trouble-shooting)
- [changedetection.io — GitHub 저장소](https://github.com/dgtlmoon/changedetection.io)
- [changedetection.io — 공식 사이트](https://changedetection.io)
