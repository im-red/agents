# NEW_PROJECT_GUIDE: Creating a Capacitor + Ionic React Application

This guide covers how to create a **new** Capacitor + Ionic React application from scratch, based on the architecture established in the `mark` project.

---

## 1. Project Scaffolding

### 1.1 Create the Vite + React Project

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
```

### 1.2 Install Dependencies

```bash
# Ionic + Router
npm install @ionic/react @ionic/react-router ionicons react-router-dom@5

# Capacitor Core
npm install @capacitor/core @capacitor/app

# Capacitor Native Plugins (add what you need)
npm install @capacitor/status-bar @capacitor/splash-screen @capacitor/filesystem

# Dev Dependencies
npm install -D @capacitor/cli @capacitor/android sass
```

### 1.3 Initialize Capacitor

```bash
npx cap init "My App" "com.myapp.app" --web-dir dist
```

This creates `capacitor.config.ts`. Edit it:

```ts
import { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'com.myapp.app',
  appName: 'My App',
  webDir: 'dist',
  server: {
    androidScheme: 'https',
  },
  plugins: {
    SplashScreen: {
      launchShowDuration: 2000,
      launchAutoHide: false,
      backgroundColor: '#3880ff',
      showSpinner: false,
      androidScaleType: 'CENTER_CROP',
      splashFullScreen: true,
      splashImmersive: true,
    },
  },
};

export default config;
```

### 1.4 Add Android Platform

```bash
npx cap add android
```

### 1.5 Create `ionic.config.json`

```json
{
  "name": "my-app",
  "integrations": {},
  "type": "react"
}
```

---

## 2. Project Structure

```
my-app/
├── android/                  # Capacitor native Android project
├── public/
│   └── favicon.png
├── src/
│   ├── components/           # Reusable UI components
│   │   ├── SideMenu.tsx
│   │   └── SideMenu.scss     # Component-specific styles (optional)
│   ├── pages/                # Page-level components (one per route)
│   │   ├── HomePage.tsx
│   │   ├── HomePage.scss     # Page-specific styles (optional)
│   │   └── SettingsPage.tsx
│   ├── data/                 # State management (React Context)
│   │   └── AppContext.tsx
│   ├── models/               # Type definitions
│   │   └── index.ts
│   ├── hooks/                # Custom React hooks
│   │   ├── useLocalStorageState.ts
│   │   └── useAppVersion.ts
│   ├── theme/                # CSS custom properties (design tokens)
│   │   └── variables.css
│   ├── util/                 # Utility functions
│   │   └── helpers.ts
│   ├── App.tsx               # Providers + router + menu
│   ├── App.scss              # Global & shared styles only
│   └── main.tsx              # Ionic CSS imports + Capacitor init
├── capacitor.config.ts
├── ionic.config.json
├── index.html
├── package.json
├── tsconfig.json
└── vite.config.ts
```

### Directory Conventions

| Directory | Purpose | Naming |
|-----------|---------|--------|
| `src/pages/` | Route-level components + page styles | `*Page.tsx`, `*Page.scss` |
| `src/components/` | Reusable UI components + component styles | `PascalCase.tsx`, `PascalCase.scss` |
| `src/data/` | React Context providers | `*Context.tsx` |
| `src/models/` | TypeScript interfaces & types | `index.ts` or `*.ts` |
| `src/hooks/` | Custom hooks | `use*.ts` |
| `src/theme/` | Design tokens only | `variables.css` |
| `src/util/` | Pure utility functions | `*.ts` |

---

## 3. Configuration Files

### 3.1 `index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport"
    content="viewport-fit=cover, width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <meta http-equiv="Content-Security-Policy"
    content="default-src 'self' data: gap: https://ssl.gstatic.com 'unsafe-eval' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; font-src 'self' data:; media-src *; img-src 'self' data: content:; connect-src 'self' http://localhost:* https:;">
  <meta name="color-scheme" content="light dark">
  <meta name="format-detection" content="telephone=no">
  <meta name="msapplication-tap-highlight" content="no">
  <title>My App</title>
</head>
<body>
  <div id="root"></div>
  <script type="module" src="/src/main.tsx"></script>
</body>
</html>
```

Key points:
- **`id="root"`** — not `id="app"`, Ionic expects `root`
- **viewport meta** — `viewport-fit=cover` + `user-scalable=no` is required for Capacitor
- **CSP meta** — allows inline styles and Capacitor's `gap:` scheme

### 3.2 `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "jsx": "react-jsx",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"]
}
```

### 3.3 `vite.config.ts`

```ts
import react from '@vitejs/plugin-react';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3005,
    host: true,
  },
  build: {
    outDir: 'dist',
    emptyOutDir: true,
  },
  resolve: {
    alias: {
      '@': '/src',
    },
  },
});
```

- `host: true` — allows access from Android emulator via `10.0.2.2:3005`
- `outDir: 'dist'` — must match `capacitor.config.ts`'s `webDir`

---

## 4. Entry Points

### 4.1 `src/main.tsx`

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { StatusBar, Style } from '@capacitor/status-bar';
import { Capacitor } from '@capacitor/core';

// Ionic CSS — MUST come before any custom styles
import '@ionic/react/css/core.css';
import '@ionic/react/css/normalize.css';
import '@ionic/react/css/structure.css';
import '@ionic/react/css/typography.css';
import '@ionic/react/css/padding.css';
import '@ionic/react/css/float-elements.css';
import '@ionic/react/css/text-alignment.css';
import '@ionic/react/css/text-transformation.css';
import '@ionic/react/css/flex-utils.css';
import '@ionic/react/css/display.css';
import '@ionic/react/css/palettes/dark.system.css';

import App from './App';

const isNative = Capacitor.isNativePlatform();

async function initCapacitor() {
  if (isNative) {
    await StatusBar.setStyle({ style: Style.Light });
    await StatusBar.setBackgroundColor({ color: '#3880ff' });
  }
}

initCapacitor();

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

**Import order matters**: Ionic CSS → your custom CSS (imported in `App.tsx`).

### 4.2 `src/App.tsx`

```tsx
import React from 'react';
import { Route } from 'react-router-dom';
import {
  IonApp,
  IonRouterOutlet,
  setupIonicReact,
} from '@ionic/react';
import { IonReactRouter } from '@ionic/react-router';
import { SplashScreen } from '@capacitor/splash-screen';

import { AppProvider } from './data/AppContext';
import HomePage from './pages/HomePage';
import SettingsPage from './pages/SettingsPage';
import AboutPage from './pages/AboutPage';
import SideMenu from './components/SideMenu';

// Custom styles — AFTER Ionic CSS (imported in main.tsx)
import './theme/variables.css';
import './App.scss';

// Force Material Design mode (consistent look on both iOS & Android)
setupIonicReact({
  mode: 'md',
});

const App: React.FC = () => {
  React.useEffect(() => {
    const hideSplash = async () => {
      try {
        await SplashScreen.hide();
      } catch (err) {
        console.warn('Error hiding splash screen', err);
      }
    };
    hideSplash();
  }, []);

  return (
    <IonApp>
      <AppProvider>
        <IonReactRouter>
          <SideMenu />
          <IonRouterOutlet id="main">
            <Route exact path="/" component={HomePage} />
            <Route exact path="/settings" component={SettingsPage} />
            <Route exact path="/about" component={AboutPage} />
          </IonRouterOutlet>
        </IonReactRouter>
      </AppProvider>
    </IonApp>
  );
};

export default App;
```

Key elements:
- **`<IonApp>`** — root wrapper, required by Ionic
- **`<IonReactRouter>`** — wraps React Router with Ionic's page transition logic
- **`<IonRouterOutlet id="main">`** — renders routes with animated transitions; `id` links to `IonMenu`
- **`setupIonicReact({ mode: 'md' })`** — forces Material Design cross-platform
- **Providers** wrap the router so all pages can access state
- **`SplashScreen.hide()`** — dismiss the splash after React renders

---

## 5. Navigation & Routing

### 5.1 Route Definition

Define all routes inside `<IonRouterOutlet>`:

```tsx
<IonRouterOutlet id="main">
  <Route exact path="/" component={HomePage} />
  <Route exact path="/board/:id" component={BoardDetailPage} />
  <Route exact path="/settings" component={SettingsPage} />
</IonRouterOutlet>
```

- Always use `exact` — Ionic Router needs it
- Route params use `:paramName` syntax
- Pages render with **slide-in/slide-out transitions** automatically

### 5.2 Programmatic Navigation

```tsx
import { useHistory } from 'react-router-dom';

const history = useHistory();
history.push('/board/123');    // Forward (slide-in)
history.goBack();              // Back (slide-out)
```

### 5.3 Declarative Navigation (Side Menu, Buttons)

```tsx
<IonItem routerLink="/settings" routerDirection="root">
  <IonLabel>Settings</IonLabel>
</IonItem>
```

`routerDirection` values:
| Value | Behavior |
|-------|----------|
| `"forward"` | Push onto stack (slide-in) |
| `"back"` | Pop from stack (slide-out) |
| `"root"` | Reset stack to this page |
| `"none"` | Navigate without animation |

### 5.4 Route Params

```tsx
interface BoardDetailPageProps {
  match?: { params: { id: string } };
}

const BoardDetailPage: React.FC<BoardDetailPageProps> = ({ match }) => {
  const boardId = match?.params.id;
  // ...
};
```

### 5.5 Android Back Button

**No manual listener needed.** `IonRouterOutlet` automatically:
- Pops route stack on back press
- Closes modals/action sheets first
- Exits app when no more history

---

## 6. Page Anatomy

Every page **must** follow this structure:

```tsx
import React from 'react';
import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonButtons,
  IonBackButton,
  IonMenuButton,
  IonContent,
} from '@ionic/react';

const MyPage: React.FC = () => (
  <IonPage>
    <IonHeader>
      <IonToolbar>
        <IonButtons slot="start">
          {/* Use IonMenuButton on root pages, IonBackButton on sub-pages */}
          <IonMenuButton />
        </IonButtons>
        <IonTitle>My Page</IonTitle>
      </IonToolbar>
    </IonHeader>

    <IonContent fullscreen className="ion-padding">
      {/* iOS collapsible large title */}
      <IonHeader collapse="condense">
        <IonToolbar>
          <IonTitle size="large">My Page</IonTitle>
        </IonToolbar>
      </IonHeader>

      {/* Page content here */}
    </IonContent>
  </IonPage>
);

export default MyPage;
```

### Required Wrappers

| Component | Purpose |
|-----------|---------|
| `<IonPage>` | Root wrapper — enables page transitions |
| `<IonHeader>` | Fixed top bar — stays during scroll |
| `<IonToolbar>` | Container for title + buttons inside header |
| `<IonContent>` | Scrollable body — must be inside `<IonPage>` |

### Header Button Rules

| Page type | Left button | Right button (common) |
|-----------|-------------|----------------------|
| Root page | `<IonMenuButton>` | Action button (`<IonButton>`) |
| Sub-page | `<IonBackButton defaultHref="/">` | Save/Edit button |

### `IonContent` Attributes

| Attribute | When |
|-----------|------|
| `fullscreen` | Always — lets content scroll behind header |
| `className="ion-padding"` | Most pages (16px padding) |
| `className="ion-text-center"` | Centered content pages |

---

## 7. Common Components

### 7.1 Side Menu

```tsx
import {
  IonMenu, IonHeader, IonToolbar, IonTitle,
  IonContent, IonList, IonItem, IonLabel,
  IonIcon, IonMenuToggle, IonFooter,
} from '@ionic/react';
import { settings, cloudUpload } from 'ionicons/icons';

const SideMenu: React.FC = () => (
  <IonMenu contentId="main" menuId="side-menu" side="start">
    <IonHeader>
      <IonToolbar>
        <IonTitle>Menu</IonTitle>
      </IonToolbar>
    </IonHeader>
    <IonContent>
      <IonList>
        <IonMenuToggle autoHide={false}>
          <IonItem button routerLink="/settings" routerDirection="none">
            <IonIcon icon={settings} slot="start" />
            <IonLabel>Settings</IonLabel>
          </IonItem>
        </IonMenuToggle>
      </IonList>
    </IonContent>
    <IonFooter>
      <IonToolbar>
        <IonTitle size="small" className="ion-text-center">v1.0.0</IonTitle>
      </IonToolbar>
    </IonFooter>
  </IonMenu>
);

export default SideMenu;
```

- `contentId="main"` — must match `<IonRouterOutlet id="main">`
- `<IonMenuToggle>` — auto-closes menu on item click
- `routerDirection="none"` — no animation when navigating from menu

### 7.2 Modal

```tsx
const [showModal, setShowModal] = useState(false);
const [inputValue, setInputValue] = useState('');

<IonModal isOpen={showModal} onDidDismiss={() => setShowModal(false)}>
  <IonHeader>
    <IonToolbar>
      <IonTitle>Create Item</IonTitle>
      <IonButtons slot="end">
        <IonButton onClick={() => setShowModal(false)}>Cancel</IonButton>
      </IonButtons>
    </IonToolbar>
  </IonHeader>
  <IonContent className="ion-padding">
    <IonList>
      <IonItem>
        <IonInput
          label="Name"
          labelPlacement="stacked"
          placeholder="Enter name"
          value={inputValue}
          onIonInput={(e) => setInputValue(e.detail.value ?? '')}
          autofocus
        />
      </IonItem>
    </IonList>
    <IonButton expand="block" onClick={handleSave} disabled={!inputValue.trim()}>
      Save
    </IonButton>
  </IonContent>
</IonModal>
```

### 7.3 Alert

```tsx
<IonAlert
  isOpen={showAlert}
  onDidDismiss={() => setShowAlert(false)}
  header="Delete Item?"
  message="This action cannot be undone."
  buttons={[
    { text: 'Cancel', role: 'cancel' },
    { text: 'Delete', role: 'destructive', handler: handleDelete },
  ]}
/>
```

### 7.4 Action Sheet

```tsx
<IonActionSheet
  isOpen={showActionSheet}
  onDidDismiss={() => setShowActionSheet(false)}
  buttons={[
    { text: 'Rename', icon: create, handler: handleRename },
    { text: 'Delete', icon: trash, role: 'destructive', handler: () => setShowDeleteAlert(true) },
    { text: 'Cancel', role: 'cancel' },
  ]}
/>
```

### 7.5 FAB (Floating Action Button)

```tsx
<IonFab vertical="bottom" horizontal="end" slot="fixed">
  <IonFabButton onClick={() => setShowModal(true)}>
    <IonIcon icon={add} />
  </IonFabButton>
</IonFab>
```

### 7.6 Icons

```tsx
import { add, create, trash, settings, homeOutline, ellipsisVertical } from 'ionicons/icons';

<IonIcon icon={add} />
<IonIcon icon={homeOutline} slot="start" />
```

Convention: Use `outline` variants for inactive, filled for active.

### Quick Component Reference

| Need | Use |
|------|-----|
| Page wrapper | `<IonPage>` |
| Top bar | `<IonHeader>` + `<IonToolbar>` |
| Back button | `<IonBackButton defaultHref="/">` |
| Menu button | `<IonMenuButton>` |
| Side menu | `<IonMenu contentId="main">` |
| Modal | `<IonModal isOpen={bool} onDidDismiss={fn}>` |
| Alert dialog | `<IonAlert isOpen={bool} onDidDismiss={fn}>` |
| Bottom sheet | `<IonActionSheet isOpen={bool} onDidDismiss={fn}>` |
| Floating button | `<IonFab>` + `<IonFabButton>` |
| List | `<IonList>` + `<IonItem>` |
| Input field | `<IonInput label="..." labelPlacement="stacked">` inside `<IonItem>` |
| Button | `<IonButton expand="block">` or `<IonButton fill="clear">` |
| Icon | `<IonIcon icon={iconName} slot="start">` |

---

## 8. State Management

### 8.1 Context Pattern

Store state in `src/data/*Context.tsx`:

```tsx
// src/data/AppContext.tsx
import { createContext, useContext, ReactNode } from 'react';
import useLocalStorageState from '../hooks/useLocalStorageState';

interface Item {
  id: string;
  name: string;
}

interface AppContextType {
  items: Item[];
  addItem: (name: string) => void;
  deleteItem: (id: string) => void;
}

const AppContext = createContext<AppContextType | null>(null);

export const AppProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  const [items, setItems] = useLocalStorageState<Item[]>('items', []);

  const addItem = (name: string) => {
    setItems(prev => [...prev, { id: crypto.randomUUID(), name }]);
  };

  const deleteItem = (id: string) => {
    setItems(prev => prev.filter(item => item.id !== id));
  };

  return (
    <AppContext.Provider value={{ items, addItem, deleteItem }}>
      {children}
    </AppContext.Provider>
  );
};

export const useApp = () => {
  const ctx = useContext(AppContext);
  if (!ctx) throw new Error('useApp must be used within AppProvider');
  return ctx;
};
```

### 8.2 `useLocalStorageState` Hook

```tsx
// src/hooks/useLocalStorageState.ts
import { useState, useEffect } from 'react';

function useLocalStorageState<T>(key: string, defaultValue: T): [T, React.Dispatch<React.SetStateAction<T>>] {
  const [state, setState] = useState<T>(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : defaultValue;
    } catch {
      return defaultValue;
    }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(state));
  }, [key, state]);

  return [state, setState];
}

export default useLocalStorageState;
```

### 8.3 Provider Nesting in App.tsx

```tsx
<IonApp>
  <AppProvider>
    <AnotherProvider>
      <IonReactRouter>
        {/* routes */}
      </IonReactRouter>
    </AnotherProvider>
  </AppProvider>
</IonApp>
```

Providers wrap the router so all pages can `useContext`.

---

## 9. Styling

### 9.1 Architecture

| File | Purpose |
|------|---------|
| `src/theme/variables.css` | Design tokens only (CSS custom properties) |
| `src/App.scss` | Global & shared styles only (html/body resets, shared utility classes) |
| `src/components/*.scss` | Component-specific styles (imported in component files) |
| `src/pages/*.scss` | Page-specific styles (imported in page files) |

**Rule**: If a CSS class is only used by one component or page, put it in that component/page's own `.scss` file. Only keep truly global styles (html/body resets) and shared utility classes (used by 2+ components/pages) in `App.scss`.

### 9.2 Design Tokens (`src/theme/variables.css`)

```css
:root {
  --primary: #3880ff;
  --primary-dark: #3171e0;
  --background: #ffffff;
  --surface: #f4f5f8;
  --text-main: #121212;
  --text-muted: #666666;
  --text-secondary: #9ca3af;
  --border-color: #e2e8f0;
  --shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  --shadow-lg: 0 8px 24px rgba(0, 0, 0, 0.12);
  --success: #22c55e;
  --error: #eb445a;
}
```

### 9.3 Ionic Utility Classes (Prefer Over Custom CSS)

| Class | Effect |
|-------|--------|
| `ion-padding` | 16px padding |
| `ion-margin` | 16px margin |
| `ion-margin-top` | 16px top margin |
| `ion-text-center` | `text-align: center` |
| `ion-text-start` | `text-align: start` |
| `ion-text-end` | `text-align: end` |

### 9.4 Import Order

In `main.tsx` (Ionic CSS first):
```tsx
import '@ionic/react/css/core.css';
import '@ionic/react/css/normalize.css';
// ... all Ionic CSS imports
```

In `App.tsx` (your styles after):
```tsx
import './theme/variables.css';
import './App.scss';
```

### 9.5 SCSS Support

`sass` is included in devDependencies. Use `.scss` for page-specific and component-specific styles:

```tsx
// In a page component
import './MyPage.scss';

// In a reusable component
import './MyComponent.scss';
```

---

## 10. Capacitor Native Integration

### 10.1 Status Bar (`main.tsx`)

```tsx
import { StatusBar, Style } from '@capacitor/status-bar';
import { Capacitor } from '@capacitor/core';

const isNative = Capacitor.isNativePlatform();

if (isNative) {
  StatusBar.setStyle({ style: Style.Light });
  StatusBar.setBackgroundColor({ color: '#3880ff' });
}
```

### 10.2 Splash Screen (`App.tsx`)

```tsx
import { SplashScreen } from '@capacitor/splash-screen';

React.useEffect(() => {
  SplashScreen.hide();
}, []);
```

### 10.3 Common Plugins

| Plugin | Install | Use Case |
|--------|---------|----------|
| `@capacitor/status-bar` | `npm install @capacitor/status-bar` | Status bar color/style |
| `@capacitor/splash-screen` | `npm install @capacitor/splash-screen` | Splash screen control |
| `@capacitor/filesystem` | `npm install @capacitor/filesystem` | File read/write |
| `@capacitor/app` | included with core | App lifecycle, back button |
| `@capacitor/haptics` | `npm install @capacitor/haptics` | Vibration feedback |
| `@capacitor/share` | `npm install @capacitor/share` | System share sheet |
| `@capacitor/camera` | `npm install @capacitor/camera` | Camera access |

### 10.4 Build & Deploy Cycle

```bash
# Development
npm run dev

# Build for production
npm run build

# Sync to native
npx cap sync

# Open in Android Studio
npx cap open android

# Live reload on device (optional)
npx cap run android --livereload --external
```

---

## 11. TypeScript Models (`src/models/`)

```ts
// src/models/index.ts
export interface Board {
  id: string;
  name: string;
  marks: Record<string, Mark>;
  createdAt: number;
  updatedAt: number;
}

export interface Mark {
  date: string;
  value: number;
  note?: string;
}
```

Use `crypto.randomUUID()` for IDs, `Date.now()` for timestamps.

---

## 12. Complete `package.json` Reference

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "sync": "npx cap sync",
    "open:android": "npx cap open android"
  },
  "dependencies": {
    "@capacitor/app": "^6.0.0",
    "@capacitor/core": "^6.0.0",
    "@capacitor/filesystem": "^6.0.0",
    "@capacitor/splash-screen": "^6.0.0",
    "@capacitor/status-bar": "^6.0.0",
    "@ionic/react": "^8.8.0",
    "@ionic/react-router": "^8.8.0",
    "ionicons": "^7.4.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-router-dom": "^5.3.4"
  },
  "devDependencies": {
    "@capacitor/android": "^6.2.0",
    "@capacitor/cli": "^6.0.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "@types/react-router-dom": "^5.3.3",
    "@vitejs/plugin-react": "^4.7.0",
    "sass": "^1.99.0",
    "typescript": "^5.0.0",
    "vite": "^6.4.0"
  }
}
```

---

## 13. Quick Start Checklist

- [ ] `npm create vite@latest my-app -- --template react-ts`
- [ ] `cd my-app && npm install`
- [ ] Install Ionic + Capacitor deps (Section 1.2)
- [ ] `npx cap init "My App" "com.myapp.app" --web-dir dist`
- [ ] `npx cap add android`
- [ ] Create `ionic.config.json`
- [ ] Replace `index.html` with Capacitor version (Section 3.1)
- [ ] Create `src/theme/variables.css` with design tokens
- [ ] Create `src/App.scss` for global & shared styles only
- [ ] Set up `src/main.tsx` with Ionic CSS imports + Capacitor init
- [ ] Set up `src/App.tsx` with `IonApp` + `IonReactRouter` + routes
- [ ] Create `src/pages/` with first page following page anatomy (Section 6)
- [ ] Create `src/data/AppContext.tsx` with state + `useLocalStorageState`
- [ ] Create `src/components/SideMenu.tsx` (optional)
- [ ] `npm run build && npx cap sync && npx cap open android`
