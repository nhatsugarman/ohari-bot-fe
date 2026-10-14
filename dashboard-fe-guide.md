# Hướng dẫn tạo project FE dashboard

Dashboard xem số liệu quảng cáo, gọi API của backend này (`/api/auth/*`, `/api/ads/*`).
Backend đã xong; tài liệu này hướng dẫn dựng phần giao diện.

## 1. Kiến trúc

```
receive-message/                 (repo này)
├── src/                         backend: bot, job sync, MCP, API
└── dashboard/                   FE — project RIÊNG, build + deploy độc lập
```

- FE nằm trong repo này nhưng có `package.json` riêng. Sửa giao diện không đụng bot.
- FE và API **cùng domain** (`bot.ohari.jp/dashboard` và `bot.ohari.jp/api`), nên đăng nhập
  dùng cookie là đủ, không cần CORS, không cần token trong JS.
- Số trên dashboard lấy từ cùng service với MCP, nên khớp với những gì Claude trả lời.

## 2. Tạo project

```bash
cd receive-message
npm create vite@latest dashboard -- --template react-ts
cd dashboard
npm install
npm install @tanstack/react-query react-router-dom recharts
npm install -D tailwindcss @tailwindcss/vite
```

| Thư viện | Dùng để |
|---|---|
| Vite + React + TypeScript | Khung, build ra file tĩnh |
| `@tanstack/react-query` | Gọi API, cache, tự xử lý loading/lỗi |
| `react-router-dom` | Trang đăng nhập / dashboard |
| `recharts` | Biểu đồ |
| Tailwind | CSS |

Bật Tailwind: trong `src/index.css` chỉ để một dòng `@import "tailwindcss";`.

## 3. Cấu hình Vite

`dashboard/vite.config.ts`:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
  // Production phục vụ ở bot.ohari.jp/dashboard/
  base: "/dashboard/",
  server: {
    // Dev: chuyển /api sang backend local để cookie vẫn "cùng domain"
    proxy: { "/api": "http://localhost:3004" },
  },
});
```

Nhờ proxy, khi dev FE chạy ở `localhost:5173` vẫn gọi `/api/...` như production.

## 4. Cấu trúc thư mục

Chia theo tính năng, giống backend:

```
dashboard/src/
├── main.tsx                     khởi tạo React, QueryClient, Router
├── App.tsx                      khai báo route
├── lib/
│   ├── api.ts                   hàm gọi API chung (fetch + xử lý lỗi)
│   └── format.ts                định dạng tiền, %, ngày
├── features/
│   ├── auth/
│   │   ├── auth.api.ts          login, logout, me
│   │   ├── useAuth.ts           hook: người đang đăng nhập
│   │   ├── LoginPage.tsx
│   │   └── RequireAuth.tsx      chặn trang khi chưa đăng nhập
│   └── ads/
│       ├── ads.types.ts         kiểu dữ liệu API trả về
│       ├── ads.api.ts           các hàm gọi /api/ads/*
│       ├── ads.hooks.ts         useLeadCost, usePerformance...
│       ├── DashboardPage.tsx
│       └── components/
│           ├── KpiCards.tsx
│           ├── DailyChart.tsx
│           ├── TopAdsTable.tsx
│           └── SyncStatus.tsx
└── components/                  nút, ô, bảng dùng chung
```

## 5. API backend

Mọi request trả JSON. Lỗi luôn có dạng `{ error: "MÃ_LỖI", message }` hoặc, khi input sai,
`{ error: "VALIDATION", issues: [{ path, message }] }`.

### Đăng nhập

| Method | URL | Body | Trả về |
|---|---|---|---|
| POST | `/api/auth/login` | `{ email, password }` | `{ user }` + đặt cookie `ohari_sid` |
| POST | `/api/auth/logout` | — | 204, xoá cookie |
| GET | `/api/auth/me` | — | `{ user }`, hoặc 401 nếu chưa đăng nhập |

`user = { id, email, name, role: "admin" | "viewer" }`

Mã lỗi: `401 UNAUTHORIZED` (sai mật khẩu / chưa đăng nhập), `429 TOO_MANY_ATTEMPTS`
(sai 5 lần trong 15 phút).

### Số liệu quảng cáo (cần đăng nhập)

| URL | Tham số | Trả về |
|---|---|---|
| `/api/ads/sync-status` | — | `{ accounts: [...] }` tài khoản + lần sync gần nhất |
| `/api/ads/campaigns` | `account_id?`, `statuses?` (`ACTIVE,PAUSED`) | `{ campaigns: [...] }` |
| `/api/ads/performance` | `level`, `from`, `to`, `ids?`, `by_date?` | `{ range, rows: Row[] }` |
| `/api/ads/compare` | `level`, `from`, `to`, `ids?` | `{ range, previousRange, rows: [{ id, name, current, previous, changePct }] }` |
| `/api/ads/lead-cost` | `level?` (mặc định `ad`), `from`, `to`, `ids?`, `min_spend?`, `limit?` | `{ range, rows: Row[] }` xếp giá data rẻ nhất trước |

- `level`: `account` \| `campaign` \| `adset` \| `ad`
- `from`, `to`: `YYYY-MM-DD`, giờ Việt Nam, tính cả hai đầu, tối đa 400 ngày
- `ids`: nhiều id cách nhau bằng dấu phẩy
- `by_date=true`: tách từng ngày (mỗi dòng có thêm `date`)

**Một dòng `Row`** (tiền đã làm tròn theo đơn vị, tỉ lệ đã đổi ra %):

```ts
{
  id: string; name: string | null; date?: string;
  currency: string | null;          // null = chưa có số chi tiêu cho dòng này
  spend: number; impressions: number; clicks: number;
  messagingStarted: number;         // "mess"
  leads: number;                    // khách THẬT để lại SĐT
  ctrPct: number | null; cpm: number | null; cpc: number | null;
  costPerMessage: number | null;    // giá mess
  costPerLead: number | null;       // giá data — chỉ số chính
  messageToLeadPct: number | null;
  costPerPurchase: number | null; roas: number | null;
}
```

**Quy ước `null`**: không đủ dữ liệu để tính (ví dụ 0 SĐT), **không phải 0**. Hiển thị là `—`.
**Không cộng tiền khác `currency`** (có account tiêu bằng PHP).

## 6. Code mẫu

### `lib/api.ts`: một chỗ gọi API

```ts
export class ApiError extends Error {
  constructor(public status: number, public code: string, message: string) {
    super(message);
  }
}

export async function api<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`/api${path}`, {
    ...init,
    headers: { "content-type": "application/json", ...init?.headers },
    // cùng domain nên trình duyệt tự gửi cookie; không lưu token trong JS
    credentials: "same-origin",
  });
  if (res.status === 204) return undefined as T;
  const body = await res.json().catch(() => ({}));
  if (!res.ok) throw new ApiError(res.status, body.error ?? "UNKNOWN", body.message ?? "Có lỗi xảy ra");
  return body as T;
}

/** { from: "2026-10-01", ids: ["a","b"] } → "?from=2026-10-01&ids=a,b" (bỏ giá trị rỗng) */
export function toQuery(params: Record<string, string | number | boolean | string[] | undefined>) {
  const q = new URLSearchParams();
  for (const [k, v] of Object.entries(params)) {
    if (v === undefined || (Array.isArray(v) && v.length === 0)) continue;
    q.set(k, Array.isArray(v) ? v.join(",") : String(v));
  }
  const s = q.toString();
  return s ? `?${s}` : "";
}
```

### `features/auth/useAuth.ts`

```ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { api, ApiError } from "../../lib/api";

export interface User { id: string; email: string; name: string | null; role: "admin" | "viewer" }

export function useCurrentUser() {
  return useQuery({
    queryKey: ["me"],
    queryFn: () => api<{ user: User }>("/auth/me").then((r) => r.user),
    retry: (count, err) => !(err instanceof ApiError && err.status === 401) && count < 2,
  });
}

export function useLogin() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (input: { email: string; password: string }) =>
      api<{ user: User }>("/auth/login", { method: "POST", body: JSON.stringify(input) }),
    onSuccess: (r) => qc.setQueryData(["me"], r.user),
  });
}

export function useLogout() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: () => api<void>("/auth/logout", { method: "POST" }),
    onSuccess: () => qc.clear(),
  });
}
```

### `features/auth/RequireAuth.tsx`

```tsx
import { Navigate, Outlet } from "react-router-dom";
import { useCurrentUser } from "./useAuth";

export function RequireAuth() {
  const { data: user, isLoading } = useCurrentUser();
  if (isLoading) return null;
  return user ? <Outlet /> : <Navigate to="/login" replace />;
}
```

### `App.tsx` và `main.tsx`

```tsx
// App.tsx
import { Route, Routes } from "react-router-dom";
import { RequireAuth } from "./features/auth/RequireAuth";
import { LoginPage } from "./features/auth/LoginPage";
import { DashboardPage } from "./features/ads/DashboardPage";

export default function App() {
  return (
    <Routes>
      <Route path="/login" element={<LoginPage />} />
      <Route element={<RequireAuth />}>
        <Route path="/" element={<DashboardPage />} />
      </Route>
    </Routes>
  );
}

// main.tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { createRoot } from "react-dom/client";
import { BrowserRouter } from "react-router-dom";
import App from "./App";
import "./index.css";

const queryClient = new QueryClient();

createRoot(document.getElementById("root")!).render(
  <QueryClientProvider client={queryClient}>
    {/* basename khớp với base: "/dashboard/" trong vite.config.ts */}
    <BrowserRouter basename="/dashboard">
      <App />
    </BrowserRouter>
  </QueryClientProvider>,
);
```

### `features/ads/ads.hooks.ts`

```ts
import { useQuery } from "@tanstack/react-query";
import { api, toQuery } from "../../lib/api";
import type { Row } from "./ads.types";

type Level = "account" | "campaign" | "adset" | "ad";

export function useLeadCost(p: { from: string; to: string; level?: Level; limit?: number }) {
  return useQuery({
    queryKey: ["lead-cost", p],
    queryFn: () => api<{ rows: Row[] }>(`/ads/lead-cost${toQuery(p)}`).then((r) => r.rows),
  });
}

export function usePerformance(p: { from: string; to: string; level: Level; byDate?: boolean }) {
  return useQuery({
    queryKey: ["performance", p],
    queryFn: () =>
      api<{ rows: Row[] }>(`/ads/performance${toQuery({ ...p, by_date: p.byDate, byDate: undefined })}`).then((r) => r.rows),
  });
}
```

### `lib/format.ts`

```ts
const vnd = new Intl.NumberFormat("vi-VN");
export const money = (v: number | null, currency: string | null) =>
  v === null ? "—" : currency === "VND" || currency === null ? `${vnd.format(v)}đ` : `${v.toFixed(2)} ${currency}`;
export const percent = (v: number | null) => (v === null ? "—" : `${v.toLocaleString("vi-VN")}%`);
export const count = (v: number) => vnd.format(v);
```

## 7. Gợi ý trang đầu tiên

Dựng lại đúng bố cục bản dashboard mẫu đã xem:

1. **Ô chỉ số 30 ngày**: khách, SĐT, tỉ lệ mess → SĐT, chi tiêu, giá data. Gọi
   `/api/ads/compare?level=account&from=…&to=…`, cộng các account **cùng currency**.
2. **Biểu đồ theo ngày**: `/api/ads/performance?level=account&by_date=true&…`, cộng theo ngày.
3. **Bảng top ad**: `/api/ads/lead-cost?level=ad&limit=20&…`.
4. **Trạng thái sync**: `/api/ads/sync-status`; lần sync gần nhất lỗi thì hiện cảnh báo
   "số hôm nay có thể chưa đầy đủ".

Lưu ý: ở cấp `account`/`campaign`, khách đến từ ad **chưa sync** chưa được đếm (chưa biết thuộc
nhánh nào); cấp `ad` thì đủ. Khi sync chạy đều, chênh lệch này gần như không còn.

## 8. Chạy local

```bash
# Terminal 1 — backend (cổng 3004)
npm run dev

# Terminal 2 — FE
cd dashboard && npm run dev      # mở http://localhost:5173/dashboard/
```

Tạo tài khoản đăng nhập:

```bash
npx tsx scripts/auth-user.ts create ban@ohari.vn admin "Tên bạn"   # in mật khẩu ngẫu nhiên
PASSWORD='mat-khau-tu-chon' npx tsx scripts/auth-user.ts create ban@ohari.vn admin
npx tsx scripts/auth-user.ts reset-password ban@ohari.vn
```

## 9. Deploy lên VPS

```bash
cd dashboard && npm ci && npm run build    # ra dashboard/dist/
```

Thêm vào `nginx/default.ssl.conf`, trong khối `server` của `bot.ohari.jp`, và mount thư mục
`dashboard/dist` vào container nginx (`docker-compose.yml`, mục `volumes` của `nginx`:
`- ./dashboard/dist:/usr/share/nginx/dashboard:ro`):

```nginx
location /dashboard/ {
    alias /usr/share/nginx/dashboard/;
    try_files $uri $uri/ /dashboard/index.html;   # React Router: mọi đường dẫn về index.html
}
```

`/api/*` đã đi qua `location /` tới backend sẵn, không cần thêm.

## 10. Bảo mật: những điều FE KHÔNG làm

- **Không lưu token** trong `localStorage` hay biến JS: cookie `HttpOnly` do trình duyệt giữ.
- **Không tự kiểm tra quyền** để thay backend: ẩn nút với `viewer` cho gọn giao diện,
  nhưng backend mới là chỗ chặn thật.
- **Không dùng `dangerouslySetInnerHTML`** với tên campaign/ad: dữ liệu từ Facebook là text
  người khác đặt.
