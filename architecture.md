# React Native 일기 앱 — 기술 스택 & 구현 가이드

> Expo ~56 + expo-router ~56 기반. 프로덕션 레벨 일기 앱을 위한 기술 선택 근거와 구현 패턴 정리.

---

## 목차

1. [최종 기술 스택](#1-최종-기술-스택)
2. [기능별 레이어 구조](#2-기능별-레이어-구조)
3. [스토리지 전략 — MMKV vs AsyncStorage](#3-스토리지-전략--mmkv-vs-asyncstorage)
4. [생체인증 잠금 구현](#4-생체인증-잠금-구현)
5. [개발 단계 (마일스톤)](#5-개발-단계-마일스톤)
6. [주의사항 & 허위 패키지 경고](#6-주의사항--허위-패키지-경고)
7. [추가 권장 스택 & 구현 주의](#7-추가-권장-스택--구현-주의)

---

## 1. 최종 기술 스택

| 영역           | 라이브러리                                             | 비고                                           |
| -------------- | ------------------------------------------------------ | ---------------------------------------------- |
| Runtime        | Expo ~56 + expo-router ~56                              | 파일 기반 라우팅                               |
| DB             | drizzle-orm 0.45 + expo-sqlite ~56                      | 타입세이프 SQLite, FTS5 전문검색               |
| 설정 스토리지  | react-native-mmkv                                      | AsyncStorage 대체 (50× 빠름)                   |
| UI             | Tamagui + @gorhom/bottom-sheet 5 + @shopify/flash-list |                                                |
| 그래픽/모션    | reanimated 4 / moti / @shopify/react-native-skia       |                                                |
| 차트           | victory-native (v40+)                                  | Skia 기반 재작성판 (repo명: victory-native-xl) |
| 리스트         | @shopify/flash-list                                    | 대규모 일기 목록 성능 최적화                   |
| 비동기 상태    | TanStack React Query 5                                 |                                                |
| 앱 상태        | Zustand                                                | ~~Context × 13개~~ → 1–2개 스토어로 통합       |
| 리치 텍스트    | @10play/tentap-editor                                  | Tiptap 기반, 현재 가장 활발히 유지보수         |
| 사진 선택·크롭 | react-native-image-crop-picker                         |                                                |
| 음성 입력      | @react-native-voice/voice                              | Expo config plugin 연동                        |
| 위치·날씨      | expo-location + OpenWeatherMap API                     |                                                |
| 알림           | expo-notifications                                     | 일기 작성 리마인더                             |
| 생체인증       | expo-local-authentication                              | FaceID / 지문                                  |
| 보안 키 저장   | expo-secure-store                                      | PIN 해시 저장 (Keychain/Keystore)              |
| 암호화         | @noble/ciphers                                         | AES-256, PIN 파생 키                           |
| 파일 내보내기  | expo-file-system + expo-sharing                        | ~~react-native-fs~~ (bare RN 전용)             |
| 달력 뷰        | react-native-calendars                                 | Wix 관리, 감정 마킹 지원                       |
| i18n           | i18next (ko/en/ja)                                     | 한글 조사 처리 포함                            |
| 날짜           | dayjs                                                  |                                                |
| 검증           | zod                                                    |                                                |

---

## 2. 기능별 레이어 구조

### 쓰기 (Write)

> Day One · Daylio · Reflectly 핵심 기능

| 기능                | 라이브러리                                  |
| ------------------- | ------------------------------------------- |
| 리치 텍스트 에디터  | @10play/tentap-editor (Tiptap 기반 WebView) |
| 음성 받아쓰기 (STT) | @react-native-voice/voice                   |
| 사진 첨부·크롭      | react-native-image-crop-picker              |
| 위치·날씨 자동 태깅 | expo-location + OpenWeatherMap API          |

### 감정 트래킹 (Mood)

> Daylio · Reflectly · Stoic 핵심 기능

| 기능             | 라이브러리                           |
| ---------------- | ------------------------------------ |
| 감정 이모지 선택 | rn-emoji-keyboard                    |
| 감정 달력        | react-native-calendars (커스텀 마킹) |
| 감정 통계 그래프 | victory-native (스플라인 곡선)       |
| AI 감정 인사이트 | Claude API (감정 패턴 분석)          |

### 탐색 (Browse)

> Day One · Journey 핵심 기능

| 기능          | 구현 방법                                             |
| ------------- | ----------------------------------------------------- |
| 타임라인 피드 | @shopify/flash-list                                   |
| 전문 검색     | expo-sqlite FTS5 가상 테이블 (별도 라이브러리 불필요) |
| 태그·필터     | drizzle-orm 쿼리 빌더                                 |

### 보안·백업 (Security)

> 모든 프리미엄 일기 앱 공통 기능

| 기능          | 구현 방법                                |
| ------------- | ---------------------------------------- |
| 생체인증 잠금 | expo-local-authentication                |
| PIN 잠금      | expo-secure-store + @noble/ciphers       |
| 클라우드 백업 | expo-file-system + Google Drive / iCloud |
| E2E 암호화    | @noble/ciphers AES-256                   |

### 알림·습관 (Habit)

> Reflectly · Stoic 차별화 기능

- 일기 작성 리마인더 → `expo-notifications`
- 연속 작성 스트릭 → drizzle-orm + Zustand
- 홈 화면 위젯 → `expo-widgets` (iOS) / `react-native-android-widget`

---

## 3. 스토리지 전략 — MMKV vs AsyncStorage

### 성능 비교

| 항목          | react-native-mmkv                  | @react-native-async-storage |
| ------------- | ---------------------------------- | --------------------------- |
| 단일 값 읽기  | ~0.1ms                             | ~5ms                        |
| 100개 키 읽기 | ~2ms                               | ~220ms                      |
| 속도 차이     | **50× 빠름**                       | —                           |
| API 방식      | 동기 (await 불필요)                | 비동기                      |
| 암호화        | AES-128 내장 (옵션)                | 없음                        |
| 타입          | getBoolean / getNumber / getString | 모두 string                 |
| Expo Go       | 미지원 (prebuild 필요)             | 지원                        |

### 일기 앱 스토리지 분담

**MMKV로 저장할 것** (설정값, 빈번히 읽는 작은 데이터)

```
- 다크모드 설정
- 폰트 크기 설정
- 알림 시간
- 마지막 선택 감정
- PIN 해시 (암호화 인스턴스)
- 온보딩 완료 여부
- 앱 잠금 상태 / 마지막 인증 시각
- 자동 잠금 유예 시간
- 생체인증 활성화 여부
```

**drizzle-orm + expo-sqlite로 저장할 것** (일기 본문, 관계형 데이터)

```
- 일기 본문 · 제목
- 감정 태그 데이터
- 사진 경로 목록
- 위치 · 날씨 데이터
- 태그 / 카테고리
- 검색 인덱스 (FTS5)
```

### MMKV 암호화 인스턴스 (PIN 연동)

```ts
import { MMKV } from "react-native-mmkv";

// PIN 검증 후 암호화 인스턴스 생성
const storage = new MMKV({
  id: "user-settings",
  encryptionKey: derivedKeyFromPIN, // @noble로 파생한 키
});
```

---

## 4. 생체인증 잠금 구현

### 아키텍처 레이어

```
expo-local-authentication  →  기기 생체인증 호출
expo-secure-store          →  PIN 해시 안전 저장 (Keychain/Keystore)
react-native-mmkv          →  잠금 설정 상태 (활성화 여부, 마지막 인증 시각)
Zustand (useLockStore)     →  전역 잠금 상태 관리
```

### 설치

```bash
npx expo install expo-local-authentication expo-secure-store expo-crypto react-native-mmkv
npm i @noble/hashes
```

`app.json`:

```json
{
  "expo": {
    "plugins": [
      [
        "expo-local-authentication",
        {
          "faceIDPermission": "일기를 잠금 해제하기 위해 Face ID를 사용합니다."
        }
      ]
    ]
  }
}
```

### 보안 유틸리티 (`/lib/auth/secureStorage.ts`)

PIN을 평문으로 저장하지 않고 pbkdf2(SHA-256, 10만 라운드)로 파생한 키만 저장합니다.
salt는 사용자별로 랜덤 생성해 SecureStore에 함께 보관합니다.

> `crypto.subtle`(WebCrypto)은 Expo 런타임에 없으므로 사용 금지. 난수는 `expo-crypto`,
> 해시·KDF는 순수 JS인 `@noble/hashes`(v2, ESM·서브경로 import)를 씁니다.

```ts
import * as SecureStore from "expo-secure-store";
import * as Crypto from "expo-crypto";
import { pbkdf2Async } from "@noble/hashes/pbkdf2.js";
import { sha256 } from "@noble/hashes/sha2.js";
import { MMKV } from "react-native-mmkv";

export const lockStorage = new MMKV({ id: "lock-settings" });

const PIN_HASH_KEY = "diary_pin_hash";
const PIN_SALT_KEY = "diary_pin_salt";

function toHex(bytes: Uint8Array): string {
  return Array.from(bytes)
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("");
}

function fromHex(hex: string): Uint8Array {
  return Uint8Array.from(hex.match(/.{2}/g)!.map((h) => parseInt(h, 16)));
}

// PIN + 사용자별 랜덤 salt로 pbkdf2 파생 (100k 라운드)
async function derivePIN(pin: string, salt: Uint8Array): Promise<string> {
  const key = await pbkdf2Async(sha256, pin, salt, { c: 100_000, dkLen: 32 });
  return toHex(key);
}

export async function savePIN(pin: string) {
  const salt = Crypto.getRandomBytes(16);
  const hash = await derivePIN(pin, salt);
  await SecureStore.setItemAsync(PIN_SALT_KEY, toHex(salt));
  await SecureStore.setItemAsync(PIN_HASH_KEY, hash);
}

export async function verifyPIN(pin: string): Promise<boolean> {
  const storedHash = await SecureStore.getItemAsync(PIN_HASH_KEY);
  const storedSalt = await SecureStore.getItemAsync(PIN_SALT_KEY);
  if (!storedHash || !storedSalt) return false;
  const hash = await derivePIN(pin, fromHex(storedSalt));
  return hash === storedHash;
}

export async function hasPIN(): Promise<boolean> {
  return (await SecureStore.getItemAsync(PIN_HASH_KEY)) !== null;
}

export async function deletePIN() {
  await SecureStore.deleteItemAsync(PIN_HASH_KEY);
  await SecureStore.deleteItemAsync(PIN_SALT_KEY);
}
```

### 생체인증 훅 (`/lib/auth/useBiometrics.ts`)

```ts
import * as LocalAuthentication from "expo-local-authentication";
import { useCallback, useEffect, useState } from "react";

export type BiometricType = "fingerprint" | "facial" | "iris" | "none";

export function useBiometrics() {
  const [state, setState] = useState({
    isAvailable: false,
    biometricType: "none" as BiometricType,
    isEnrolled: false,
  });

  useEffect(() => {
    checkAvailability();
  }, []);

  async function checkAvailability() {
    const compatible = await LocalAuthentication.hasHardwareAsync();
    const enrolled = await LocalAuthentication.isEnrolledAsync();
    const types = await LocalAuthentication.supportedAuthenticationTypesAsync();

    let biometricType: BiometricType = "none";
    if (
      types.includes(LocalAuthentication.AuthenticationType.FACIAL_RECOGNITION)
    ) {
      biometricType = "facial";
    } else if (
      types.includes(LocalAuthentication.AuthenticationType.FINGERPRINT)
    ) {
      biometricType = "fingerprint";
    } else if (types.includes(LocalAuthentication.AuthenticationType.IRIS)) {
      biometricType = "iris";
    }

    setState({ isAvailable: compatible, biometricType, isEnrolled: enrolled });
  }

  const authenticate = useCallback(async (): Promise<boolean> => {
    if (!state.isAvailable || !state.isEnrolled) return false;

    const result = await LocalAuthentication.authenticateAsync({
      promptMessage: "일기를 열려면 인증하세요",
      fallbackLabel: "PIN 입력", // iOS 전용
      cancelLabel: "취소",
      disableDeviceFallback: false, // false = 기기 PIN/패턴 폴백 허용
    });

    return result.success;
  }, [state]);

  return { ...state, authenticate };
}
```

### 잠금 상태 전역 관리 (`/lib/auth/useLockStore.ts`)

```ts
import { create } from "zustand";
import { lockStorage } from "./secureStorage";

const LAST_AUTH_KEY = "last_authenticated_at";
const AUTO_LOCK_MINUTES_KEY = "auto_lock_minutes";
const BIOMETRIC_ENABLED_KEY = "biometric_enabled";

interface LockStore {
  isLocked: boolean;
  isBiometricEnabled: boolean;
  autoLockMinutes: number;
  lock: () => void;
  unlock: () => void;
  setAutoLockMinutes: (minutes: number) => void;
  setBiometricEnabled: (enabled: boolean) => void;
  checkShouldLock: () => void;
}

export const useLockStore = create<LockStore>((set, get) => ({
  isLocked: true,
  isBiometricEnabled: lockStorage.getBoolean(BIOMETRIC_ENABLED_KEY) ?? false,
  autoLockMinutes: lockStorage.getNumber(AUTO_LOCK_MINUTES_KEY) ?? 5,

  lock: () => set({ isLocked: true }),

  unlock: () => {
    lockStorage.set(LAST_AUTH_KEY, Date.now());
    set({ isLocked: false });
  },

  setAutoLockMinutes: (minutes) => {
    lockStorage.set(AUTO_LOCK_MINUTES_KEY, minutes);
    set({ autoLockMinutes: minutes });
  },

  setBiometricEnabled: (enabled) => {
    lockStorage.set(BIOMETRIC_ENABLED_KEY, enabled);
    set({ isBiometricEnabled: enabled });
  },

  // 백그라운드 복귀 시 유예 시간 초과 여부 확인
  checkShouldLock: () => {
    // 0 = 즉시 잠금 (백그라운드 복귀 시 무조건 잠금)
    if (get().autoLockMinutes === 0) {
      set({ isLocked: true });
      return;
    }
    const lastAuth = lockStorage.getNumber(LAST_AUTH_KEY) ?? 0;
    const autoLockMs = get().autoLockMinutes * 60 * 1000;
    const elapsed = Date.now() - lastAuth;
    if (elapsed > autoLockMs) set({ isLocked: true });
  },
}));
```

### 잠금 화면 (`/components/auth/LockScreen.tsx`)

```tsx
import { useBiometrics } from "@/lib/auth/useBiometrics";
import { useLockStore } from "@/lib/auth/useLockStore";
import { verifyPIN } from "@/lib/auth/secureStorage";
import { useEffect, useRef, useState } from "react";
import { View, Text, TextInput, TouchableOpacity } from "react-native";

const MAX_ATTEMPTS = 5;

export function LockScreen() {
  const { unlock } = useLockStore();
  const { authenticate, biometricType, isEnrolled } = useBiometrics();
  const isBiometricEnabled = useLockStore((s) => s.isBiometricEnabled);

  const [pin, setPin] = useState("");
  const [attempts, setAttempts] = useState(0);
  const [error, setError] = useState("");
  const triedBiometric = useRef(false);

  // 진입 시 즉시 생체인증 시도 (isEnrolled는 비동기로 늦게 채워지므로 deps에 포함)
  useEffect(() => {
    if (!triedBiometric.current && isBiometricEnabled && isEnrolled) {
      triedBiometric.current = true;
      tryBiometric();
    }
  }, [isBiometricEnabled, isEnrolled]);

  async function tryBiometric() {
    const success = await authenticate();
    if (success) unlock();
  }

  async function handlePINChange(input: string) {
    setPin(input);
    if (input.length !== 4) return;

    if (await verifyPIN(input)) {
      unlock();
      return;
    }

    const next = attempts + 1;
    setAttempts(next);
    setPin("");
    // 오답 피드백(흔들기/진동)은 여기서 트리거
    setError(
      next >= MAX_ATTEMPTS
        ? "시도 초과. 1분 후 다시 시도하세요."
        : `PIN이 올바르지 않습니다. (${next}/${MAX_ATTEMPTS})`,
    );
  }

  // 스타일·PIN 도트 표시는 생략 — 잠금 로직만 표기
  return (
    <View>
      <Text>일기 잠금</Text>

      {error ? <Text>{error}</Text> : null}

      <TextInput
        value={pin}
        onChangeText={handlePINChange}
        keyboardType="number-pad"
        maxLength={4}
        secureTextEntry
        autoFocus
      />

      {isBiometricEnabled && isEnrolled && (
        <TouchableOpacity onPress={tryBiometric}>
          <Text>
            {biometricType === "facial" ? "Face ID로 열기" : "지문으로 열기"}
          </Text>
        </TouchableOpacity>
      )}
    </View>
  );
}
```

### 앱 루트에 잠금 게이트 연결 (`/app/_layout.tsx`)

```tsx
import { useEffect } from "react";
import { AppState } from "react-native";
import { Slot } from "expo-router";
import { useLockStore } from "@/lib/auth/useLockStore";
import { LockScreen } from "@/components/auth/LockScreen";

export default function RootLayout() {
  const { isLocked, checkShouldLock } = useLockStore();

  // 백그라운드 → 포그라운드 복귀 시 자동 잠금 체크
  useEffect(() => {
    const sub = AppState.addEventListener("change", (state) => {
      if (state === "active") checkShouldLock();
    });
    return () => sub.remove();
  }, []);

  if (isLocked) return <LockScreen />;

  return <Slot />; // expo-router
}
```

---

## 5. 개발 단계 (마일스톤)

### Phase 1 — MVP (4–6주)

스토어 출시 가능한 최소 완성 상태.

- [ ] 일기 CRUD (drizzle-orm + expo-sqlite)
- [ ] 감정 태그 선택
- [ ] 캘린더 뷰 (react-native-calendars)
- [ ] PIN 잠금 + 생체인증
- [ ] 시도 횟수 영속화 + 잠금 만료 (보안 필수 — §7 참조)
- [ ] 에러 바운더리 (DB·SecureStore 오류 차단 — §7 참조)
- [ ] 기본 타임라인 피드 (flash-list)
- [ ] 다크모드 + i18n (ko/en)

### Phase 2 — 핵심 차별화 (3–4주)

앱 완성도를 올리는 기능들.

- [ ] 리치 텍스트 에디터 (@10play/tentap-editor)
- [ ] 사진 첨부·크롭
- [ ] 사진 documentDirectory 영구 저장 (캐시 삭제 방지 — §7 참조)
- [ ] 위치·날씨 자동 태깅
- [ ] FTS5 전문 검색
- [ ] FTS5 인덱스 동기화 (저장과 동일 트랜잭션 — §7 참조)
- [ ] 감정 통계 그래프 (victory-native)

### Phase 3 — 리텐션 기능 (2–3주)

사용자가 계속 돌아오게 만드는 기능들.

- [ ] 연속 작성 스트릭
- [ ] 일기 작성 리마인더 알림
- [ ] JSON 내보내기 / 가져오기
- [ ] 클라우드 백업 (Google Drive / iCloud)

### Phase 4 — 프리미엄 기능 (2–3주)

유료 전환율을 높이는 기능들.

- [ ] 음성 받아쓰기 STT
- [ ] AI 감정 인사이트 (Claude API)
- [ ] 홈 화면 위젯
- [ ] E2E 암호화

---

## 6. 주의사항 & 허위 패키지 경고

### 존재하지 않는 패키지

> `@chaitrabhairappa/react-native-rich-text-editor`는 npm에 없는 라이브러리입니다.
> AI가 생성한 허위 패키지명이며, 설치 시 오류가 발생합니다.

리치 텍스트 에디터 현실적인 선택지:

| 라이브러리            | 방식             | 상태                               |
| --------------------- | ---------------- | ---------------------------------- |
| @10play/tentap-editor | WebView (Tiptap) | 활발히 유지보수 중 — **권장**      |
| react-native-cn-quill | WebView (Quill)  | 유지보수 느림                      |
| 순수 네이티브 에디터  | Native           | 프로덕션 수준 라이브러리 현재 없음 |

### Expo 환경 주의사항

- `react-native-fs` → Expo에서는 `expo-file-system`으로 대체
- `react-native-mmkv` → Expo Go 미지원, `npx expo prebuild` 필요
- `react-native-image-crop-picker` → Expo config plugin 필요
- `@react-native-voice/voice` → Expo config plugin + 마이크 권한 설정 필요

### 기존 스택에서 변경된 항목 요약

| 항목          | 변경 전                  | 변경 후                    | 이유                                    |
| ------------- | ------------------------ | -------------------------- | --------------------------------------- |
| 설정 스토리지 | async-storage            | react-native-mmkv          | 50× 성능, 암호화 내장, 동기 API         |
| 차트          | victory-native v36 (SVG) | victory-native v40+ (Skia) | Skia 기반 재작성, 같은 패키지 메이저 업 |
| 상태 관리     | Context × 13             | Zustand                    | 리렌더링 최소화, 코드 단순화            |
| 리스트        | FlatList                 | @shopify/flash-list        | 대규모 목록 성능                        |

---

## 7. 추가 권장 스택 & 구현 주의

### 추가 권장 스택

| 영역            | 추가 권장              | 이유                              |
| --------------- | ---------------------- | --------------------------------- |
| 에러 추적       | `@sentry/react-native` | 크래시·예외 수집                  |
| DB 마이그레이션 | `drizzle-kit`          | 스키마 변경 시 사용자 데이터 보존 |
| 테스트          | Jest + Testing Library | Phase 1부터                       |
| OTA 업데이트    | `expo-updates`         | 스토어 심사 없이 핫픽스           |

### 구현 시 필수 처리

- **시도 횟수 영속화** — `LockScreen`의 `attempts`가 컴포넌트 state에만 있어 앱 강제 종료 시 잠금 우회됨. MMKV/SecureStore에 실패 횟수 + 잠금 만료 시각을 저장해야 함 (현재 `"1분 후 다시 시도"` 메시지는 미구현 상태).
- **에러 바운더리** — DB 마이그레이션·SecureStore 접근 실패를 잡도록 `_layout.tsx` 루트를 `<ErrorBoundary fallback={<CriticalErrorScreen />}>`로 감쌀 것.
- **FTS5 동기화** — 일기 저장과 FTS 가상 테이블 업데이트를 같은 트랜잭션에서 처리. 분리되면 검색 결과 불일치.
- **사진 저장 경로** — `react-native-image-crop-picker` 결과를 `expo-file-system`의 `documentDirectory`로 복사. 캐시 디렉토리는 OS가 삭제함.

---
