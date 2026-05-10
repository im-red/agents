# MIGRATION_GUIDE: Migrating Capacitor React Apps to Ionic

This guide documents how to migrate existing Capacitor React apps that use **custom UI components** (hand-built headers, side menus, overlays) to use **Ionic components** instead — gaining native transitions, automatic back-button handling, and platform-adaptive UI.

## The Migration Path

Your existing apps (`chengyujielong`, `memo-pads`, etc.) are already **Capacitor React apps** — they have Capacitor configured, native plugins working, splash screens, and haptics. What they lack is Ionic's UI framework.

| Before (Current) | After (Target) |
|-------------------|----------------|
| `useState`-based view switching | `IonReactRouter` + URL-based routing |
| Custom `<header>` with manual buttons | `<IonHeader>` + `<IonToolbar>` + `<IonBackButton>` |
| Custom side menu (hand-built + touch gestures) | `<IonMenu>` (swipe, backdrop, auto-close) |
| Custom overlay/modal (`position: fixed`) | `<IonModal>` (slide-up animation) |
| Custom alert (`window.alert` / `confirm`) | `<IonAlert>` (styled, accessible) |
| Custom action sheet (dropdown div) | `<IonActionSheet>` (bottom sheet) |
| Manual Android back-button listener (100+ lines) | Auto-handled by `IonRouterOutlet` |
| Monolithic `styles.css` (40-50KB) | `theme/variables.css` + `App.scss` + component/page SCSS |
| All components in `src/components/` | Separated into `src/pages/` + `src/components/` |

---

## 1. Project Structure

### Before (Custom UI Capacitor App)

```
src/
├── components/          # ALL components mixed (pages + UI + overlays)
├── hooks/               # Custom hooks
├── utils/               # Utility functions
├── types/               # Type definitions (or types.ts at root)
├── App.tsx              # Giant view-switcher with useState
├── main.tsx             # Entry point
└── styles.css           # Monolithic CSS (40-50KB)
```

### After (Capacitor + Ionic App)

```
src/
├── components/          # Reusable UI components only
│   ├── BoardCard.tsx
│   ├── BoardCard.scss
│   ├── Calendar.tsx
│   ├── Calendar.scss
│   ├── MarkSelector.tsx
│   ├── MarkSelector.scss
│   └── SideMenu.tsx
├── pages/               # Page-level components (one per route)
│   ├── HomePage.tsx
│   ├── HomePage.scss
│   ├── BoardDetailPage.tsx
│   ├── BoardDetailPage.scss
│   ├── MarkManagementPage.tsx
│   ├── MarkManagementPage.scss
│   ├── SettingsPage.tsx
│   └── AboutPage.tsx
│   └── AboutPage.scss
├── data/                # State management (was context/ or inline)
│   ├── BoardContext.tsx
│   └── MarkSuiteContext.tsx
├── models/              # Type definitions (was types/)
│   └── index.ts
├── theme/               # CSS custom properties
│   └── variables.css
├── util/                # Utility functions (was utils/)
│   ├── export.ts
│   └── import.ts
├── hooks/               # Custom React hooks
│   ├── useLocalStorageState.ts
│   └── useAppVersion.ts
├── App.tsx              # Providers + router + menu
├── App.scss             # Global & shared styles only
└── main.tsx             # Ionic CSS imports + Capacitor init
```

### Directory Rename Map

| Before | After | Why |
|--------|-------|-----|
| `src/components/*Page.tsx` | `src/pages/` | Separate routes from reusable UI |
| `src/context/` | `src/data/` | Follows Ionic demo convention |
| `src/types/` | `src/models/` | Follows Ionic demo convention |
| `src/utils/` | `src/util/` | Follows Ionic demo convention |
| `src/styles.css` | `src/theme/variables.css` + `src/App.scss` | Split tokens from styles |

---

## 2. Navigation: From useState to Router

### The Problem with useState View Switching

Your current apps use a `view` state to switch screens:

```tsx
// ❌ CURRENT: useState-based navigation
const [view, setView] = useState<'home' | 'game' | 'settings'>('home');

return (
  <>
    {view === 'home' && <HomePage onGoSettings={() => setView('settings')} />}
    {view === 'game' && <GamePage onBack={() => setView('home')} />}
  </>
);
```

Problems:
- No browser history — can't use browser back
- No deep linking — can't open `/board/123` directly
- Manual Android back-button handling (100+ lines of `if/else`)
- No page transitions — screens just appear/disappear
- Can't pass URL params

### The Ionic Router Pattern

```tsx
// ✅ AFTER: IonReactRouter
import { IonReactRouter } from '@ionic/react-router';
import { IonRouterOutlet } from '@ionic/react';
import { Route } from 'react-router-dom';

<IonApp>
  <DataProvider>
    <IonReactRouter>
      <SideMenu />
      <IonRouterOutlet id="main">
        <Route exact path="/" component={HomePage} />
        <Route exact path="/board/:id" component={BoardDetailPage} />
        <Route exact path="/settings" component={SettingsPage} />
      </IonRouterOutlet>
    </IonReactRouter>
  </DataProvider>
</IonApp>
```

Benefits:
- **Automatic page transitions** (slide-in/slide-out)
- **Automatic Android back-button** — no manual listener
- **URL-based navigation** — deep linking works
- **`<IonBackButton>`** uses browser history automatically

### Navigation Between Pages

```tsx
import { useHistory } from 'react-router-dom';

const history = useHistory();
history.push('/board/123');    // Forward (slide-in)
history.goBack();              // Back (slide-out)
```

In side menu items:
```tsx
<IonItem routerLink="/" routerDirection="root">
  <IonLabel>Home</IonLabel>
</IonItem>
```

### Route Params

```tsx
interface BoardDetailPageProps {
  match?: { params: { id: string } };
}

const BoardDetailPage: React.FC<BoardDetailPageProps> = ({ match }) => {
  const boardId = match?.params.id;
};
```

### Eliminating the Manual Back-Button Listener

Your current apps have 100+ lines like this:
```tsx
// ❌ BEFORE: Manual back-button handling
CapacitorApp.addListener('backButton', ({ canGoBack }) => {
  if (isSideMenuOpen) { setIsSideMenuOpen(false); return; }
  if (isAddMemoOpen) { setIsAddMemoOpen(false); return; }
  if (view === 'game') { setView('home'); return; }
  if (view === 'settings') { setView('home'); return; }
  // ... 20 more conditions
  if (canGoBack) { window.history.back(); }
  else { CapacitorApp.exitApp(); }
});
```

With Ionic Router, this is **completely unnecessary**. `IonRouterOutlet` handles it automatically:
- Back button pops the route stack
- Modals/action sheets close first
- If no more history, the app exits

---

## 3. Page Anatomy

### Before (Custom HTML)

```tsx
// ❌ BEFORE: Hand-built page
function GamePage({ onBack }) {
  return (
    <div className="app-shell">
      <header className="app-header">
        <button className="back-btn" onClick={onBack}>←</button>
        <div className="header-title"><h1>Game</h1></div>
      </header>
      <main>
        <section className="container">
          {/* content */}
        </section>
      </main>
    </div>
  );
}
```

### After (Ionic)

```tsx
// ✅ AFTER: Ionic page
import {
  IonPage, IonHeader, IonToolbar, IonTitle,
  IonButtons, IonBackButton, IonContent,
} from '@ionic/react';

const GamePage: React.FC = () => (
  <IonPage>
    <IonHeader>
      <IonToolbar>
        <IonButtons slot="start">
          <IonBackButton defaultHref="/" />
        </IonButtons>
        <IonTitle>Game</IonTitle>
      </IonToolbar>
    </IonHeader>
    <IonContent fullscreen className="ion-padding">
      {/* Collapsible large title for iOS */}
      <IonHeader collapse="condense">
        <IonToolbar>
          <IonTitle size="large">Game</IonTitle>
        </IonToolbar>
      </IonHeader>
      {/* content */}
    </IonContent>
  </IonPage>
);
```

### Header Button Rules

| Page type | Left button | Example |
|-----------|-------------|---------|
| Root page (no parent) | `<IonMenuButton>` | HomePage |
| Sub-page (has parent) | `<IonBackButton defaultHref="/">` | SettingsPage, BoardDetailPage |

### Key `IonContent` Attributes

| Attribute | When to use |
|-----------|-------------|
| `fullscreen` | Always (enables content to scroll behind header) |
| `className="ion-padding"` | Most pages (adds 16px padding) |
| `className="ion-text-center"` | Centered content pages (About) |

---

## 4. Replacing Custom Components with Ionic

### Side Menu

```tsx
// ❌ BEFORE: Custom side menu (50+ lines of CSS + touch handling)
const [isSideMenuOpen, setIsSideMenuOpen] = useState(false);
// ... backdrop click handler
// ... touch swipe handler
// ... 30 lines of CSS

{isSideMenuOpen && <div className="side-menu-backdrop" onClick={() => setIsSideMenuOpen(false)} />}
<div className={`side-menu ${isSideMenuOpen ? 'side-menu--open' : ''}`}>
  <div className="side-menu-header"><h2>Menu</h2><button>×</button></div>
  <div className="side-menu-content">
    <button className="side-menu-item" onClick={() => { setIsSideMenuOpen(false); setView('settings'); }}>⚙️ Settings</button>
  </div>
</div>
```

```tsx
// ✅ AFTER: IonMenu (built-in swipe, backdrop, auto-close)
import { IonMenu, IonHeader, IonToolbar, IonTitle, IonContent, IonList, IonItem, IonLabel, IonIcon } from '@ionic/react';
import { settingsOutline, trashOutline } from 'ionicons/icons';

const SideMenu: React.FC = () => (
  <IonMenu side="start" menuId="side-menu" contentId="main">
    <IonHeader>
      <IonToolbar>
        <IonTitle>Menu</IonTitle>
      </IonToolbar>
    </IonHeader>
    <IonContent>
      <IonList>
        <IonItem routerLink="/" routerDirection="root">
          <IonIcon icon={homeOutline} slot="start" />
          <IonLabel>Home</IonLabel>
        </IonItem>
        <IonItem routerLink="/settings" routerDirection="root">
          <IonIcon icon={settingsOutline} slot="start" />
          <IonLabel>Settings</IonLabel>
        </IonItem>
      </IonList>
    </IonContent>
  </IonMenu>
);
```

Key properties:
- `contentId="main"` — must match `IonRouterOutlet`'s `id`
- `routerDirection="root"` — resets navigation stack
- `routerLink` on `<IonItem>` — auto-closes menu and navigates
- Swipe-to-close and backdrop click are built-in

### Modals / Overlays

```tsx
// ❌ BEFORE: Custom overlay
{isAddMemoOpen && (
  <div className="overlay-backdrop" onClick={() => setIsAddMemoOpen(false)}>
    <div className="overlay-panel" onClick={e => e.stopPropagation()}>
      <div className="overlay-header">
        <h2>Add Memo</h2>
        <button onClick={() => setIsAddMemoOpen(false)}>×</button>
      </div>
      <div className="overlay-content">...</div>
    </div>
  </div>
)}
```

```tsx
// ✅ AFTER: IonModal
<IonModal isOpen={isAddMemoOpen} onDidDismiss={() => setIsAddMemoOpen(false)}>
  <IonHeader>
    <IonToolbar>
      <IonTitle>Add Memo</IonTitle>
      <IonButtons slot="end">
        <IonButton onClick={() => setIsAddMemoOpen(false)}>Cancel</IonButton>
      </IonButtons>
    </IonToolbar>
  </IonHeader>
  <IonContent className="ion-padding">
    {/* form content */}
    <IonButton expand="block" onClick={handleSave}>Save</IonButton>
  </IonContent>
</IonModal>
```

### Alerts

```tsx
// ❌ BEFORE: window.alert / window.confirm
alert('Import successful!');
if (confirm('Delete this item?')) { handleDelete(); }
```

```tsx
// ✅ AFTER: IonAlert
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

### Action Sheets / Context Menus

```tsx
// ❌ BEFORE: Custom dropdown
{isHeaderMenuOpen && (
  <div className="header-menu-dropdown">
    <button onClick={handleEdit}>Edit</button>
    <button onClick={handleDelete}>Delete</button>
  </div>
)}
```

```tsx
// ✅ AFTER: IonActionSheet
<IonActionSheet
  isOpen={showActionSheet}
  onDidDismiss={() => setShowActionSheet(false)}
  buttons={[
    { text: 'Rename', icon: create, handler: handleRename },
    { text: 'Export as Image', icon: image, handler: handleExport },
    { text: 'Delete', icon: trash, role: 'destructive', handler: () => setShowDeleteAlert(true) },
    { text: 'Cancel', role: 'cancel' },
  ]}
/>
```

### Quick Reference: Custom → Ionic

| Custom Component | Replace With |
|-----------------|--------------|
| `<div className="app-shell">` | `<IonPage>` |
| `<header className="app-header">` | `<IonHeader>` + `<IonToolbar>` |
| `<button className="back-btn">←</button>` | `<IonBackButton defaultHref="/">` |
| `<button className="menu-trigger-btn">☰</button>` | `<IonMenuButton>` |
| Custom side menu + touch handling | `<IonMenu>` |
| Custom overlay/backdrop | `<IonModal>` |
| `window.alert()` / `confirm()` | `<IonAlert>` |
| Custom dropdown menu | `<IonActionSheet>` |
| `<button>` | `<IonButton>` |
| `<ul>` + `<li>` | `<IonList>` + `<IonItem>` |
| `<input>` | `<IonInput>` inside `<IonItem>` |
| Custom floating button | `<IonFab>` + `<IonFabButton>` |

---

## 5. Styling

### Before: Monolithic CSS

```css
/* styles.css — 40-50KB single file */
:root { --primary: #3880ff; ... }
.app-header { ... }
.side-menu { ... }
.overlay { ... }
.calendar { ... }
/* everything in one file */
```

### After: Split Architecture

```
src/theme/variables.css       ← Design tokens only (CSS custom properties)
src/App.scss                  ← Global & shared styles only (html/body, shared utility classes)
src/components/Calendar.scss  ← Component-specific styles
src/components/BoardCard.scss ← Component-specific styles
src/components/MarkSelector.scss ← Component-specific styles
src/pages/HomePage.scss       ← Page-specific styles
src/pages/BoardDetailPage.scss   ← Page-specific styles
src/pages/MarkManagementPage.scss ← Page-specific styles
src/pages/AboutPage.scss      ← Page-specific styles
```

**`src/theme/variables.css`** — Extract all `:root` variables:
```css
:root {
  --primary: #3880ff;
  --primary-dark: #3171e0;
  --background: #ffffff;
  --surface: #f4f5f8;
  --text-main: #121212;
  --text-muted: #666666;
  --border-color: #e2e8f0;
  --shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  --error: #eb445a;
}
```

**`src/App.scss`** — Only truly global styles (html/body resets) and shared utility classes used by multiple components/pages (e.g., `.mark-grid`, `.overlay__text`). Component-specific and page-specific styles belong in their own `.scss` files.

**Component styles** (`src/components/*.scss`) — Styles scoped to a single reusable component. Imported directly in the component file:
```tsx
// In a component file
import './Calendar.scss';
```

**Page styles** (`src/pages/*.scss`) — Styles scoped to a single page route. Imported directly in the page file:
```tsx
// In a page file
import './BoardDetailPage.scss';
```

### CSS You Can DELETE After Migration

When switching to Ionic, these custom CSS classes become unnecessary:

| Delete | Reason |
|-------|--------|
| `.app-header`, `.back-btn`, `.menu-trigger-btn` | Replaced by `IonToolbar` + `IonBackButton` + `IonMenuButton` |
| `.side-menu`, `.side-menu--open`, `.side-menu-backdrop` | Replaced by `IonMenu` |
| `.overlay-backdrop`, `.overlay-panel`, `.overlay-header` | Replaced by `IonModal` |
| `.app-shell` | Replaced by `IonPage` |

### Ionic Utility Classes (Prefer Over Custom CSS)

| Class | Effect |
|-------|--------|
| `ion-padding` | 16px padding |
| `ion-margin` | 16px margin |
| `ion-margin-top` | 16px top margin |
| `ion-text-center` | `text-align: center` |

### Import Order

In `main.tsx` (Ionic CSS must come first):
```tsx
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
```

In `App.tsx` (your custom styles after Ionic):
```tsx
import './theme/variables.css';
import './App.scss';
```

### SCSS Support

```bash
npm install -D sass
```

Then use `.scss` files for page-specific and component-specific styles:
```tsx
// In a page component
import './BoardDetailPage.scss';

// In a reusable component
import './Calendar.scss';
```

---

## 6. State Management

### Before: State in App.tsx

Your current apps store all state in `App.tsx` and pass it down via props:

```tsx
// ❌ BEFORE: Prop drilling
function App() {
  const [notebooks, setNotebooks] = useLocalStorageState('notebooks', []);
  const [memos, setMemos] = useLocalStorageState('memos', []);
  // ... 20 more state variables

  return (
    <>
      {view === 'home' && (
        <NotebookList
          notebooks={notebooks}
          onSelect={handleSelectNotebook}
          onDelete={handleDeleteNotebook}
          onEdit={handleEditNotebook}
          // ... 10 more props
        />
      )}
    </>
  );
}
```

### After: Context Pattern in `src/data/`

```tsx
// src/data/BoardContext.tsx
import { createContext, useContext } from 'react';
import useLocalStorageState from '../hooks/useLocalStorageState';

interface BoardContextType {
  boards: Board[];
  createBoard: (name: string) => Board;
  updateBoard: (id: string, updates: Partial<Board>) => void;
  deleteBoard: (id: string) => void;
}

const BoardContext = createContext<BoardContextType | null>(null);

export const BoardProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  const [boards, setBoards] = useLocalStorageState<Board[]>('boards', []);

  const createBoard = (name: string) => {
    const board: Board = { id: crypto.randomUUID(), name, marks: {} };
    setBoards(prev => [...prev, board]);
    return board;
  };

  return (
    <BoardContext.Provider value={{ boards, createBoard, updateBoard, deleteBoard }}>
      {children}
    </BoardContext.Provider>
  );
};

export const useBoard = () => {
  const ctx = useContext(BoardContext);
  if (!ctx) throw new Error('useBoard must be used within BoardProvider');
  return ctx;
};
```

In `App.tsx`:
```tsx
<IonApp>
  <BoardProvider>
    <MarkSuiteProvider>
      <IonReactRouter>
        {/* routes */}
      </IonReactRouter>
    </MarkSuiteProvider>
  </BoardProvider>
</IonApp>
```

### `useLocalStorageState` Hook

Already shared across your projects — keep it in `src/hooks/`:
```tsx
function useLocalStorageState<T>(key: string, defaultValue: T): [T, React.Dispatch<React.SetStateAction<T>>] {
  const [state, setState] = useState<T>(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : defaultValue;
    } catch { return defaultValue; }
  });
  useEffect(() => { localStorage.setItem(key, JSON.stringify(state)); }, [key, state]);
  return [state, setState];
}
```

---

## 7. Capacitor Configuration (Already Done)

Your existing Capacitor setup is already correct. Only minor changes needed:

### `index.html` Change

```diff
- <div id="app"></div>
+ <div id="root"></div>
```

Also update viewport meta:
```html
<meta name="viewport" content="viewport-fit=cover, width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=no" />
```

### `main.tsx` Changes

```diff
+ // Ionic CSS imports (add BEFORE your custom styles)
+ import '@ionic/react/css/core.css';
+ import '@ionic/react/css/normalize.css';
+ import '@ionic/react/css/structure.css';
+ import '@ionic/react/css/typography.css';
+ import '@ionic/react/css/padding.css';
+ import '@ionic/react/css/float-elements.css';
+ import '@ionic/react/css/text-alignment.css';
+ import '@ionic/react/css/text-transformation.css';
+ import '@ionic/react/css/flex-utils.css';
+ import '@ionic/react/css/display.css';
+ import '@ionic/react/css/palettes/dark.system.css';

- import './styles.css';

- const rootElement = document.getElementById('app');
+ const rootElement = document.getElementById('root');
```

### `capacitor.config.ts` — No Changes Needed

Your existing config is already correct. Keep it as-is.

### Remove Manual Back-Button Listener

```diff
- useEffect(() => {
-   const handle = await CapacitorApp.addListener('backButton', ({ canGoBack }) => {
-     if (isSideMenuOpen) { setIsSideMenuOpen(false); return; }
-     if (view === 'game') { setView('home'); return; }
-     // ... 20 more conditions
-   });
- }, [view, isSideMenuOpen, ...]);
```

`IonRouterOutlet` handles this automatically.

---

## 8. Migration Checklist

### Step 1: Install Ionic Dependencies

- [ ] `npm install @ionic/react @ionic/react-router ionicons`
- [ ] `npm install -D sass`
- [ ] Create `ionic.config.json`

### Step 2: Restructure Source

- [ ] Create `src/pages/` and move `*Page.tsx` from `src/components/`
- [ ] Keep reusable components in `src/components/`
- [ ] Rename `src/context/` → `src/data/` (or create `src/data/` from App.tsx state)
- [ ] Rename `src/types/` → `src/models/`
- [ ] Rename `src/utils/` → `src/util/`
- [ ] Extract CSS variables from `styles.css` → `src/theme/variables.css`
- [ ] Move global/shared styles → `src/App.scss` (only html/body resets and shared utility classes)
- [ ] Move component-specific styles → `src/components/*.scss` (imported in component files)
- [ ] Move page-specific styles → `src/pages/*.scss` (imported in page files)
- [ ] Delete `styles.css`

### Step 3: Convert Navigation

- [ ] Replace `useState<ViewType>` with `IonReactRouter` + routes
- [ ] Replace `setView('page')` calls with `history.push('/page')`
- [ ] Replace `onBack={() => setView('home')}` with `<IonBackButton>`
- [ ] Delete manual `CapacitorApp.addListener('backButton', ...)` listener

### Step 4: Convert Pages to IonPage

- [ ] Wrap each page in `<IonPage>` + `<IonHeader>` + `<IonContent>`
- [ ] Replace custom `<header className="app-header">` with `<IonToolbar>`
- [ ] Replace `←` back button with `<IonBackButton defaultHref="/">`
- [ ] Replace `☰` menu button with `<IonMenuButton>`
- [ ] Add `fullscreen` and `className="ion-padding"` to `<IonContent>`

### Step 5: Replace Custom UI with Ionic

- [ ] Replace custom side menu with `<IonMenu>`
- [ ] Replace custom overlays with `<IonModal>`
- [ ] Replace `window.alert` / `confirm` with `<IonAlert>`
- [ ] Replace custom dropdown menus with `<IonActionSheet>`
- [ ] Replace `<button>` with `<IonButton>`
- [ ] Replace `<ul>/<li>` lists with `<IonList>/<IonItem>`
- [ ] Delete now-unused CSS (app-header, side-menu, overlay classes)

### Step 6: Update Config

- [ ] `index.html`: `<div id="root">` + updated viewport meta
- [ ] `main.tsx`: Add Ionic CSS imports, change `getElementById('root')`
- [ ] Delete `<link rel="stylesheet" href="/src/styles.css">` from `index.html`

### Step 7: Build & Test

- [ ] `npm run build` — verify no errors
- [ ] `npx cap sync` — copy build to native
- [ ] Test page transitions work (slide-in/slide-out)
- [ ] Test Android back button works automatically
- [ ] Test side menu swipe-to-close
- [ ] Test modals dismiss correctly

---

## 9. Common Patterns

### Modal with Form

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

### Action Sheet

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

### Confirm Alert

```tsx
<IonAlert
  isOpen={showAlert}
  onDidDismiss={() => setShowAlert(false)}
  header="Delete Item?"
  message="This action cannot be undone."
  buttons={[
    { text: 'Cancel', role: 'cancel' },
    { text: 'Delete', role: 'destructive', handler: handleConfirmDelete },
  ]}
/>
```

### Icons

```tsx
import { add, create, trash, ellipsisVertical, homeOutline, settingsOutline } from 'ionicons/icons';

<IonIcon icon={add} />
<IonIcon icon={homeOutline} slot="start" />
```

Convention: Use `outline` variants for inactive icons, filled for active.

---

## 10. Key Differences Summary

| Aspect | Current (Custom UI) | After (Ionic) |
|--------|-------------------|----------------|
| Navigation | `useState<ViewType>` | `IonReactRouter` + routes |
| Page wrapper | `<div className="app-shell">` | `<IonPage>` |
| Header | Custom `<header>` + buttons | `<IonHeader>` + `<IonToolbar>` |
| Back button | `←` + `onBack` callback | `<IonBackButton>` (auto history) |
| Menu button | `☰` + `setIsSideMenuOpen` | `<IonMenuButton>` |
| Side menu | Custom div + touch handling | `<IonMenu>` (swipe built-in) |
| Modal | Custom overlay + backdrop | `<IonModal>` (animation built-in) |
| Alert | `window.alert()` | `<IonAlert>` |
| Context menu | Custom dropdown | `<IonActionSheet>` |
| Android back | 100+ line manual listener | Auto by `IonRouterOutlet` |
| Transitions | None (instant swap) | Slide-in/slide-out |
| Deep linking | Not possible | URL-based (`/board/123`) |
| State access | Prop drilling | Context + `useXyz()` hooks |
| Styles | Monolithic `styles.css` | Split `variables.css` + `App.scss` + component/page `.scss` |
| Root element | `<div id="app">` | `<div id="root">` |
