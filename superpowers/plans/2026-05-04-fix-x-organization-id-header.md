# Fix Missing X-Organization-Id Header Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the frontend console send the `X-Organization-Id` header on all API requests that the NestJS backend's `OrgGuard` requires, eliminating 400 "Missing X-Organization-Id header" errors on the dashboard and other org-scoped pages.

**Architecture:** Add an `orgId` field to `ApiRequestOptions` (handled once in `requestApi`) so every API function can forward the current organization ID as a header without changing the HTTP client signature. Hooks read `organizationId` from the URL via the existing `useRouteOrganization()` hook and pass it into API functions. Server-side fetch functions also receive `orgId` as a parameter so SSR pages stay consistent.

**Tech Stack:** Next.js 15 App Router, TanStack Query, NestJS backend with `OrgGuard`, `useRouteOrganization()` hook, TypeScript.

---

## Background

The NestJS `OrgGuard` (`api/src/shared/guards/org.guard.ts:34`) throws `BadRequestException('Missing X-Organization-Id header')` whenever a route decorated with `@RequiresOrg()` is called without the header. The frontend `clientApiFetch` / `serverApiFetch` in `console/lib/api/http.ts` never sets this header, so every org-scoped endpoint (monitors, heartbeats, integrations, on-call, status-pages, etc.) fails with HTTP 400.

The `organizationId` is already in the URL (`/org/[organizationId]/...`) and the hook `useRouteOrganization()` at `console/hooks/use-route-organization.ts` reads it. It just needs to be threaded through to the API layer.

## Endpoints that require `X-Organization-Id` (backend `@RequiresOrg()`)

| Endpoint prefix | Controller |
|---|---|
| `GET/POST /api/v1/uptime` | OrganizationUptimeMonitorController |
| `GET /api/v1/monitors` | OrganizationMonitorController |
| `GET /api/v1/monitors/statistics` | OrganizationMonitorController |
| `GET/POST /api/v1/heartbeats` | HeartbeatController |
| `GET /api/v1/integrations/apps*` | IntegrationsCatalogController |
| `GET/POST/PATCH/DELETE /api/v1/integrations/installations*` | IntegrationsInstallationController |
| `GET/POST/PATCH/DELETE /api/v1/integrations/installations/:id/destination-configs*` | IntegrationsDestinationConfigController |
| `GET/POST/PATCH/DELETE /api/v1/on-call/teams*` | OnCallTeamController |
| `GET/POST/PATCH/DELETE /api/v1/on-call/policies*` | OnCallPolicyController |
| `GET/POST/PATCH/DELETE /api/v1/on-call/monitors/:id/policies*` | MonitorOnCallPolicyController |
| `GET/POST/PATCH/DELETE /api/v1/on-call/teams/:id/channels*` | OnCallChannelController |
| `GET /api/v1/organizations/enriched` | staging backend |
| `GET /api/v1/organizations/my-role` | staging backend |
| `GET /api/v1/manage/status-pages*` | StatusPageController (verify) |

## File Map

| File | Change |
|---|---|
| `console/lib/api/types.ts` | Add `orgId?: string` to `ApiRequestOptions` |
| `console/lib/api/http.ts` | Set `X-Organization-Id` header when `options.orgId` is present |
| `console/lib/api/monitors.api.ts` | Add `orgId` param to org-list/create/stats functions |
| `console/lib/api/heartbeats.api.ts` | Add `orgId` param to list/create functions |
| `console/lib/api/integrations.api.ts` | Add `orgId` param to all functions |
| `console/lib/api/on-call.api.ts` | Add `orgId` param to all functions |
| `console/lib/api/organizations.api.ts` | Add `orgId` param to `fetchOrganizationsEnriched`, `fetchUserRole` |
| `console/lib/api/status-pages.api.ts` | Add `orgId` param to org-scoped functions |
| `console/hooks/use-monitors.ts` | Read orgId via `useRouteOrganization()`, pass to API |
| `console/hooks/use-heartbeats.ts` | Same pattern |
| `console/hooks/use-integrations.ts` | Same pattern |
| `console/hooks/use-on-call.ts` | Same pattern |
| `console/hooks/use-organizations.ts` | Same pattern (enriched + my-role) |
| `console/hooks/use-status-pages.ts` | Same pattern |
| Server-side pages under `org/[organizationId]` | Pass `organizationId` from params to `serverApiFetch` wrappers |

---

## Task 1: Add `orgId` to ApiRequestOptions and HTTP client

**Files:**
- Modify: `console/lib/api/types.ts`
- Modify: `console/lib/api/http.ts`

- [ ] **Step 1: Add `orgId` field to `ApiRequestOptions`**

In `console/lib/api/types.ts`, add `orgId` to the interface:

```typescript
export interface ApiRequestOptions<TBody = unknown> {
  method?: ApiMethod;
  query?: ApiQueryParams;
  body?: TBody;
  authToken?: string | null;
  headers?: HeadersInit;
  forwardCookies?: boolean;
  orgId?: string;  // ← add this line
  init?: Omit<RequestInit, "body" | "headers" | "method">;
}
```

- [ ] **Step 2: Apply the header in `requestApi`**

In `console/lib/api/http.ts`, `requestApi` at line 69 destructures specific fields. Add `orgId` to that destructure block AND add the header-set call. The updated function signature section (lines 68–103):

```typescript
async function requestApi<TResponse, TBody>(
  path: string,
  options: ApiRequestOptions<TBody> = {},
): Promise<TResponse> {
  const {
    method = "GET",
    query,
    body,
    authToken,
    headers,
    forwardCookies,
    orgId,          // ← add to destructure
    init,
  } = options;

  const requestHeaders = new Headers(headers);
  const cookieHeader = await getForwardedCookieHeader(forwardCookies);

  requestHeaders.set("accept", "application/json");

  if (orgId) {
    requestHeaders.set("x-organization-id", orgId);   // ← reference destructured variable
  }

  if (authToken) {
    requestHeaders.set("authorization", `Bearer ${authToken}`);
  }
  // ... rest of the function unchanged
```

- [ ] **Step 3: Verify the type compiles**

```bash
cd /Users/washington/Projects/GetMonitor/console && npx tsc --noEmit --project tsconfig.json 2>&1 | head -20
```

Expected: no errors about `orgId`.

- [ ] **Step 4: Commit**

```bash
cd /Users/washington/Projects/GetMonitor/console
git add lib/api/types.ts lib/api/http.ts
git commit -m "feat: add orgId option to ApiRequestOptions to forward X-Organization-Id header"
```

---

## Task 2: Update `monitors.api.ts`

**Files:**
- Modify: `console/lib/api/monitors.api.ts`

Org-scoped endpoints: `GET /api/v1/uptime`, `POST /api/v1/uptime`, `GET /api/v1/monitors`, `GET /api/v1/monitors/statistics`, `GET /api/v1/monitors/organization/statistics`.

- [ ] **Step 1: Add `orgId` to org-list and statistics functions**

Replace the following functions in `console/lib/api/monitors.api.ts`:

```typescript
export async function serverFetchMonitorsByOrgWithMetrics(orgId: string): Promise<unknown[]> {
  return serverApiFetch("/api/v1/uptime", {
    forwardCookies: true,
    orgId,
    query: { include: "metrics" },
  });
}

export async function fetchMonitorsByOrg(orgId: string, withMetrics?: boolean): Promise<unknown[]> {
  return clientApiFetch("/api/v1/uptime", {
    orgId,
    query: withMetrics ? { include: "metrics" } : undefined,
  });
}

export async function fetchAllMonitorsByOrg(orgId: string): Promise<unknown[]> {
  return clientApiFetch("/api/v1/monitors", { orgId });
}

export async function serverFetchAllMonitorsByOrg(orgId: string): Promise<unknown[]> {
  return serverApiFetch("/api/v1/monitors", { forwardCookies: true, orgId });
}

export async function fetchOrgMonitorStatistics(orgId: string, timeRange?: "24h" | "7d" | "30d"): Promise<unknown> {
  return clientApiFetch("/api/v1/monitors/statistics", {
    orgId,
    query: timeRange ? { timeRange } : undefined,
  });
}

export async function serverFetchOrgMonitorStatistics(orgId: string, timeRange?: "24h" | "7d" | "30d"): Promise<unknown> {
  return serverApiFetch("/api/v1/monitors/statistics", {
    forwardCookies: true,
    orgId,
    query: timeRange ? { timeRange } : undefined,
  });
}

export async function createMonitor(orgId: string, data: unknown): Promise<unknown> {
  return clientApiFetch("/api/v1/uptime", { orgId, method: "POST", body: data });
}

export async function fetchOrganizationMonitorStatistics(orgId: string, timeRange?: "24h" | "7d" | "30d"): Promise<unknown> {
  return clientApiFetch("/api/v1/monitors/organization/statistics", {
    orgId,
    query: timeRange ? { timeRange } : undefined,
  });
}
```

Note: `fetchMonitorById`, `fetchMonitorStatistics`, `fetchUptimeLogs`, `updateMonitor`, `deleteMonitor` target individual monitor resources and likely don't use `@RequiresOrg()` — leave them unchanged.

Also fix `useOrganizationMonitorStatistics` query key collision (see Task 7 Step 1). Use a distinct key `["monitors", "organization", "statistics"]` for `fetchOrganizationMonitorStatistics` to avoid cache collision with `fetchOrgMonitorStatistics`.

- [ ] **Step 2: Type-check**

```bash
cd /Users/washington/Projects/GetMonitor/console && npx tsc --noEmit 2>&1 | grep monitors | head -20
```

Expected: TypeScript errors pointing to the hooks/callers of `fetchMonitorsByOrg` etc. (they must now pass `orgId` — that's fixed in Task 7).

- [ ] **Step 3: Commit**

```bash
git add lib/api/monitors.api.ts
git commit -m "feat: add orgId param to org-scoped monitor API functions"
```

---

## Task 3: Update `heartbeats.api.ts`

**Files:**
- Modify: `console/lib/api/heartbeats.api.ts`

Org-scoped: `GET /api/v1/heartbeats`, `POST /api/v1/heartbeats`.

- [ ] **Step 1: Add `orgId` to list and create functions**

```typescript
export async function serverFetchHeartbeats(orgId: string, params?: { includeInactive?: boolean }): Promise<unknown[]> {
  return serverApiFetch("/api/v1/heartbeats", {
    forwardCookies: true,
    orgId,
    query: params?.includeInactive !== undefined
      ? { includeInactive: params.includeInactive }
      : undefined,
  });
}

export async function fetchHeartbeats(orgId: string, params?: { includeInactive?: boolean }): Promise<unknown[]> {
  return clientApiFetch("/api/v1/heartbeats", {
    orgId,
    query: params?.includeInactive !== undefined
      ? { includeInactive: params.includeInactive }
      : undefined,
  });
}

export async function createHeartbeat(orgId: string, data: {
  name: string;
  description?: string;
  intervalSeconds: number;
  gracePeriodSeconds: number;
}): Promise<unknown> {
  return clientApiFetch("/api/v1/heartbeats", { orgId, method: "POST", body: data });
}
```

Leave `fetchHeartbeatById`, `serverFetchHeartbeatById`, `updateHeartbeat`, `deleteHeartbeat`, `regenerateHeartbeatToken`, `pauseHeartbeat`, `unpauseHeartbeat` unchanged (individual resource endpoints).

- [ ] **Step 2: Commit**

```bash
git add lib/api/heartbeats.api.ts
git commit -m "feat: add orgId param to org-scoped heartbeat API functions"
```

---

## Task 4: Update `integrations.api.ts`

**Files:**
- Modify: `console/lib/api/integrations.api.ts`

All integration endpoints require `@RequiresOrg()`. File has 91 lines — update EVERY function including the destination-config functions (lines 50-66) AND `getInstallationByApp` (lines 72-91).

- [ ] **Step 1: Add `orgId` to all integration functions**

Replace the entire file content with:

```typescript
import { clientApiFetch, serverApiFetch } from "@/lib/api/http";

// Catalog
export const fetchIntegrationApps = (orgId: string, appType?: "source" | "destination") =>
  clientApiFetch<unknown[]>("/api/v1/integrations/apps", {
    orgId,
    query: appType ? { appType } : undefined,
  });

export const serverFetchIntegrationApps = (orgId: string, appType?: "source" | "destination") =>
  serverApiFetch<unknown[]>("/api/v1/integrations/apps", {
    forwardCookies: true,
    orgId,
    query: appType ? { appType } : undefined,
  });

export const fetchIntegrationApp = (orgId: string, appId: string) =>
  clientApiFetch(`/api/v1/integrations/apps/${appId}`, { orgId });

export const serverFetchIntegrationApp = (orgId: string, appId: string) =>
  serverApiFetch(`/api/v1/integrations/apps/${appId}`, { forwardCookies: true, orgId });

export const fetchIntegrationAppCounts = (orgId: string) =>
  clientApiFetch("/api/v1/integrations/apps/counts", { orgId });

// Installations
export const fetchInstallations = (orgId: string, params?: { status?: string; appType?: string; appId?: string }) =>
  clientApiFetch<unknown[]>("/api/v1/integrations/installations", {
    orgId,
    query: params as Record<string, string | undefined>,
  });

export const fetchInstallation = (orgId: string, id: string) =>
  clientApiFetch(`/api/v1/integrations/installations/${id}`, { orgId });

export const fetchInstallationHealth = (orgId: string, id: string) =>
  clientApiFetch(`/api/v1/integrations/installations/${id}/health`, { orgId });

export const createInstallation = (orgId: string, data: { appId: string; connectionMetadata?: object }) =>
  clientApiFetch("/api/v1/integrations/installations", { orgId, method: "POST", body: data });

export const disableInstallation = (orgId: string, id: string) =>
  clientApiFetch(`/api/v1/integrations/installations/${id}/disable`, { orgId, method: "POST" });

export const reconnectInstallation = (orgId: string, id: string) =>
  clientApiFetch(`/api/v1/integrations/installations/${id}/reconnect`, { orgId, method: "POST" });

export const updateInstallationConfig = (orgId: string, id: string, connectionMetadata: object) =>
  clientApiFetch(`/api/v1/integrations/installations/${id}`, {
    orgId,
    method: "PATCH",
    body: { connectionMetadata },
  });

// Destination configs
export const fetchDestinationConfigs = (orgId: string, installationId: string) =>
  clientApiFetch<unknown[]>(
    `/api/v1/integrations/installations/${installationId}/destination-configs`,
    { orgId },
  );

export const createDestinationConfig = (
  orgId: string,
  installationId: string,
  data: { name: string; destinationType: string; config: object },
) =>
  clientApiFetch(`/api/v1/integrations/installations/${installationId}/destination-configs`, {
    orgId,
    method: "POST",
    body: data,
  });

export const updateDestinationConfig = (orgId: string, installationId: string, configId: string, data: unknown) =>
  clientApiFetch(
    `/api/v1/integrations/installations/${installationId}/destination-configs/${configId}`,
    { orgId, method: "PATCH", body: data },
  );

export const deleteDestinationConfig = (orgId: string, installationId: string, configId: string): Promise<void> =>
  clientApiFetch(
    `/api/v1/integrations/installations/${installationId}/destination-configs/${configId}`,
    { orgId, method: "DELETE" },
  );

export const previewTemplate = (
  orgId: string,
  installationId: string,
  data: { template: string; variables: object },
) =>
  clientApiFetch(
    `/api/v1/integrations/installations/${installationId}/destination-configs/preview`,
    { orgId, method: "POST", body: data },
  );

export const testDestinationConfig = (orgId: string, installationId: string, configId: string) =>
  clientApiFetch(
    `/api/v1/integrations/installations/${installationId}/destination-configs/${configId}/test`,
    { orgId, method: "POST" },
  );

export async function getInstallationByApp(
  organizationId: string,
  appId: string,
): Promise<unknown | null> {
  try {
    const installations = await serverApiFetch<unknown[]>(
      `/api/v1/integrations/installations`,
      {
        forwardCookies: true,
        orgId: organizationId,   // ← forward as header
        query: { appId },
      },
    );
    if (Array.isArray(installations) && installations.length > 0) {
      return installations[0];
    }
    return null;
  } catch {
    return null;
  }
}
```

- [ ] **Step 2: Update `use-integrations.ts` mutation functions that reference API by name**

`mutationFn: api.createInstallation` will now have a different arity — wrap it to inject `orgId`. Every hook in `use-integrations.ts` needs `const { organizationId } = useRouteOrganization();` AND explicit wrapping:

```typescript
// useCreateInstallation — was: mutationFn: api.createInstallation
export function useCreateInstallation() {
  const { organizationId } = useRouteOrganization();
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (data: { appId: string; connectionMetadata?: object }) =>
      api.createInstallation(organizationId!, data),
    onSuccess: () => qc.invalidateQueries({ queryKey: ["integrations", "installations"] }),
  });
}

// useInstallations — was: queryFn: () => api.fetchInstallations(params)
export function useInstallations(params?: { status?: string; appType?: string; appId?: string }) {
  const { organizationId } = useRouteOrganization();
  return useQuery({
    queryKey: qk.integrations.installations(params as Record<string, string>),
    queryFn: () => api.fetchInstallations(organizationId!, params),
    enabled: !!organizationId,
  });
}
```

Apply this pattern to ALL hooks in `use-integrations.ts` — every query and mutation function that calls an API function now takes `organizationId!` as the first arg. Also add `enabled: !!organizationId` to every `useQuery`.

- [ ] **Step 3: Commit**

```bash
git add lib/api/integrations.api.ts hooks/use-integrations.ts
git commit -m "feat: add orgId param to all integration API functions and hooks"
```

---

## Task 5: Update `on-call.api.ts`

**Files:**
- Modify: `console/lib/api/on-call.api.ts`

All on-call endpoints require `@RequiresOrg()`. The file has 99 lines including **alias functions** at lines 67-98 (`fetchOnCallChannelsForTeam`, `fetchMonitorOnCallPoliciesForOrg`, etc.) that duplicate the primary functions — those must also receive `orgId`.

Also note: `fetchOnCallGraph` (`GET /api/v1/on-call/graph` at line 63) — verify in the backend whether it has `@RequiresOrg()` before adding `orgId`. If uncertain, add it anyway to be safe.

- [ ] **Step 1: Add `orgId` as first parameter to every exported function**

Pattern for primary functions:

```typescript
// Before:
export const fetchOnCallTeams = () => clientApiFetch<unknown[]>("/api/v1/on-call/teams");

// After:
export const fetchOnCallTeams = (orgId: string) =>
  clientApiFetch<unknown[]>("/api/v1/on-call/teams", { orgId });
```

```typescript
// Before (function with body data):
export const createOnCallTeam = (data: { name: string; description?: string }) =>
  clientApiFetch("/api/v1/on-call/teams", { method: "POST", body: data });

// After:
export const createOnCallTeam = (orgId: string, data: { name: string; description?: string }) =>
  clientApiFetch("/api/v1/on-call/teams", { orgId, method: "POST", body: data });
```

Apply this pattern to ALL functions including alias functions (lines 67-98):

```typescript
// Alias functions — same treatment:
export const fetchOnCallChannelsForTeam = (orgId: string, teamId: string): Promise<unknown[]> =>
  clientApiFetch<unknown[]>(`/api/v1/on-call/teams/${teamId}/channels`, { orgId });

export const fetchMonitorOnCallPoliciesForOrg = (orgId: string, monitorId: string): Promise<unknown> =>
  clientApiFetch(`/api/v1/on-call/monitors/${monitorId}/policies`, { orgId });

export const assignOnCallPolicyToMonitor = (orgId: string, monitorId: string, policyId: string): Promise<unknown> =>
  clientApiFetch(`/api/v1/on-call/monitors/${monitorId}/policies`, {
    orgId,
    method: "POST",
    body: { policyId },
  });

// Apply the same pattern to: addOnCallChannelToTeam, removeOnCallChannelFromTeam,
// toggleOnCallChannelInTeam, unassignOnCallPolicyFromMonitor, toggleOnCallPolicyForMonitor,
// and fetchOnCallGraph (add orgId here too)
```

- [ ] **Step 2: Commit**

```bash
git add lib/api/on-call.api.ts
git commit -m "feat: add orgId param to all on-call API functions including alias variants"
```

---

## Task 6: Update `organizations.api.ts` and `status-pages.api.ts`

**Files:**
- Modify: `console/lib/api/organizations.api.ts`
- Modify: `console/lib/api/status-pages.api.ts`

- [ ] **Step 1: Add `orgId` to org-scoped organization functions**

In `console/lib/api/organizations.api.ts`:

```typescript
export async function fetchOrganizationsEnriched(orgId: string): Promise<unknown[]> {
  return clientApiFetch("/api/v1/organizations/enriched", { orgId });
}

export async function serverFetchOrganizationsEnriched(orgId: string): Promise<unknown[]> {
  return serverApiFetch("/api/v1/organizations/enriched", { forwardCookies: true, orgId });
}

export async function fetchUserRole(orgId: string): Promise<unknown> {
  return clientApiFetch("/api/v1/organizations/my-role", { orgId });
}
```

Leave `fetchOrganizations`, `fetchGuestAccessOrganizations`, `fetchOrganizationStatus(orgId)`, `updateOrganization`, etc. unchanged — they either don't use `@RequiresOrg()` or already have the `orgId` in the URL path (not as a header).

- [ ] **Step 2: Add `orgId` to status-pages list and create functions**

In `console/lib/api/status-pages.api.ts`, the `GET` and `POST /api/v1/manage/status-pages` endpoints are org-scoped. `createStatusPage` also targets this path and needs the header:

```typescript
export async function fetchStatusPages(orgId: string): Promise<unknown[]> {
  return clientApiFetch("/api/v1/manage/status-pages", { orgId });
}

export async function serverFetchStatusPages(orgId: string): Promise<unknown[]> {
  return serverApiFetch("/api/v1/manage/status-pages", { forwardCookies: true, orgId });
}

export async function createStatusPage(orgId: string, data: {
  name: string;
  visibility: "public" | "private";
  domain: string;
  description?: string;
}): Promise<unknown> {
  return clientApiFetch("/api/v1/manage/status-pages", { orgId, method: "POST", body: data });
}
```

Leave `fetchStatusPageById`, `serverFetchStatusPageById`, `updateStatusPage`, `deleteStatusPage` unchanged (they reference individual resources by ID, not org-scoped list operations, or use a different controller path).

- [ ] **Step 3: Commit**

```bash
git add lib/api/organizations.api.ts lib/api/status-pages.api.ts
git commit -m "feat: add orgId param to org-scoped organization and status-page API functions"
```

---

## Task 7: Update client-side hooks

All hooks inside `console/hooks/` that call org-scoped API functions need to read `organizationId` from the route and pass it.

**Pattern** (apply to every hook below):

```typescript
import { useRouteOrganization } from "@/hooks/use-route-organization";

export function useMonitorsByOrg(withMetrics?: boolean) {
  const { organizationId } = useRouteOrganization();
  return useQuery({
    queryKey: withMetrics ? qk.monitors.byOrgWithMetrics() : qk.monitors.byOrg(),
    queryFn: () => fetchMonitorsByOrg(organizationId!, withMetrics),
    enabled: !!organizationId,
  });
}
```

**Files:**
- Modify: `console/hooks/use-monitors.ts`
- Modify: `console/hooks/use-heartbeats.ts`
- Modify: `console/hooks/use-integrations.ts`
- Modify: `console/hooks/use-on-call.ts`
- Modify: `console/hooks/use-organizations.ts`
- Modify: `console/hooks/use-status-pages.ts`

- [ ] **Step 1: Update `use-monitors.ts`**

```typescript
"use client";

import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { useRouteOrganization } from "@/hooks/use-route-organization";
import {
  fetchMonitorsByOrg,
  fetchAllMonitorsByOrg,
  fetchMonitorById,
  fetchMonitorsByStatusPage,
  fetchMonitorStatistics,
  fetchOrgMonitorStatistics,
  fetchStatusPageMonitorStatistics,
  fetchUptimeLogs,
  createMonitor,
  updateMonitor,
  deleteMonitor,
  fetchOrganizationMonitorStatistics,
} from "@/lib/api/monitors.api";
import { qk } from "@/lib/api/query-keys";

export function useMonitorsByOrg(withMetrics?: boolean) {
  const { organizationId } = useRouteOrganization();
  return useQuery({
    queryKey: withMetrics ? qk.monitors.byOrgWithMetrics() : qk.monitors.byOrg(),
    queryFn: () => fetchMonitorsByOrg(organizationId!, withMetrics),
    enabled: !!organizationId,
  });
}

export function useAllMonitorsByOrg() {
  const { organizationId } = useRouteOrganization();
  return useQuery({
    queryKey: qk.monitors.allByOrg(),
    queryFn: () => fetchAllMonitorsByOrg(organizationId!),
    enabled: !!organizationId,
  });
}

export function useOrgMonitorStatistics(timeRange?: "24h" | "7d" | "30d") {
  const { organizationId } = useRouteOrganization();
  return useQuery({
    queryKey: [...qk.monitors.orgStatistics(), timeRange],
    queryFn: () => fetchOrgMonitorStatistics(organizationId!, timeRange),
    enabled: !!organizationId,
  });
}

export function useOrganizationMonitorStatistics(timeRange?: "24h" | "7d" | "30d") {
  const { organizationId } = useRouteOrganization();
  return useQuery({
    // IMPORTANT: use a DISTINCT key from useOrgMonitorStatistics above to avoid cache collision.
    // fetchOrganizationMonitorStatistics calls /api/v1/monitors/organization/statistics
    // while fetchOrgMonitorStatistics calls /api/v1/monitors/statistics — different endpoints.
    queryKey: ["monitors", "organization", "statistics", timeRange],
    queryFn: () => fetchOrganizationMonitorStatistics(organizationId!, timeRange),
    enabled: !!organizationId,
  });
}

export function useCreateMonitor() {
  const { organizationId } = useRouteOrganization();
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (data: unknown) => createMonitor(organizationId!, data),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: qk.monitors.byOrg() });
      qc.invalidateQueries({ queryKey: qk.monitors.byOrgWithMetrics() });
    },
  });
}

// Hooks without org header unchanged: useMonitorById, useMonitorsByStatusPage,
// useMonitorStatistics, useUptimeLogs, useUpdateMonitor, useDeleteMonitor
```

- [ ] **Step 2: Update `use-heartbeats.ts`**

Add `const { organizationId } = useRouteOrganization();` to `useHeartbeats` (list hook) and any create hook. Pass `organizationId!` as the first argument to `fetchHeartbeats`, `createHeartbeat`. Add `enabled: !!organizationId`.

- [ ] **Step 3: Update `use-integrations.ts`**

Add `useRouteOrganization()` to every hook that calls an org-scoped integration function. Pass `organizationId!` as the first argument. Add `enabled: !!organizationId` to queries.

- [ ] **Step 4: Update `use-on-call.ts`**

Same pattern. Every hook and mutation reads `organizationId` from `useRouteOrganization()` and passes it.

- [ ] **Step 5: Update `use-organizations.ts`**

**Note on scope:** `fetchOrganizationsEnriched` and `fetchUserRole` are called from within org dashboard pages where `organizationId` is always present. Adding `enabled: !!organizationId` prevents firing on non-org routes (org selector page, onboarding), but those pages don't currently call these hooks — verify before adding the guard, and confirm with the backend that these staging endpoints truly require `@RequiresOrg()`.

For `useOrganizationsEnriched` and `useUserRole`:

```typescript
export function useOrganizationsEnriched() {
  const { organizationId } = useRouteOrganization();
  return useQuery({
    queryKey: ["organizations", "enriched"],   // raw literal — qk.organizations.enriched() added in Task 10
    queryFn: () => fetchOrganizationsEnriched(organizationId!),
    enabled: !!organizationId,   // only fires inside org routes; verify this doesn't break callers
  });
}

export function useUserRole() {
  const { organizationId } = useRouteOrganization();
  return useQuery({
    queryKey: ["organizations", "my-role"],
    queryFn: () => fetchUserRole(organizationId!),
    enabled: !!organizationId,
  });
}
```

- [ ] **Step 6: Update `use-status-pages.ts`**

Add `useRouteOrganization()` to `useStatusPages` (list hook), pass `organizationId!` to `fetchStatusPages`.

- [ ] **Step 7: Type-check the full console project**

```bash
cd /Users/washington/Projects/GetMonitor/console && npx tsc --noEmit 2>&1 | head -40
```

Expected: no new errors.

- [ ] **Step 8: Commit**

```bash
cd /Users/washington/Projects/GetMonitor/console
git add hooks/use-monitors.ts hooks/use-heartbeats.ts hooks/use-integrations.ts hooks/use-on-call.ts hooks/use-organizations.ts hooks/use-status-pages.ts
git commit -m "feat: thread organizationId from route into all org-scoped hooks"
```

---

## Task 8: Fix server-side fetches in Server Components

Server-side API functions (e.g. `serverFetchMonitorsByOrgWithMetrics`, `serverFetchStatusPages`) are called from server components that already have access to `params.organizationId`. They need to pass it through.

**Files:**
- Modify: Any page/layout under `console/app/[lang]/(auth)/org/[organizationId]/` that calls a `serverFetch*` function.

- [ ] **Step 1: Find all server-side API calls**

```bash
grep -r "serverFetch" /Users/washington/Projects/GetMonitor/console/app --include="*.tsx" --include="*.ts" -n
```

- [ ] **Step 2: Update each usage**

For each file found, extract `organizationId` from params and pass it.

**Known call site** — `console/app/[lang]/(auth)/org/[organizationId]/monitors/page.tsx` line 40 already destructures `organizationId`. Update the `Promise.all` call there (both `serverFetchMonitorsByOrgWithMetrics` AND `serverFetchHeartbeats` are in the same call):

```typescript
// Before:
const [monitors, heartbeats] = await Promise.all([
  serverFetchMonitorsByOrgWithMetrics(),
  serverFetchHeartbeats({ includeInactive: true }),
]);

// After:
const { organizationId } = await params;
const [monitors, heartbeats] = await Promise.all([
  serverFetchMonitorsByOrgWithMetrics(organizationId),
  serverFetchHeartbeats(organizationId, { includeInactive: true }),
]);
```

Apply the same `organizationId` injection pattern to all other server-side call sites found by the grep in Step 1.

- [ ] **Step 3: Type-check**

```bash
cd /Users/washington/Projects/GetMonitor/console && npx tsc --noEmit 2>&1 | head -20
```

Expected: clean output.

- [ ] **Step 4: Commit**

```bash
git add app/
git commit -m "feat: pass organizationId to server-side API fetches in org pages"
```

---

## Task 9: Smoke test in browser

- [ ] **Step 1: Run the dev server**

```bash
cd /Users/washington/Projects/GetMonitor/console && npm run dev
```

- [ ] **Step 2: Open the dashboard**

Navigate to the org dashboard page (e.g. `/en-US/org/<orgId>/dashboard`).

- [ ] **Step 3: Verify no 400 errors**

Open DevTools → Network → Filter XHR. Confirm:
- `enriched` returns 200
- `monitors` returns 200
- `statistics` returns 200
- `status-pages` returns 200
- `my-role` returns 200
- No requests show "Missing X-Organization-Id header"

- [ ] **Step 4: Verify sidebar, monitors, heartbeats, integrations, on-call pages load correctly**

Navigate to each section and confirm data loads.

---

## Task 10: Query key hygiene (if missing keys needed)

If `qk.organizations.enriched()` or `qk.organizations.myRole()` are missing from `console/lib/api/query-keys.ts`, add them.

- [ ] **Step 1: Check query-keys.ts**

```bash
grep -n "enriched\|myRole\|my-role" /Users/washington/Projects/GetMonitor/console/lib/api/query-keys.ts
```

- [ ] **Step 2: Add missing keys if any**

Example addition:
```typescript
enriched: () => ["organizations", "enriched"] as const,
myRole: () => ["organizations", "my-role"] as const,
```

- [ ] **Step 3: Commit (if changes made)**

```bash
git add lib/api/query-keys.ts
git commit -m "feat: add enriched and myRole query keys for organizations"
```
