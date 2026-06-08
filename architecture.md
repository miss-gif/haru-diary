# React Native 일기 앱 — 기술 스택 & 구현 가이드

> Expo ~54 + expo-router ~6 기반. 프로덕션 레벨 일기 앱을 위한 기술 선택 근거와 구현 패턴 정리.

---

## 목차

1. [최종 기술 스택](#1-최종-기술-스택)
2. [기능별 레이어 구조](#2-기능별-레이어-구조)
3. [스토리지 전략 — MMKV vs AsyncStorage](#3-스토리지-전략--mmkv-vs-asyncstorage)
4. [생체인증 잠금 구현](#4-생체인증-잠금-구현)
5. [개발 단계 (마일스톤)](#5-개발-단계-마일스톤)
6. [주의사항 & 허위 패키지 경고](#6-주의사항--허위-패키지-경고)

---

## 1. 최종 기술 스택

| 영역           | 라이브러리                                             | 비고                                     |
| -------------- | ------------------------------------------------------ | ---------------------------------------- |
| Runtime        | Expo ~54 + expo-router ~6                              | 파일 기반 라우팅                         |
| DB             | drizzle-orm 0.45 + expo-sqlite 16                      | 타입세이프 SQLite, FTS5 전문검색         |
| 설정 스토리지  | react-native-mmkv                                      | AsyncStorage 대체 (50× 빠름)             |
| UI             | Tamagui + @gorhom/bottom-sheet 5 + @shopify/flash-list |                                          |
| 그래픽/모션    | reanimated 4 / moti / @shopify/react-native-skia       |                                          |
| 차트           | victory-native-xl                                      | ~~victory-native~~ → Skia 기반으로 교체  |
| 리스트         | @shopify/flash-list                                    | 대규모 일기 목록 성능 최적화             |
| 비동기 상태    | TanStack React Query 5                                 |                                          |
| 앱 상태        | Zustand                                                | ~~Context × 13개~~ → 1–2개 스토어로 통합 |
| 리치 텍스트    | @10play/tentap-editor                                  | Tiptap 기반, 현재 가장 활발히 유지보수   |
| 사진 선택·크롭 | react-native-image-crop-picker                         |                                          |
| 음성 입력      | @react-native-voice/voice                              | Expo config plugin 연동                  |
| 위치·날씨      | expo-location + OpenWeatherMap API                     |                                          |
| 알림           | expo-notifications                                     | 일기 작성 리마인더                       |
| 생체인증       | expo-local-authentication                              | FaceID / 지문                            |
| 보안 키 저장   | expo-secure-store                                      | PIN 해시 저장 (Keychain/Keystore)        |
| 암호화         | @noble/ciphers                                         | AES-256, PIN 파생 키                     |
| 파일 내보내기  | expo-file-system + expo-sharing                        | ~~react-native-fs~~ (bare RN 전용)       |
| 달력 뷰        | react-native-calendars                                 | Wix 관리, 감정 마킹 지원                 |
| i18n           | i18next (ko/en/ja)                                     | 한글 조사 처리 포함                      |
| 날짜           | dayjs                                                  |                                          |
| 검증           | zod                                                    |                                          |

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
| 감정 이모지 선택 | react-native-rn-emoji-keyboard       |
| 감정 달력        | react-native-calendars (커스텀 마킹) |
| 감정 통계 그래프 | victory-native-xl (스플라인 곡선)    |
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
npx expo install expo-local-authentication expo-secure-store
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

PIN을 평문으로 저장하지 않고 SHA-256 해시로만 저장합니다.
프로덕션에서는 `@noble/hashes`의 `pbkdf2`를 권장합니다.

```ts
import * as SecureStore from "expo-secure-store";
import { MMKV } from "react-native-mmkv";

export const lockStorage = new MMKV({ id: "lock-settings" });

const PIN_HASH_KEY = "diary_pin_hash";

async function hashPIN(pin: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(pin + "diary_salt_v1");
  const hashBuffer = await crypto.subtle.digest("SHA-256", data);
  return Array.from(new Uint8Array(hashBuffer))
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("");
}

export async function savePIN(pin: string) {
  const hash = await hashPIN(pin);
  await SecureStore.setItemAsync(PIN_HASH_KEY, hash);
}

export async function verifyPIN(pin: string): Promise<boolean> {
  const stored = await SecureStore.getItemAsync(PIN_HASH_KEY);
  if (!stored) return false;
  const hash = await hashPIN(pin);
  return hash === stored;
}

export async function hasPIN(): Promise<boolean> {
  const stored = await SecureStore.getItemAsync(PIN_HASH_KEY);
  return stored !== null;
}

export async function deletePIN() {
  await SecureStore.deleteItemAsync(PIN_HASH_KEY);
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
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  Vibration,
  Animated,
} from "react-native";

const MAX_ATTEMPTS = 5;

export function LockScreen() {
  const { unlock } = useLockStore();
  const { authenticate, biometricType, isEnrolled } = useBiometrics();
  const isBiometricEnabled = useLockStore((s) => s.isBiometricEnabled);

  const [pin, setPin] = useState("");
  const [attempts, setAttempts] = useState(0);
  const [error, setError] = useState("");
  const shakeAnim = useRef(new Animated.Value(0)).current;

  // 진입 시 즉시 생체인증 시도
  useEffect(() => {
    if (isBiometricEnabled && isEnrolled) tryBiometric();
  }, []);

  async function tryBiometric() {
    const success = await authenticate();
    if (success) unlock();
  }

  function shake() {
    Vibration.vibrate(400);
    Animated.sequence([
      Animated.timing(shakeAnim, {
        toValue: 10,
        duration: 60,
        useNativeDriver: true,
      }),
      Animated.timing(shakeAnim, {
        toValue: -10,
        duration: 60,
        useNativeDriver: true,
      }),
      Animated.timing(shakeAnim, {
        toValue: 6,
        duration: 60,
        useNativeDriver: true,
      }),
      Animated.timing(shakeAnim, {
        toValue: 0,
        duration: 60,
        useNativeDriver: true,
      }),
    ]).start();
  }

  async function handlePINChange(input: string) {
    setPin(input);
    if (input.length === 4) {
      const correct = await verifyPIN(input);
      if (correct) {
        unlock();
        return;
      }

      const next = attempts + 1;
      setAttempts(next);
      setPin("");
      shake();
      setError(
        next >= MAX_ATTEMPTS
          ? "시도 초과. 1분 후 다시 시도하세요."
          : `PIN이 올바르지 않습니다. (${next}/${MAX_ATTEMPTS})`,
      );
    }
  }

  return (
    <View style={styles.container}>
      <Text style={styles.title}>일기 잠금</Text>

      <Animated.View style={{ transform: [{ translateX: shakeAnim }] }}>
        <View style={styles.dotsRow}>
          {[0, 1, 2, 3].map((i) => (
            <View
              key={i}
              style={[styles.dot, i < pin.length && styles.dotFilled]}
            />
          ))}
        </View>
      </Animated.View>

      {error ? <Text style={styles.error}>{error}</Text> : null}

      <TextInput
        value={pin}
        onChangeText={handlePINChange}
        keyboardType="number-pad"
        maxLength={4}
        secureTextEntry
        autoFocus
        style={styles.hiddenInput}
      />

      {isBiometricEnabled && isEnrolled && (
        <TouchableOpacity onPress={tryBiometric} style={styles.bioButton}>
          <Text style={styles.bioLabel}>
            {biometricType === "facial" ? "Face ID로 열기" : "지문으로 열기"}
          </Text>
        </TouchableOpacity>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: "center",
    justifyContent: "center",
    gap: 24,
  },
  title: { fontSize: 20, fontWeight: "500" },
  dotsRow: { flexDirection: "row", gap: 16 },
  dot: {
    width: 16,
    height: 16,
    borderRadius: 8,
    borderWidth: 1.5,
    borderColor: "#888",
  },
  dotFilled: { backgroundColor: "#534AB7", borderColor: "#534AB7" },
  error: { color: "#E24B4A", fontSize: 13 },
  hiddenInput: { position: "absolute", opacity: 0, width: 0, height: 0 },
  bioButton: { marginTop: 8, padding: 12 },
  bioLabel: { color: "#534AB7", fontSize: 15 },
});
```

### 앱 루트에 잠금 게이트 연결 (`/app/_layout.tsx`)

```tsx
import { useEffect } from "react";
import { AppState } from "react-native";
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

```tsx
// Tamagui 테마 토큰 정의 예시
const diaryTokens = createTokens({
  color: {
    background: "#FAFAF8",
    surface: "#F3F0EB",
    accent: "#7C6EE8", // 포인트 컬러 — 앱 전체에서 일관 적용
    text: "#1A1A1A",
    textMuted: "#6B6B6B",
  },
  space: { sm: 8, md: 16, lg: 24 },
  radius: { card: 16, button: 10 },
});

// 사용 시
<YStack bg="$background" p="$md" borderRadius="$card">
  <Text color="$text" fontSize={16}>
    오늘의 일기
  </Text>
</YStack>;
```

---

## 5. 개발 단계 (마일스톤)

### Phase 1 — MVP (4–6주)

스토어 출시 가능한 최소 완성 상태.

- [ ] 일기 CRUD (drizzle-orm + expo-sqlite)
- [ ] 감정 태그 선택
- [ ] 캘린더 뷰 (react-native-calendars)
- [ ] PIN 잠금 + 생체인증
- [ ] 기본 타임라인 피드 (flash-list)
- [ ] 다크모드 + i18n (ko/en)

### Phase 2 — 핵심 차별화 (3–4주)

앱 완성도를 올리는 기능들.

- [ ] 리치 텍스트 에디터 (@10play/tentap-editor)
- [ ] 사진 첨부·크롭
- [ ] 위치·날씨 자동 태깅
- [ ] FTS5 전문 검색
- [ ] 감정 통계 그래프 (victory-native-xl)

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

| 항목          | 변경 전        | 변경 후             | 이유                            |
| ------------- | -------------- | ------------------- | ------------------------------- |
| 설정 스토리지 | async-storage  | react-native-mmkv   | 50× 성능, 암호화 내장, 동기 API |
| 차트          | victory-native | victory-native-xl   | Skia 기반, 유지보수 활성화      |
| 상태 관리     | Context × 13   | Zustand             | 리렌더링 최소화, 코드 단순화    |
| 리스트        | FlatList       | @shopify/flash-list | 대규모 목록 성능                |

---

전체적으로 잘 정리된 스택입니다. 몇 가지 보완할 포인트를 짚어드릴게요.

## 잘 된 부분

- MMKV vs SQLite 역할 분리가 명확함
- 생체인증 레이어 구조(LocalAuth → SecureStore → MMKV → Zustand)가 논리적
- 허위 패키지 경고 섹션이 실용적

---

## 보완이 필요한 부분

### 1. 보안 — 가장 중요

**PIN 해싱이 취약합니다.**

현재 코드가 `SHA-256 + 고정 salt("diary_salt_v1")`를 사용하는데, 문서 안에도 pbkdf2 권장이라고 적혀있지만 실제 코드엔 반영이 안 됐습니다. 프로덕션 전에 반드시 교체해야 합니다.

```ts
// 현재 (취약)
crypto.subtle.digest("SHA-256", encode(pin + "diary_salt_v1"));

// 권장
import { pbkdf2 } from "@noble/hashes/pbkdf2";
import { sha256 } from "@noble/hashes/sha256";
// 사용자별 랜덤 salt를 SecureStore에 함께 저장
pbkdf2(sha256, pin, randomSalt, { c: 100_000, dkLen: 32 });
```

**시도 초과 처리가 메모리에만 있습니다.**

`attempts` state가 컴포넌트 언마운트 시 초기화되므로, 앱을 강제 종료하면 잠금이 풀립니다. MMKV나 SecureStore에 시도 횟수 + 잠금 만료 시각을 저장해야 합니다.

---

### 2. 아키텍처 — 누락된 레이어

**에러 바운더리가 없습니다.**

DB 마이그레이션 실패, SecureStore 접근 오류 등 앱 시작 시 크리티컬한 오류를 잡을 장치가 필요합니다.

```ts
// app/_layout.tsx에 추가 필요
<ErrorBoundary fallback={<CriticalErrorScreen />}>
  {isLocked ? <LockScreen /> : <Slot />}
</ErrorBoundary>
```

**drizzle 마이그레이션 전략이 없습니다.**

Phase 1 MVP 이후 스키마가 바뀔 때 기존 사용자 데이터를 어떻게 처리할지 초기에 설계해야 합니다. `drizzle-kit`의 migration 파일 관리 방식을 스택에 명시하는 게 좋습니다.

---

### 3. 성능 — 간과하기 쉬운 부분

**사진 저장 경로 전략이 없습니다.**

`react-native-image-crop-picker`로 선택한 이미지를 어디에 저장할지 명시가 없습니다. 캐시 디렉토리에 두면 OS가 삭제할 수 있으므로, `expo-file-system`의 `documentDirectory`로 복사하는 로직이 필요합니다.

**FTS5 인덱스 업데이트 시점이 불명확합니다.**

일기 저장 시 FTS 가상 테이블도 같은 트랜잭션 안에서 업데이트해야 합니다. 별도 처리로 분리되면 검색 결과가 불일치할 수 있습니다.

---

### 4. 누락된 스택 항목

| 영역            | 현재      | 추가 권장                            |
| --------------- | --------- | ------------------------------------ |
| 에러 추적       | 없음      | Sentry (`@sentry/react-native`)      |
| DB 마이그레이션 | 언급 없음 | `drizzle-kit` 명시                   |
| 테스트          | 없음      | Jest + Testing Library (Phase 1부터) |
| OTA 업데이트    | 없음      | `expo-updates`                       |

---

### 5. 사소하지만 실제로 막히는 것

`checkShouldLock`이 `autoLockMinutes === 0`(즉시 잠금)을 처리하지 않습니다. 백그라운드로 갔다 바로 돌아와도 잠기길 원하는 사용자 설정을 지원하려면 0 케이스를 명시적으로 처리해야 합니다.

---

**결론적으로** 기술 선택 자체는 훌륭하고, Phase 1 MVP 진행에 큰 무리는 없습니다. 다만 PIN 해싱과 시도 횟수 영속화는 보안 관련이라 MVP 전에 반드시 수정하고, 나머지는 Phase 2 전에 챙기면 됩니다.
