# Hiro Frontend - Web Application Architecture

## Technology Stack

- **Framework**: React 18.2 + TypeScript
- **Build Tool**: Vite 4.3
- **State Management**: Redux Toolkit + Redux Persist
- **Routing**: React Router DOM v6
- **Styling**: Tailwind CSS 3.3
- **API Client**: Axios
- **Forms**: Formik + Yup validation

---

## Application Structure

```
client/src/
├── App.tsx                 # Root component
├── main.tsx               # Entry point
├── services/              # API integration layer
│   ├── AuthService.ts     # Authentication APIs
│   ├── ApplicantService.ts
│   ├── ApplicationService.ts
│   ├── JobServices.ts
│   └── BaseService.ts     # Axios instance with interceptors
├── store/                 # Redux state management
│   ├── index.ts          # Configure store + persist
│   └── slices/           # Feature-based state slices
├── views/                # Page components
│   ├── auth/             # Login, Signup
│   ├── crm/             # Dashboard, Applicants
│   └── ...
├── components/           # Reusable UI components
│   ├── layouts/         # Page layouts
│   └── template/        # Base UI components
└── configs/             # App configuration
    └── app.config.ts    # Base URLs, API prefix
```

---

## State Management (Redux)

### Store Configuration

**File**: `client/src/store/index.ts`

```typescript
import { persistStore, persistReducer } from 'redux-persist'
import storage from 'redux-persist/lib/storage' // localStorage

const store = configureStore({
    reducer: rootReducer,  // Combined reducers
    middleware: ...
})

export const persistor = persistStore(store)
```

**Persistence**: Saves Redux state to `localStorage`, survives page refreshes

**Key Slices**:
- `auth`: User authentication state (logged in user, token)
- `theme`: UI theme preferences
- Various feature-specific slices for CRM, jobs, etc.

---

## API Integration Layer

### Base Service Configuration

**File**: `client/src/services/BaseService.ts`

```typescript
const BACKEND_BASE_URL = 'http://127.0.0.1:8000'

const BaseService = axios.create({
    timeout: 60000,
    baseURL: BACKEND_BASE_URL + '/api',  // http://127.0.0.1:8000/api
    withCredentials: true,  // Sends session cookies
    headers: {
        'Content-Type': 'application/json',
    },
})
```

**Response Interceptor**:
```typescript
BaseService.interceptors.response.use(
    (response) => response,
    (error) => {
        if (response?.status === 401 || response?.status === 403) {
            store.dispatch(signOutSuccess())  // Auto-logout on auth failure
        }
        return Promise.reject(error)
    }
)
```

---

### Authentication Service

**File**: `client/src/services/AuthService.ts`

**Base URL**: `http://127.0.0.1:8000` (direct, not `/api` prefix)

```typescript
const authClient = axios.create({
    baseURL: 'http://127.0.0.1:8000',
    withCredentials: true,  // Required for session cookies
})

export async function apiSignIn(data: SignInCredential) {
    return authClient.post<SignInResponse>('/users/login/', {
        username: data.userName,
        password: data.password,
    })
}

export async function apiSignUp(data: SignUpCredential) {
    return authClient.post<SignUpResponse>('/users/register/', {
        username: data.userName,
        email: data.email,
        password: data.password,
        password_confirm: data.password,
    })
}

export async function apiGetCurrentUser() {
    return authClient.get<SignInResponse>('/users/me/')
}

export async function apiSignOut() {
    return authClient.post('/users/logout/')
}
```

**Integration Flow**:
1. Component dispatches action (e.g., `signIn(credentials)`)
2. Redux thunk calls `AuthService.apiSignIn()`
3. Response sets `sessionid` cookie automatically
4. Redux updates auth state with user data
5. Subsequent requests include cookie via `withCredentials: true`

---

### Applicant Service

**File**: `client/src/services/ApplicantService.ts`

**Base URL**: `http://127.0.0.1:8000/applicants/`

```typescript
export async function listApplicants(jobId?: string | number) {
    const url = jobId ? `${BASE_URL}?job=${jobId}` : BASE_URL
    const res = await fetch(url)
    if (!res.ok) throw new Error('Failed to fetch applicants')
    return res.json()
}

export async function createApplicant(applicantData: {...}) {
    const formData = new FormData()
    formData.append('name', applicantData.name)
    formData.append('email', applicantData.email)
    // ... add other fields
    
    const res = await fetch(BASE_URL, {
        method: 'POST',
        body: formData,
    })
    return res.json()
}
```

**Note**: Uses native `fetch` API instead of axios

---

### Application Service

**File**: `client/src/services/ApplicationService.ts`

**Base URL**: `http://127.0.0.1:8000/api/applications/`

```typescript
export async function createApplication(applicationData: {
    name: string
    email: string
    job: number
    resume_file: File
    score?: string
    status?: string
}) {
    const formData = new FormData()
    formData.append('name', applicationData.name)
    formData.append('email', applicationData.email)
    formData.append('job', applicationData.job.toString())
    formData.append('resume_file', applicationData.resume_file)
    
    const res = await fetch(BASE_URL, {
        method: 'POST',
        body: formData,
    })
    return res.json()
}

export async function updateApplication(id: string | number, applicationData: {...}) {
    const isStatusOnly = /* checks if only status field is being updated */
    const method = isStatusOnly ? 'PATCH' : 'PUT'
    
    const res = await fetch(`${BASE_URL}${id}/`, {
        method: method,
        body: formData,
    })
    return res.json()
}
```

**Key Logic**: Uses `PATCH` for partial updates (status only), `PUT` for full updates

---

## Routing & Page Structure

### Main App Component

**File**: `client/src/App.tsx`

```typescript
function App() {
    return (
        <Provider store={store}>
            <PersistGate loading={null} persistor={persistor}>
                <BrowserRouter>
                    <Theme>
                        <Layout />  {/* Renders routes */}
                    </Theme>
                </BrowserRouter>
            </PersistGate>
        </Provider>
    )
}
```

**Hierarchy**:
1. Redux Provider (state management)
2. PersistGate (waits for persisted state to load)
3. BrowserRouter (routing)
4. Theme (theme context provider)
5. Layout (renders route-based components)

---

## Key Features & Data Flow

### User Login Flow

1. **Component**: Login page renders Formik form
2. **Submission**: Form calls `dispatch(signIn({ userName, password }))`
3. **Redux Thunk**: Async action calls `AuthService.apiSignIn()`
4. **Backend**: Returns user data + sets `sessionid` cookie
5. **State Update**: Redux stores user in `auth.user`
6. **Redirect**: Router navigates to dashboard
7. **Persistence**: State saved to localStorage via redux-persist

### Application Submission Flow

1. **Component**: Application form with file upload
2. **Submission**: Calls `ApplicationService.createApplication(formData)`
3. **FormData Construction**:
   ```typescript
   const formData = new FormData()
   formData.append('name', name)
   formData.append('email', email)
   formData.append('job', jobId)
   formData.append('resume_file', fileInput.files[0])
   ```
4. **Backend**: Creates Applicant + Application + stores resume as BLOB
5. **Response**: Returns application with `resume_url`
6. **UI Update**: Refreshes applicant list

### Viewing Applicant Profile

1. **Component**: Applicant detail page
2. **API Call**: `GET /api/applicants/{id}/profile/`
3. **Backend Logic**:
   - Checks if profile exists
   - If not, triggers MLOps extraction from resume
   - Returns structured profile data
4. **Rendering**: Displays skills, experience, education in structured format
5. **Social Links**: Shows LinkedIn/GitHub if extracted

