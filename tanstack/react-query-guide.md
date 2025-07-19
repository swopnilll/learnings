# React Query & State Management

## Understanding Client State vs Server State

### Client State in LIMS App

Client state represents data that exists only in the browser session and doesn't need to be persisted on the server. In the LIMS App context:

**Examples of Client State:**
- **UI Theme**: Dark/light mode preference for the dashboard
- **Language Selection**: User's preferred language (English/Spanish/French)
- **Dashboard Layout**: Which panels are expanded/collapsed
- **Map Zoom Level**: Current zoom level on the LIMS App field map
- **Filter Settings**: Currently selected filters (date range, well type, status)
- **Modal States**: Which modals are open/closed
- **Form Input States**: Current values in forms before submission
- **Navigation State**: Current active tab or page section

```javascript
// Client State Examples
const [theme, setTheme] = useState('dark');
const [language, setLanguage] = useState('en');
const [mapZoom, setMapZoom] = useState(10);
const [selectedFilters, setSelectedFilters] = useState({
  dateRange: 'last30days',
  wellType: 'all',
  status: 'active'
});
```

### Server State in LIMS App

Server state is data that originates from and is stored on the server but needs to be displayed on the client. This data is shared across multiple users and sessions.

**Examples of Server State:**
- **Well Data**: Information about LIMS App wells (location, production rates, status)
- **Production Reports**: Daily/monthly production statistics
- **Equipment Status**: Real-time status of drilling equipment
- **User Profiles**: User account information and permissions
- **Historical Data**: Past production data and trends
- **Maintenance Records**: Equipment maintenance logs
- **Safety Reports**: Incident reports and safety metrics
- **Asset Information**: Details about LIMS App rigs, pipelines, and facilities

```javascript
// Server State Examples - Raw data structures
const wellData = {
  id: 'WELL_001',
  name: 'Thunder Ridge #1',
  location: { lat: -32.123, lng: 115.456 },
  status: 'active',
  dailyProduction: 1250, // barrels
  lastInspection: '2024-07-15',
  equipment: ['pump', 'separator', 'tank']
};

const productionReport = {
  wellId: 'WELL_001',
  date: '2024-07-18',
  LIMS AppProduced: 1250,
  gasProduced: 850,
  waterCut: 15,
  pressure: 2850
};
```

## React Query Cache Management

### The Cache as Source of Truth

React Query maintains a client-side cache that serves as the single source of truth for server data. Instead of components directly fetching data with `fetch()` or `axios`, they query the React Query cache.

```javascript
// Traditional approach (WITHOUT React Query)
const WellDashboard = () => {
  const [wells, setWells] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchWells = async () => {
      try {
        setLoading(true);
        const response = await fetch('/api/wells');
        const data = await response.json();
        setWells(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchWells();
  }, []);

  // Every component needs to manage its own loading/error states
  if (loading) return <div>Loading wells...</div>;
  if (error) return <div>Error: {error}</div>;
  
  return <div>{/* Render wells */}</div>;
};
```

```javascript
// React Query approach
import { useQuery } from '@tanstack/react-query';

const WellDashboard = () => {
  const { 
    data: wells, 
    isLoading, 
    error, 
    refetch 
  } = useQuery({
    queryKey: ['wells'],
    queryFn: async () => {
      const response = await fetch('/api/wells');
      if (!response.ok) throw new Error('Failed to fetch wells');
      return response.json();
    },
    staleTime: 30 * 1000, // 30 seconds
    refetchOnWindowFocus: true
  });

  if (isLoading) return <div>Loading wells...</div>;
  if (error) return <div>Error: {error.message}</div>;
  
  return <div>{/* Render wells */}</div>;
};
```

### Cache Invalidation and Updates

React Query provides both **imperative** and **declarative** ways to keep cache data fresh:

#### Imperative Cache Updates

```javascript
import { useQueryClient } from '@tanstack/react-query';

const WellMaintenanceForm = () => {
  const queryClient = useQueryClient();

  const handleMaintenanceComplete = async (wellId) => {
    // Update server
    await fetch(`/api/wells/${wellId}/maintenance`, {
      method: 'POST',
      body: JSON.stringify({ status: 'completed' })
    });

    // Invalidate cache - forces refetch
    queryClient.invalidateQueries({ queryKey: ['wells'] });
    queryClient.invalidateQueries({ queryKey: ['well', wellId] });
  };
};
```

#### Declarative Cache Updates

```javascript
const useWellsQuery = () => {
  return useQuery({
    queryKey: ['wells'],
    queryFn: fetchWells,
    staleTime: 30 * 1000, // Data is fresh for 30 seconds
    refetchOnWindowFocus: true, // Refetch when window regains focus
    refetchInterval: 5 * 60 * 1000, // Refetch every 5 minutes
    refetchOnReconnect: true // Refetch when internet reconnects
  });
};
```

## Advanced React Query Features for LIMS App

### 1. Loading and Error States Management

```javascript
const ProductionDashboard = () => {
  const wellsQuery = useQuery({
    queryKey: ['wells'],
    queryFn: fetchWells
  });

  const productionQuery = useQuery({
    queryKey: ['production', 'today'],
    queryFn: () => fetchProductionData(new Date())
  });

  // React Query automatically manages loading states
  if (wellsQuery.isLoading || productionQuery.isLoading) {
    return <LoadingSpinner />;
  }

  if (wellsQuery.error) {
    return <ErrorMessage message="Failed to load wells" />;
  }

  if (productionQuery.error) {
    return <ErrorMessage message="Failed to load production data" />;
  }

  return (
    <div>
      <WellsList wells={wellsQuery.data} />
      <ProductionChart data={productionQuery.data} />
    </div>
  );
};
```

### 2. Pagination and Infinite Scroll

```javascript
const useWellsPagination = (filters) => {
  return useInfiniteQuery({
    queryKey: ['wells', 'infinite', filters],
    queryFn: ({ pageParam = 0 }) => 
      fetchWells({ ...filters, page: pageParam }),
    getNextPageParam: (lastPage, pages) => {
      if (lastPage.hasMore) return pages.length;
      return undefined;
    },
    initialPageParam: 0
  });
};

const WellsList = () => {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage
  } = useWellsPagination({ status: 'active' });

  return (
    <div>
      {data?.pages.map((page, i) => (
        <div key={i}>
          {page.wells.map(well => (
            <WellCard key={well.id} well={well} />
          ))}
        </div>
      ))}
      
      {hasNextPage && (
        <button 
          onClick={() => fetchNextPage()}
          disabled={isFetchingNextPage}
        >
          {isFetchingNextPage ? 'Loading...' : 'Load More Wells'}
        </button>
      )}
    </div>
  );
};
```

### 3. Prefetching Data

```javascript
const WellCard = ({ well }) => {
  const queryClient = useQueryClient();

  const handleWellHover = () => {
    // Prefetch detailed well data on hover
    queryClient.prefetchQuery({
      queryKey: ['well', well.id, 'details'],
      queryFn: () => fetchWellDetails(well.id),
      staleTime: 10 * 60 * 1000 // 10 minutes
    });
  };

  return (
    <div 
      onMouseEnter={handleWellHover}
      onClick={() => navigate(`/wells/${well.id}`)}
    >
      <h3>{well.name}</h3>
      <p>Production: {well.dailyProduction} barrels</p>
    </div>
  );
};
```

### 4. Mutations and Optimistic Updates

```javascript
const useUpdateWellStatus = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ wellId, status }) => {
      const response = await fetch(`/api/wells/${wellId}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ status })
      });
      return response.json();
    },
    
    // Optimistic update - update UI immediately
    onMutate: async ({ wellId, status }) => {
      await queryClient.cancelQueries({ queryKey: ['wells'] });
      
      const previousWells = queryClient.getQueryData(['wells']);
      
      queryClient.setQueryData(['wells'], (oldData) =>
        oldData.map(well => 
          well.id === wellId ? { ...well, status } : well
        )
      );
      
      return { previousWells };
    },
    
    // Rollback on error
    onError: (err, variables, context) => {
      queryClient.setQueryData(['wells'], context.previousWells);
    },
    
    // Refetch on success to ensure consistency
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['wells'] });
    }
  });
};
```

### 5. Request Deduplication

```javascript
// Multiple components can request the same data
const WellOverview = ({ wellId }) => {
  const { data } = useQuery({
    queryKey: ['well', wellId],
    queryFn: () => fetchWellDetails(wellId)
  });
  
  return <div>{data?.name}</div>;
};

const WellProduction = ({ wellId }) => {
  // Same query key - React Query will deduplicate this request
  const { data } = useQuery({
    queryKey: ['well', wellId], // Same key as above
    queryFn: () => fetchWellDetails(wellId)
  });
  
  return <div>{data?.dailyProduction}</div>;
};
```

### 6. Retry Logic and Callbacks

```javascript
const useWellsQuery = () => {
  return useQuery({
    queryKey: ['wells'],
    queryFn: fetchWells,
    retry: (failureCount, error) => {
      // Don't retry on 404
      if (error.status === 404) return false;
      // Retry up to 3 times for other errors
      return failureCount < 3;
    },
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
    
    onSuccess: (data) => {
      console.log('Wells loaded successfully:', data.length);
      // Could trigger analytics or notifications
    },
    
    onError: (error) => {
      console.error('Failed to load wells:', error);
      // Could show toast notification
      showErrorToast('Failed to load wells data');
    }
  });
};
```

## Next.js Approach (Latest Version)

Next.js 13+ with the App Router handles server and client state differently:

### Server Components and Server State

```javascript
// app/wells/page.js (Server Component)
import { Suspense } from 'react';
import WellsList from './WellsList';
import { fetchWells } from '@/lib/api';

export default async function WellsPage() {
  // This runs on the server
  const wells = await fetchWells();

  return (
    <div>
      <h1>LIMS App Wells Dashboard</h1>
      <Suspense fallback={<div>Loading wells...</div>}>
        <WellsList initialWells={wells} />
      </Suspense>
    </div>
  );
}
```

### Client Components for Interactive State

```javascript
// app/wells/WellsList.jsx (Client Component)
'use client';

import { useState, useEffect } from 'react';

export default function WellsList({ initialWells }) {
  const [wells, setWells] = useState(initialWells);
  const [filter, setFilter] = useState('all'); // Client state
  const [theme, setTheme] = useState('dark'); // Client state

  // Client-side filtering
  const filteredWells = wells.filter(well => 
    filter === 'all' || well.status === filter
  );

  return (
    <div className={theme === 'dark' ? 'dark' : ''}>
      <div>
        <select 
          value={filter} 
          onChange={(e) => setFilter(e.target.value)}
        >
          <option value="all">All Wells</option>
          <option value="active">Active</option>
          <option value="inactive">Inactive</option>
        </select>
        
        <button onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}>
          Toggle Theme
        </button>
      </div>
      
      {filteredWells.map(well => (
        <WellCard key={well.id} well={well} />
      ))}
    </div>
  );
}
```

### Key Differences: Next.js vs React Query

| Aspect | React Query | Next.js App Router |
|--------|-------------|-------------------|
| **Server State** | Cached on client, managed by React Query | Fetched on server, streamed to client |
| **Loading States** | Automatic via `isLoading` | Manual with `loading.js` files |
| **Error Handling** | Automatic via `error` state | Manual with `error.js` files |
| **Caching** | Client-side cache with TTL | Server-side caching, CDN caching |
| **Revalidation** | Background refetching | On-demand or time-based revalidation |
| **Hydration** | Client-side hydration of cached data | Server-rendered HTML with selective hydration |

### Combined Approach

You can use both together for optimal performance:

```javascript
// Server Component for initial data
export default async function WellsPage() {
  const initialWells = await fetchWells();
  
  return (
    <HydrationBoundary>
      <WellsWithReactQuery initialWells={initialWells} />
    </HydrationBoundary>
  );
}

// Client Component with React Query
'use client';
function WellsWithReactQuery({ initialWells }) {
  const { data: wells, isLoading, error } = useQuery({
    queryKey: ['wells'],
    queryFn: fetchWells,
    initialData: initialWells,
    staleTime: 30 * 1000
  });

  // React Query manages updates, while Next.js provided initial data
  return <WellsList wells={wells} />;
}
```

This hybrid approach gives you the best of both worlds: fast initial page loads with server-side rendering, and powerful client-side state management with React Query for interactive features.