# 한섭(KR) 클라이언트 크래시 조사 기록

대상: XIVLauncherKR 전용 재현 크래시 (`0xC0000005`)

## 확인된 크래시 지점

### 1. KamiToolKit AttachNode → GetAddonByNode
- 호출 경로: `NodeBase.AttachNode(...)` → `PerformNativeAttach` → `UpdateParentAddon` → `RaptureAtkUnitManager.GetAddonByNode`
- 재현 지점: GuideWindow, Cabinet/Crystallize 버튼 생성
- 조치: KamiToolKit 네이티브 노드 부착 제거, ImGui로 대체
- 커밋: `main` 브랜치, 5개 커밋

### 2. clib AtkUnitBase.IsAddonReady(string) → GetAddonByName
- 크래시 덤프: `crash-20260821001443.tspack`
- 호출 경로: `CommandService+<>c.<Build>b__10_0()` → `clib.Extensions.AtkUnitBaseExtensions.IsAddonReady` → `RaptureAtkUnitManager.GetAddonByName` (`ffxiv_dx11.exe+67C9A0`)
- 트리거: AutoDuty IPC `EntrustAll` → `/glamourlog store` 커맨드의 Cabinet 준비 상태 체크
- KamiToolKit 의존: 없음
- 조치: `NativeAddon.IsReady()` 헬퍼로 대체 — `Svc.GameGui.GetAddonByName<T>()` + `AtkUnitBase.IsVisible`/`IsReady`(비트필드 읽기) + `AtkUnitBase.IsFullyLoaded()`([VirtualFunction], vtable 호출)
- 판정 기준: clib 원본과 동일(`IsVisible && IsReady && IsFullyLoaded()`) — clib.dll 디컴파일로 확인
- 파일: `GlamourLog/Services/NativeAddon.cs`, `main` 브랜치

### 공통 패턴
- 크래시 함수: FFXIVClientStructs `[MemberFunction]`(시그니처 스캔) 기반 네이티브 호출, `RaptureAtkUnitManager` 계열
- 안전 확인된 경로: `Svc.GameGui.GetAddonByName`(Dalamud 자체 구현), 단순 필드 읽기(`IsReady`/`IsVisible` 등), `[VirtualFunction]`(vtable 호출, 예: `IsFullyLoaded`) — 시그니처 스캔이 아니라서 위험도 별개

## KR 클라이언트 지원 체계

포크 구조:
```
goatcorp (글로벌 원본)
  └─ ottercorp — CN 포크, 조직 관리
       └─ dal4kr — KR 포크, 개인/소규모 관리
```

- goatcorp: https://github.com/goatcorp
- ottercorp: https://github.com/ottercorp — 저장소: FFXIVQuickLauncher(XIVLauncherCN), Dalamud, FFXIVClientStructs(기본 브랜치 `cn`)
- dal4kr: https://github.com/dal4kr — 저장소: Dalamud, FFXIVClientStructs, Dalamud.Updater, Dalamud.Resources
  - 이슈: https://github.com/dal4kr/Dalamud.Updater/issues
  - 디스코드: https://discord.gg/UnRkQPMQFh

GetAddonByName 시그니처 비교 (dal4kr/ottercorp/aers 3사 동일):
```csharp
[MemberFunction("E8 ?? ?? ?? ?? 48 8B F8 41 B0 01"), GenerateStringOverloads]
public partial AtkUnitBase* GetAddonByName(CStringPointer name, int index = 1);
```

원인 분류: 시그니처 텍스트 오류 아님. 바이트 패턴의 KR 바이너리 내 매치 위치 오류.
수정 주체: dal4kr (KR 바이너리 직접 디스어셈블 필요).

## 시그니처 직접 조사 시 참고

- 도구: Ghidra(무료, 디컴파일러·디버거 포함, 함수 개수 제한 없음). IDA Pro: 유료(개인용 라이선스 수백 달러), 커뮤니티 표준.
- 절차: 정적 디스어셈블로 후보 위치 특정 → 치트엔진/x64dbg로 라이브 검증
- 진단 도구: FFXIVClientStructs "hook verifier" — 부팅 시 시그니처 해석 실패 목록 출력

## 크래시 덤프 분석

- 파일 형식: `.tspack` = zip
- 포함 파일: `crash.log`(콜스택), `dalamud.log`, `trouble.json`
- 크래시 지점 특정 기준: 콜스택 마지막 managed 프레임 다음의 네이티브 프레임(`ffxiv_dx11.exe+...`)
