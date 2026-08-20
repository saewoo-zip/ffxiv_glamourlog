# 한섭(KR) 클라이언트 크래시 조사 기록

한국 서버(XIVLauncherKR)에서만 재현되는 네이티브 크래시(`0xC0000005`)를 조사하며 알게 된 것들.
비슷한 크래시를 또 만나면 여기부터 다시 보면 됨.

## 지금까지 확인된 크래시 지점

### 1. KamiToolKit `AttachNode` → `GetAddonByNode`
- `NodeBase.AttachNode(...)` → `PerformNativeAttach` → `UpdateParentAddon` →
  `RaptureAtkUnitManager.GetAddonByNode`에서 크래시.
- GuideWindow, Cabinet/Crystallize 버튼 생성 등 여러 곳에서 재현됨.
- 대응: KamiToolKit 네이티브 노드 부착 방식을 전부 걷어내고 순수 ImGui로 재작성
  (`imgui-migration-kr-crash-fix` 브랜치, 커밋 5개로 정리).

### 2. `clib`의 `AtkUnitBase.IsAddonReady(string)` → `GetAddonByName`
- crash dump: `crash-20260821001443.tspack`
- AutoDuty IPC(`EntrustAll` → `/glamourlog store`)가 자동 정리를 시작하기 전에 호출하는
  "Cabinet 준비됐나?" 체크에서 죽음. KamiToolKit과는 완전히 무관한 별도 경로.
- 스택: `CommandService+<>c.<Build>b__10_0()` → `clib.Extensions.AtkUnitBaseExtensions.IsAddonReady`
  → `RaptureAtkUnitManager.GetAddonByName` (네이티브, `ffxiv_dx11.exe+67C9A0`).
- 대응: `Svc.GameGui.GetAddonByName<T>()`(Dalamud 자체 구현, 별도 네이티브 호출 없음) +
  `AtkUnitBase.IsReady`(단순 비트필드 읽기)로 대체하는 `NativeAddon.IsReady()` 헬퍼 추가.
  (`GlamourLog/Services/NativeAddon.cs`, `imgui-migration-kr-crash-fix` 브랜치)

**공통점**: 둘 다 FFXIVClientStructs가 시그니처 스캔(바이트 패턴 매칭)으로 찾아내는 네이티브
함수를 직접 호출하는 코드였고, `RaptureAtkUnitManager` 관련 함수가 특히 의심됨.
반대로 `Svc.GameGui.GetAddonByName`처럼 Dalamud가 자체 구현한 API나, `AtkUnitBase.IsReady`처럼
단순 메모리 필드 읽기는 지금까지 문제 없었음.

## KR 클라이언트 지원 체계 (fork 관계)

```
goatcorp (본가, 글로벌 클라이언트)
  └─ ottercorp "Odder Otter Interactive" — CN(중국섭) 포크, 조직 단위로 관리, 활동 활발
       └─ dal4kr "Dalamud for KR" — KR(한섭) 포크, 개인/소규모 관리 (팔로워 3명)
```

- goatcorp: https://github.com/goatcorp — 본가 Dalamud/XIVLauncher
- ottercorp: https://github.com/ottercorp — `FFXIVQuickLauncher`(XIVLauncherCN), `Dalamud`, `FFXIVClientStructs`(기본 브랜치 `cn`)
- dal4kr: https://github.com/dal4kr — `Dalamud`, `FFXIVClientStructs`, `Dalamud.Updater`, `Dalamud.Resources`
  - 이슈: https://github.com/dal4kr/Dalamud.Updater/issues
  - 디스코드: https://discord.gg/UnRkQPMQFh

dal4kr 쪽 Discord(#개발) 대화 보면, 시그니처/오프셋이 한섭에서 안 맞을 때 종종 중섭(ottercorp)
버전을 베껴서 고치는 경우가 있다고 함("한섭은 중섭에 가까운거같기도"). 다만 이번 크래시들의
`GetAddonByName` 시그니처는 dal4kr/ottercorp/aers(글로벌 원본) 세 곳 다 **완전히 동일**했음:

```csharp
[MemberFunction("E8 ?? ?? ?? ?? 48 8B F8 41 B0 01"), GenerateStringOverloads]
public partial AtkUnitBase* GetAddonByName(CStringPointer name, int index = 1);
```

→ 즉 "시그니처 텍스트가 틀렸다"가 아니라, 이 바이트 패턴이 KR 바이너리 안에서 의도한 위치가
아닌 다른 곳에 매치되고 있다는 뜻. 중섭 걸로 갈아끼워도 도움 안 됨 — dal4kr이 KR 바이너리를
직접 디스어셈블해서 진짜 위치를 다시 찾아야 하는 문제.

## 직접 시그니처를 찾아본다면

- **도구**: IDA Pro가 커뮤니티(FFXIVClientStructs 포함) 사실상 표준이지만 유료(개인용도 몇백 달러).
  무료 대안은 [Ghidra](https://ghidra-sre.org) — 함수 개수 제한 없고 디버거 포함, 이 정도 작업엔 충분.
  IDA Free는 게임 리버싱 씬에서 거의 안 씀(디버거 없음 + 대형 바이너리 제약).
- **워크플로우**: 정적 디스어셈블(exe 자체를 열어 어셈블리로 읽기)로 후보 위치를 찾고,
  치트엔진/x64dbg로 라이브 검증. 파판14는 안티치트가 없어서 이 라이브 검증 루프가 상대적으로 쉬움.
- FFXIVClientStructs에는 부팅 시 어떤 시그니처가 해석 실패했는지 알려주는 "hook verifier" 진단
  도구가 있음(dal4kr Discord에서 언급) — 맨땅에서 찾는 게 아니라 "어디가 문제인지"는 이미 나옴.

## 참고: 크래시 덤프 분석 방법

`.tspack` 파일은 zip 압축 — `crash.log`(콜스택), `dalamud.log`, `trouble.json` 등 포함.
콜스택에서 마지막 관리 코드(managed) 프레임 다음에 오는 네이티브 프레임(`ffxiv_dx11.exe+...`)이
보통 시그니처 스캔이 잘못 짚은 지점.
