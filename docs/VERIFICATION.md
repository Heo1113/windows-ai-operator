# 검증 결과

작성 기준: 브랜치 `claude/nifty-pascal-8te367`. 개발 환경은 Linux 컨테이너라 Windows GUI를 직접 실행할 수 없어, 검증을 세 곳으로 나눴습니다.

| 어디서 | 무엇을 | 성격 |
|---|---|---|
| Linux(개발 컨테이너·CI ubuntu-latest) | 전체 빌드(WPF XAML 컴파일 포함), 단위·회귀 테스트 150개 | 모의 데스크톱(MOCK) — 실제 GUI 아님 |
| Windows CI(GitHub Actions windows-latest, 대화형 데스크톱) | 같은 단위 테스트, **실제 UIA 테스트 11개**, 포터블 패키지, 게시된 EXE의 GUI 스모크 테스트 | 실제 Windows UI Automation (100% 배율 가상 머신, 영어 Windows) |
| 사용자 PC | 아래 7절 | **아직 하지 않음** |

## 1. 단위·회귀 테스트 (Linux와 Windows에서 동일하게 실행)

`dotnet test --project tests/WindowsAiOperator.Tests` — 150개, 모두 통과.

| 묶음 | 내용 |
|---|---|
| ExecutorTests (29) | 클릭·입력·확인란·승인 거부/승인/시간 초과·승인 중 대상 변경·허용 목록 밖 앱·차단 프로그램·자기 창·비밀번호 칸·읽기 전용 앱·사라진 요소·앱 종료·대화상자·예상 밖 창 일시정지·중단·일시정지/재개·단계 한도·작업 시간 한도·연속 실패 자동 중단·계획 확인 거부·화면 주입 문구·observe 재계획·메모장·메뉴·단축키 표·비전 좌표 재검증·폴더 목록 |
| PlanParserTests, HotkeyTests | 추론 블록·코드 펜스 허용, 정의되지 않은 동작·스키마 밖 필드·필수 필드·타입·Windows 키·Alt+Tab·비전 없는 좌표·잘못된 ref·컨트롤 형식·중복 id·잘린 JSON·평문 응답 거부, 단계 수·텍스트 길이 상한 |
| PolicyEngineTests, SecurityTests | 위험 키워드 분류(한/영), 결제·관리자 권한 거부, 단어 경계, 끌 수 없는 규칙, 텍스트 규칙(비밀번호·민감값·줄바꿈·기존 내용 대체), 앱 수준 규칙, 실행 허용, 비밀정보 가림, 로그에 비밀정보 없음, 주입 문구 탐지 |
| CoordinateTests | 100/150/175/225% 논리↔물리 변환, 왼쪽 모니터(음수 좌표), 4K 175% 창 매핑, 이동·DPI 변경·오래된 스크린샷·창 밖·다른 창이면 클릭 거부 |
| ModelProviderTests | Strata 모델 목록·`/v1/status` 비전·구조화 출력 읽기, 연결 실패·401, 텍스트 전용 모델에 이미지 차단, json_schema 거부 시 폴백·기억, 리디렉션 거부, 외부 서버 동의·https, 키는 헤더에만, 추론만 한 응답, 이미지 테스트(두 색 모두), 잘못된 출력·오프라인 모델이 데스크톱에 닿지 않음, 자연어→계획→실행 전체(MOCK), 응답 형식 검사, 프롬프트의 신뢰하지 않는 데이터 감싸기 |
| StorageTests | 단계·이벤트 기록과 비밀정보 가림, 비정상 종료 복구(부분 완료/중단/불확실/미실행), 백업 검사·복원 전 안전 백업, 새 버전 DB 쓰기 금지, 보존 기간 |
| WorkflowTests, EvaluationTests, ScreenshotTests | 기본 워크플로 검증·왕복, 매개변수, 성공 작업→워크플로(ref를 안정 선택자로), 승인 힌트, 계획 유효와 GUI 성공을 나눈 모델 비교, 스크린샷 저장 설정 |
| PresentationTests (8) | 8개 화면, 자연어 작업→기록, 승인 창 경로 거부, 워크플로, 창 검사기 직접 조작, 안전 설정 검증, 모델 확인·비교, 백업·무결성 |

## 2. 실제 Windows UI Automation 테스트 (Windows CI)

`dotnet test --project tests/WindowsAiOperator.UiaTests` — 별도 테스트 앱(`OperatorTestTarget.exe`)과 메모장을 실제 UIA3(FlaUI)로 조작. **11개 모두 통과**
(CI 실행 #3, #5 이후). 처음 실행에서 실패한 2개(WPF 콤보 항목이 UIA에 두 번 노출, 여러 줄 입력의 줄바꿈 규칙)를 고쳐 통과했습니다.

| 테스트 | 확인한 것 |
|---|---|
| Lists_the_window_and_reads_its_structure | 창 열거(프로세스·DPI·경계·권한), UIA 트리, 패턴, 확인란 상태, 비밀번호 칸 값 미수집 |
| Clicks_types_toggles_selects_and_verifies | Invoke·ValueSetValue·덧붙이기(여러 줄)·Toggle·SelectionItem(라디오·목록)·ScrollPattern, 결과 검증, 읽은 값 보고 |
| Combo_box_expand_and_select | 콤보 펼치기·항목 선택(중복 노출 처리) |
| Menu_opens_a_dialog_that_is_handled_with_approval | 메뉴 경로 → 모달 대화상자 → 다른 창 지정 → 확인 버튼 승인 → 결과 확인 |
| Denied_delete_leaves_the_list_untouched | 승인 거부 시 실행 안 함(목록 5개 그대로) |
| Vanished_element_and_closed_app_are_reported | 사라진 요소(ElementNotFound), 실행 중 앱 종료(WindowClosed) |
| Programs_outside_the_allow_list_and_password_fields_are_refused | 허용 목록 밖 앱 차단(카운트 그대로), 비밀번호 칸 입력 거부 |
| Injected_screen_text_forces_approval | 화면의 주입 문구 탐지 → 모든 변경 승인 요구 |
| Safe_hotkey_reaches_the_focused_window | 창 활성화 후 SendInput 단축키 |
| Window_capture_returns_a_png_with_bounds_and_dpi | 창 캡처(PNG, 경계, DPI) — 비전 보조의 전제 |
| Notepad_launch_and_type_without_saving | 허용 앱 실행 → 메모장 문서에 덧붙이기(저장 안 함, 테스트가 시작한 프로세스만 종료) |

## 3. GUI 스모크 테스트 (게시된 EXE, Windows CI)

`scripts/smoke-gui.ps1` — 결과는 CI 아티팩트 `smoke-test-evidence`(화면별 스크린샷, JSON 보고서, 로그).

1. **mock**: MOCK 데스크톱 + MOCK 모델. 8개 화면 방문·스크린샷, 바인딩 오류·UI 예외 0개 확인, 자연어 요청(메모장) → 계획 확인 창 2회 → 완료,
   삭제 단계 → 승인 창 → 거부 → 실행 안 됨(부분 완료), 백업·무결성 검사.
2. **real-uia**: 실제 데스크톱 + MOCK 모델. 기본 워크플로(테스트 앱 실행·클릭·입력·읽기) 완료, 자연어 요청으로 테스트 앱 확인란 읽기,
   창 검사기에서 테스트 앱 UIA 트리 표시.
3. **after-kill**: 긴 작업 실행 중 프로세스 강제 종료 → `wao status`가 미완료 작업 1개·무결성 ok 확인 → 다음 실행에서 '부분 완료'로 정리, 자동 재실행 없음.

결과: 아래 6절.

## 4. 필수 검증 시나리오 대응

| # | 시나리오 | 근거 | 상태 |
|---|---|---|---|
| 1 | UI 요소 탐색 | UIA: Lists_the_window…, 스모크 real-uia 창 검사기 | 실제 UIA로 확인 |
| 2 | 안전한 버튼 클릭 | UIA: Clicks_types…(InvokePattern) | 실제 UIA로 확인 |
| 3 | 테스트 앱에 텍스트 입력 | UIA: Clicks_types…, Notepad_launch… | 실제 UIA로 확인 |
| 4 | 앱이 중간에 종료 | UIA: Vanished_element_and_closed_app… | 실제 UIA로 확인 |
| 5 | 대상 창 변경 | UIA: Menu_opens_a_dialog…, 단위: 예상 밖 창 일시정지·승인 중 대상 변경 | 실제 UIA(대화상자 전환) + 단위(예상 밖 창) |
| 6 | UI 요소가 사라짐 | UIA: Vanished_element… | 실제 UIA로 확인 |
| 7 | 잘못된 모델 출력 거부 | 단위: PlanParserTests, Bad_model_output… | 단위(모델 응답은 가짜 HTTP 서버) |
| 8 | 허용되지 않은 앱 조작 차단 | UIA: Programs_outside_the_allow_list… | 실제 UIA로 확인 |
| 9 | 승인 거부 시 아무것도 실행 안 함 | UIA: Denied_delete…, 스모크 mock 승인 창 | 실제 UIA로 확인 |
| 10 | 실행 도중 사용자 중단 | 단위: Stop_ends…, Pause_holds…, 승인 창 닫기=거부 | 단위(실제 GUI에서 중단 버튼 클릭은 미검증) |
| 11 | 비정상 종료 후 복구 | 단위: Interrupted_tasks…, 스모크 after-kill | 게시된 EXE로 확인(6절) |
| 12 | 로컬 모델 서버 연결 실패 | 단위: Probe_reports_connection_failure, Offline 모델 | 단위 |
| 13 | 비전 미지원 모델 처리 | 단위: Text_only_model…, Vision_test…, click_point 거부 | 단위 |
| 14 | 다른 DPI 좌표 처리 | 단위: CoordinateTests | 수식만 확인. **CI는 100% 고정 — 175%·다중 모니터 실기 미검증** |
| 15 | DB 기록·백업 | 단위: StorageTests, 스모크 백업·무결성 | 확인 |
| 16 | 민감정보 로깅 방지 | 단위: Logs_never_contain_secrets, Journal…redacts, Api_key…header, Prompt… | 단위 |

## 5. 알려진 한계

- Windows 11 새 메모장(탭, 패키지 앱)에서 `launch_app notepad`는 기존 창에 새 탭으로 열릴 수 있어, 이 경우 "기존 창 사용"으로 처리합니다. CI 러너(Windows Server)의 메모장과 동작이 다를 수 있습니다.
- Chromium/Electron 앱은 접근성 트리가 꺼져 있으면 '구조적 인식 불가'입니다. 비전 보조는 이미지 입력이 확인된 모델에서만 켜지며, 좌표 클릭은 항상 승인을 받습니다.
- 위험 판단은 요소 이름·AutomationId 키워드에 기대므로, 이름을 속이는 앱은 막지 못할 수 있습니다. 이런 앱은 허용 목록에 넣지 않거나 '모든 조작 승인'을 켜세요.
- UWP 앱(계산기·설정)은 창이 최소화되어 있으면 실제 앱 프로세스를 확인하지 못해 차단될 수 있습니다(허용 오류보다 차단 쪽으로 동작).
- 승인 창이 앞에 뜰 때 포커스를 가져가므로, 승인 뒤 단축키·좌표 단계는 대상 창을 다시 앞으로 가져온 뒤 실행합니다.

## 6. CI 실행 결과

(아래는 CI 결과를 확인한 뒤 갱신합니다.)

## 7. 사용자 PC에서 확인할 것 (미검증)

| 항목 | CI에서 못 한 이유 | 확인 방법 |
|---|---|---|
| 4K·175% 배율, 다중 모니터·서로 다른 배율 | CI 화면은 100% 단일 모니터 | 창 검사기에서 DPI·배율 표시 확인, 테스트 앱을 다른 모니터로 옮겨 워크플로 실행, [화면에서 강조 표시]가 요소 위에 맞게 그려지는지 |
| 실제 Strata 모델로 계획·실행 | 개발 환경에 모델 없음 | 모델 화면 [연결 확인]·[응답 형식 검사]·[이미지 입력 테스트], 작업 실행 예시 요청 |
| 한국어 Windows의 메모장·설정 앱 이름 | CI는 영어 Windows | "메모장을 열고 …" 요청, "설정 창에서 현재 선택된 항목을 확인해 줘" |
| Windows 11 새 메모장 | CI는 Windows Server | 메모장 워크플로 실행 |
| 실제 GUI에서 일시정지·중단·Ctrl+Alt+F12 | CI는 자동 응답 | 긴 작업 중 버튼·단축키 누르기 |
| 승인 창이 다른 앱 위에 뜨는지 | CI는 자동 응답 | 테스트 앱 '선택 항목 삭제' 단계 실행 |
| 모델 비교(실제 테스트 앱) | 모델 없음 | 모델 화면 비교에서 '실제 테스트 앱' 켜고 실행 |
| 화면 배치·가독성 | 스크린샷은 CI 아티팩트로만 남고 개발 환경에서 내려받지 못함 | 각 화면을 열어 확인 |
