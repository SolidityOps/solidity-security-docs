# Dashboard API Integration

**Last Updated**: 2025-10-06
**Status**: ✅ Production Ready
**Version**: 1.0.0

## Overview

The Solidity Security Dashboard is fully integrated with the FastAPI backend, providing real-time security analysis data through a modern React-based interface.

## Architecture

### Technology Stack

**Frontend**:
- React 18.2.0
- TypeScript 5.2.2
- Vite 5.0.0 (build tool)
- React Query (TanStack Query) 5.8.4 - Data fetching and caching
- Axios 1.6.2 - HTTP client
- Recharts 2.8.0 - Data visualization
- Tailwind CSS 3.3.6 - Styling

**Backend**:
- FastAPI (Python)
- PostgreSQL (database)
- JWT authentication
- RESTful API endpoints

### Data Flow

```
User Browser
    ↓
React Components (DashboardLive.tsx)
    ↓
React Query Hooks (useDashboardData.ts)
    ↓
API Client Layer (lib/api/*.ts)
    ↓
Axios HTTP Client (with interceptors)
    ↓
FastAPI Backend (localhost:8001/api/v1)
    ↓
PostgreSQL Database
```

## API Client Layer

### Type-Safe API Clients

All API interactions are handled through dedicated TypeScript client modules:

#### 1. Contracts API (`lib/api/contracts.ts`)

```typescript
// List contracts with pagination
await contractsApi.listContracts({
  skip: 0,
  limit: 10,
  network: 'ethereum',
  status: 'pending'
});

// Get specific contract
await contractsApi.getContract(contractId);

// Create new contract
await contractsApi.createContract({
  name: 'MyToken',
  address: '0x123...',
  network: 'ethereum',
  source_code: '...'
});
```

**Endpoints**:
- `GET /api/v1/contracts` - List contracts
- `GET /api/v1/contracts/{id}` - Get contract details
- `POST /api/v1/contracts` - Create contract
- `PATCH /api/v1/contracts/{id}` - Update contract
- `DELETE /api/v1/contracts/{id}` - Delete contract

#### 2. Scans API (`lib/api/scans.ts`)

```typescript
// List scans
await scansApi.listScans({
  skip: 0,
  limit: 10,
  status: 'completed'
});

// Create new scan
await scansApi.createScan({
  contract_id: contractId,
  scan_type: 'full'
});
```

**Endpoints**:
- `GET /api/v1/scans` - List scans
- `GET /api/v1/scans/{id}` - Get scan details
- `POST /api/v1/scans` - Create scan

#### 3. Vulnerabilities API (`lib/api/vulnerabilities.ts`)

```typescript
// List all vulnerabilities
await vulnerabilitiesApi.listVulnerabilities({
  severity: 'critical',
  status: 'open'
});

// Get vulnerabilities for a contract
await vulnerabilitiesApi.listVulnerabilitiesByContract(contractId);

// Update vulnerability status
await vulnerabilitiesApi.updateVulnerabilityStatus(vulnId, {
  status: 'fixed'
});
```

**Endpoints**:
- `GET /api/v1/vulnerabilities` - List vulnerabilities
- `GET /api/v1/vulnerabilities/{id}` - Get vulnerability details
- `GET /api/v1/vulnerabilities/contracts/{id}/vulnerabilities` - Get by contract
- `PATCH /api/v1/vulnerabilities/{id}/status` - Update status

#### 4. Statistics API (`lib/api/statistics.ts`)

```typescript
// Get dashboard statistics
await statisticsApi.getDashboardStatistics();
// Returns: { total_scans, total_vulnerabilities, contracts_scanned, average_risk_score }

// Get 30-day scan history
await statisticsApi.getScanHistory();
// Returns: { history: [{ date, scans, vulnerabilities }] }
```

**Endpoints**:
- `GET /api/v1/statistics/dashboard` - Dashboard stats
- `GET /api/v1/statistics/scan-history` - Historical data

#### 5. Users API (`lib/api/users.ts`)

```typescript
// Get current user profile
await usersApi.getCurrentUser();

// Update profile
await usersApi.updateCurrentUser({
  email: 'new@example.com'
});
```

**Endpoints**:
- `GET /api/v1/users/me` - Get user profile
- `PATCH /api/v1/users/me` - Update profile

## React Query Integration

### Data Fetching Hooks

All data fetching uses React Query hooks for automatic caching, background updates, and error handling.

#### Dashboard Statistics Hook

```typescript
import { useDashboardStatistics } from '../hooks/useDashboardData';

function DashboardComponent() {
  const {
    data: stats,
    isLoading,
    error
  } = useDashboardStatistics();

  if (isLoading) return <Loading />;
  if (error) return <Error message={error.message} />;

  return <div>Total Scans: {stats.total_scans}</div>;
}
```

**Features**:
- Auto-refresh every 30 seconds
- Caches data for 5 minutes
- Automatic retry on failure
- Loading and error states

#### Available Hooks

```typescript
// Dashboard statistics (30s refresh)
const { data } = useDashboardStatistics();

// 30-day scan history (60s refresh)
const { data } = useScanHistory();

// Recent contracts (30s refresh)
const { data } = useRecentContracts(limit);

// Recent vulnerabilities (30s refresh)
const { data } = useRecentVulnerabilities(limit);
```

## Authentication

### JWT Token Management

The dashboard uses JWT bearer tokens for authentication:

```typescript
// Login
const response = await authApi.login({
  email: 'user@example.com',
  password: 'password'
});

// Tokens automatically stored in localStorage
localStorage.setItem('access_token', response.access_token);
localStorage.setItem('refresh_token', response.refresh_token);
```

### Automatic Token Refresh

Axios interceptors automatically handle token refresh:

```typescript
// On 401 response, attempt token refresh
if (error.response?.status === 401) {
  const refreshToken = localStorage.getItem('refresh_token');
  const response = await axios.post('/auth/refresh', { refreshToken });

  // Save new tokens
  localStorage.setItem('access_token', response.access_token);

  // Retry original request
  return apiClient(originalRequest);
}
```

## Dashboard Components

### Live Dashboard (`DashboardLive.tsx`)

The main dashboard component displays real-time data:

**Statistics Cards**:
- Total Scans
- Contracts Scanned
- Vulnerabilities Found
- Average Risk Score

**Visualizations**:
1. **Scan History Chart** (Line Chart)
   - 30-day trend of scans and vulnerabilities
   - Data from `/api/v1/statistics/scan-history`

2. **Severity Distribution** (Pie Chart)
   - Breakdown by severity (Critical/High/Medium/Low)
   - Calculated from real vulnerability data

3. **Top Vulnerability Categories** (Bar Chart)
   - Most common vulnerability types
   - Grouped by category field

**Data Tables**:
- Recent Contracts table with network, lines of code, status
- Clickable rows for future detail pages

**API Status Indicator**:
- Green pulse: API Connected
- Red: API Disconnected
- Shows service version when connected

## Error Handling

### Loading States

```typescript
if (isLoading) {
  return (
    <div className="flex items-center justify-center">
      <div className="animate-spin rounded-full h-16 w-16 border-b-2 border-blue-600" />
      <p>Loading dashboard data...</p>
    </div>
  );
}
```

### Error States

```typescript
if (error) {
  return (
    <div className="bg-red-50 border-l-4 border-red-400 p-4">
      <h3>Failed to load dashboard data</h3>
      <p>{error.message}</p>
      <p>Make sure the API service is running and you are authenticated.</p>
    </div>
  );
}
```

### Network Error Recovery

- Automatic retry on network failures
- Exponential backoff
- User-friendly error messages
- Fallback to cached data when available

## Performance Optimization

### React Query Caching

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,  // 5 minutes
      cacheTime: 10 * 60 * 1000,  // 10 minutes
      refetchOnWindowFocus: false,
      retry: 1,
    },
  },
});
```

### Code Splitting

The dashboard uses dynamic imports for route-based code splitting:

```typescript
const Dashboard = lazy(() => import('./pages/DashboardLive'));
```

### Bundle Size

- Production build: 662.93 kB
- Gzipped: 192.45 kB
- Build time: ~23 seconds

## Development Workflow

### Local Development

1. Start the API service:
```bash
kubectl port-forward -n api-service-local svc/api-service 8001:8000
```

2. Start the dashboard:
```bash
cd /Users/pwner/Git/ABS/solidity-security-dashboard
npm run dev
```

3. Set authentication token:
   - Open `file:///tmp/set-dashboard-auth.html`
   - Click "Set Authentication Token"
   - Navigate to `http://localhost:5173`

### Testing

```bash
# Run tests
npm test

# Type checking
npm run type-check

# Linting
npm run lint

# Build for production
npm run build
```

## API Testing

Comprehensive test scripts available:

```bash
# Test all endpoints
/tmp/test-api.sh

# Test vulnerabilities
/tmp/test-vulnerabilities.sh

# Test file upload
/tmp/test-upload.sh

# Test scan history
/tmp/test-scan-history.sh
```

**Results**: 14/14 endpoints passing (100% success rate)

## Deployment

### Production Build

```bash
npm run build
```

Output:
```
dist/index.html                   0.47 kB
dist/assets/index-zj84KrDI.css   13.90 kB (gzipped: 3.44 kB)
dist/assets/index-CWRXzC_4.js   662.93 kB (gzipped: 192.45 kB)
```

### Environment Variables

```bash
# .env.production
VITE_API_BASE_URL=https://api.soliditysecurity.com
```

### Docker Deployment

```dockerfile
FROM nginx:alpine
COPY dist/ /usr/share/nginx/html
EXPOSE 80
```

## Security Considerations

### Token Security

- Access tokens stored in localStorage
- Automatic token refresh before expiry
- Tokens cleared on logout
- HTTPS required in production

### API Security

- JWT bearer token authentication
- Automatic 401 handling
- CORS configured for specific origins
- Rate limiting on API endpoints

### XSS Protection

- All user input sanitized
- React's built-in XSS protection
- Content Security Policy headers

## Troubleshooting

### Common Issues

**Dashboard shows "API Disconnected"**:
- Check API service is running: `kubectl get pods -n api-service-local`
- Check port forward: `lsof -i :8001`
- Verify network connectivity

**Authentication fails**:
- Check token expiry
- Verify credentials in Vault
- Check API logs: `kubectl logs -n api-service-local deployment/api-service`

**Data not updating**:
- Check browser console for errors
- Verify React Query cache settings
- Force refresh: Clear localStorage and reload

## Future Enhancements

### Planned Features

1. **Transaction Hash Support**
   - Scan contracts by transaction hash
   - Auto-populate contract details from blockchain
   - Blockchain explorer integration

2. **Project Grouping**
   - Group multiple contracts into projects
   - Aggregated vulnerability reporting
   - Cross-contract analysis

3. **Real-Time Updates**
   - WebSocket support for live scan updates
   - Push notifications for new vulnerabilities
   - Live progress indicators

4. **Additional Pages**
   - Contracts list with search/filter
   - Vulnerability details page
   - Scan management page
   - File upload UI component

## Resources

**Documentation**:
- API Test Results: `/tmp/api-test-results-summary.md`
- Integration Summary: `/tmp/dashboard-integration-summary.md`
- Auth Helper: `/tmp/set-dashboard-auth.html`

**Code Repositories**:
- Dashboard: `https://github.com/SolidityOps/solidity-security-dashboard`
- API Service: `https://github.com/SolidityOps/solidity-security-api-service`

**Pull Requests**:
- Dashboard Integration: [#5](https://github.com/SolidityOps/solidity-security-dashboard/pull/5) ✅ Merged

## Support

For issues or questions:
1. Check API health: `GET /api/v1/health/live`
2. Review browser console logs
3. Check API logs: `kubectl logs -n api-service-local deployment/api-service`
4. Refer to test scripts in `/tmp/` for working examples
