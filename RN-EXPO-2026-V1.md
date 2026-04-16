Below is the **Technical Specification for React Native & Expo Development (2026 Edition)** in Markdown format.

# Technical Specification: React Native & Expo
**Standardized Best Practices & Architectural Guidelines — 2026 Edition**

---

## 1. Executive Summary
This specification defines the architectural and development standards for modern React Native applications within the Expo ecosystem. For 2026, the primary focus is on the **New Architecture**, modular feature-based design, and type-safe infrastructure. Adherence to these standards ensures maximum performance, cross-platform consistency, and long-term maintainability.

> **Objective:** To establish a "Framework-First" methodology leveraging Expo’s managed infrastructure while maintaining the performance levels of pure native implementations.

---

## 2. Core Architecture & Engine

### 2.1 The New Architecture (Fabric & TurboModules)
* **Requirement:** All new projects MUST enable the New Architecture by default. The legacy asynchronous bridge is considered legacy/deprecated.
* **Mechanism:** Use the **JSI (JavaScript Interface)** for synchronous communication between JavaScript and Native layers.
* **Constraint:** Avoid libraries that have not migrated to TurboModules to prevent "bridge-hops" that degrade Time to Interactive (TTI).

### 2.2 JavaScript Engine (Hermes)
* **Standard:** Hermes is the mandatory engine.
* **Optimization:** Utilize **Static Hermes** where applicable for bytecode pre-compilation.
* **Memory Management:** Monitor garbage collection triggers via `performance.memory` APIs in development to ensure zero leaks during navigation transitions.

---

## 3. Project Structure & Organization

### 3.1 Feature-Sliced Design (FSD)
Move away from type-based grouping (e.g., placing all components in a single folder) toward **Feature-Based Modules**. Each feature should be self-contained.

```text
/src
  /features
    /auth
      /api, /components, /hooks, /types
    /monitoring
      /api, /components, /hooks
  /shared
    /ui, /utils, /hooks
```

---

## 4. Routing & Navigation

### 4.1 Expo Router
* **Pattern:** Utilize file-based routing. The URL/Path must be the "source of truth" for the app state to facilitate seamless deep linking.
* **Type Safety:** Routes must be generated using `expo-router` typed routes. Avoid hardcoded strings for `router.push()`.
* **Shared Routes:** Use `(group)` folders to share layouts across functional areas without affecting the URL structure.

---

## 5. Performance Optimization Standards

### 5.1 Rendering Strategy
| Component | Standard Recommendation (2026) |
| :--- | :--- |
| **Lists** | `@shopify/flash-list` (Mandatory for lists > 20 items) |
| **Images** | `expo-image` (Supports WebP, AVIF, and aggressive caching) |
| **Animations** | `React Native Reanimated 3+` (Worklet-based, JS-thread independent) |

### 5.2 Resource Management
* **Assets:** All static images MUST be provided in **WebP** format to minimize memory footprint.
* **Bundle Size:** Utilize **EAS Update** with selective code-splitting to keep initial JS bundle sizes optimized.

---

## 6. State Management & Networking

### 6.1 Server State vs. UI State
* **Server Data:** Use **TanStack Query (React Query)**. Local state should never hold API data. Query caching and background fetching are the defaults.
* **Local UI State:** Use **Zustand** or **Jotai** for global UI state (e.g., themes, guest mode).
* **Persistence:** Use `react-native-mmkv` for high-speed key-value storage. Avoid `AsyncStorage` for performance-critical paths.

---

## 7. UI and User Experience (UX)

### 7.1 Styling Engine
* **Choice:** Use **Unistyles 2.0** (Runtime-based, high performance) or **NativeWind v4** (Tailwind-style).
* **Constraint:** Avoid inline styles and standard `StyleSheet.create` for complex conditional styling to prevent unnecessary object allocations.

### 7.2 Atomic Design System
Build a shared UI library in `/shared/ui` containing atomic components (Button, Input, Card) that are strictly themed. This ensures consistency across Android, iOS, and Web (Expo Web).

---

## 8. Deployment & CI/CD
* **Build System:** **EAS Build** is the standard for generating production binaries.
* **Environment Variables:** Use `expo-env` for type-safe environment variables across development, staging, and production environments.
* **Monitoring:** Integrate **Sentry** for error tracking and **PostHog** for feature flagging and session replay.

---
*End of Specification — Document Ref: RN-EXPO-2026-V1*
