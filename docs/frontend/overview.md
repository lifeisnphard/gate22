# Frontend Architecture

The Gate22 frontend is a modern Next.js 15 application built with TypeScript, providing an intuitive management portal for the MCP gateway platform.

## Project Structure

```
frontend/
├── src/
│   ├── app/                   # Next.js App Router
│   │   ├── (auth)/           # Authentication routes
│   │   │   └── login/
│   │   ├── (dashboard)/      # Main app routes
│   │   │   ├── agents/       # Agent bundles
│   │   │   ├── apps/         # Applications
│   │   │   ├── mcp-servers/  # MCP server management
│   │   │   ├── settings/     # Settings
│   │   │   └── members/      # Team members
│   │   ├── (landing)/        # Public landing page
│   │   ├── layout.tsx        # Root layout
│   │   └── globals.css       # Global styles
│   │
│   ├── features/             # Feature modules
│   │   ├── auth/            # Authentication
│   │   │   ├── api/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── types/
│   │   │   └── store/
│   │   ├── mcp/             # MCP management
│   │   ├── bundles/         # Bundle management
│   │   └── ...
│   │
│   ├── components/          # Shared components
│   │   ├── ui/             # Base UI components
│   │   ├── ui-extensions/  # Enhanced components
│   │   ├── layout/         # Layout components
│   │   └── providers/      # Context providers
│   │
│   ├── lib/                # Utilities
│   │   ├── api-client.ts   # API client
│   │   ├── utils.ts        # Helper functions
│   │   └── constants.ts    # Constants
│   │
│   └── config/             # Configuration
│       └── api.constants.ts
│
├── public/                 # Static assets
│   ├── icons/
│   └── logos/
│
├── next.config.ts          # Next.js config
├── tailwind.config.ts      # Tailwind config
├── tsconfig.json          # TypeScript config
├── components.json        # shadcn/ui config
└── package.json           # Dependencies
```

## Technology Stack

### Core Framework

- **Next.js 15** - React framework with App Router
- **TypeScript 5** - Type-safe development
- **React 19** - UI library

### Styling & UI

- **Tailwind CSS v4** - Utility-first CSS
- **Radix UI** - Unstyled, accessible components
- **shadcn/ui** - Beautifully designed components
- **Lucide Icons** - Icon library

### State Management

- **Zustand** - Lightweight state management
- **React Query (TanStack Query)** - Server state management
- **React Hook Form** - Form state management
- **Zod** - Schema validation

### Testing

- **Vitest** - Unit testing framework
- **Testing Library** - React component testing
- **MSW (Mock Service Worker)** - API mocking

## Key Features

### 1. Authentication

Location: `src/features/auth/`

#### Google OAuth Flow

```typescript
// src/features/auth/components/GoogleLoginButton.tsx

export function GoogleLoginButton() {
  const handleGoogleLogin = async () => {
    // Redirect to Google OAuth
    const authUrl = `https://accounts.google.com/o/oauth2/v2/auth?${new URLSearchParams({
      client_id: process.env.NEXT_PUBLIC_GOOGLE_CLIENT_ID!,
      redirect_uri: `${process.env.NEXT_PUBLIC_APP_URL}/auth/google/callback`,
      response_type: 'code',
      scope: 'openid email profile',
    })}`;
    
    window.location.href = authUrl;
  };
  
  return (
    <Button onClick={handleGoogleLogin}>
      <GoogleIcon className="mr-2" />
      Sign in with Google
    </Button>
  );
}
```

#### OAuth Callback Handler

```typescript
// src/app/(auth)/login/google/callback/page.tsx

export default async function GoogleCallbackPage({
  searchParams,
}: {
  searchParams: { code?: string };
}) {
  if (!searchParams.code) {
    redirect('/login?error=no_code');
  }
  
  // Exchange code for tokens
  const response = await fetch(`${API_URL}/v1/control-plane/auth/google`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ code: searchParams.code }),
  });
  
  const { access_token, refresh_token } = await response.json();
  
  // Store tokens and redirect
  cookies().set('access_token', access_token, { httpOnly: true });
  cookies().set('refresh_token', refresh_token, { httpOnly: true });
  
  redirect('/');
}
```

### 2. MCP Server Management

Location: `src/features/mcp/`

#### Server List Component

```typescript
// src/features/mcp/components/MCPServerList.tsx

export function MCPServerList() {
  const { data: servers, isLoading } = useMCPServers();
  const [filters, setFilters] = useState<ServerFilters>({
    category: 'all',
    search: '',
  });
  
  const filteredServers = useMemo(() => {
    return servers?.filter(server => {
      const matchesCategory = filters.category === 'all' || 
        server.categories.includes(filters.category);
      const matchesSearch = server.name.toLowerCase()
        .includes(filters.search.toLowerCase());
      
      return matchesCategory && matchesSearch;
    });
  }, [servers, filters]);
  
  return (
    <div className="space-y-4">
      <ServerFilters filters={filters} onChange={setFilters} />
      
      <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
        {filteredServers?.map(server => (
          <ServerCard key={server.id} server={server} />
        ))}
      </div>
    </div>
  );
}
```

#### Custom MCP Server Creation

```typescript
// src/app/(dashboard)/mcp-servers/custom/new/page.tsx

export default function AddCustomMCPServerPage() {
  const [currentStep, setCurrentStep] = useState(1);
  const [serverDetails, setServerDetails] = useState({
    name: '',
    url: '',
    description: '',
  });
  
  const [authMethods, setAuthMethods] = useState({
    no_auth: false,
    api_key: false,
    oauth2: false,
  });
  
  const createServer = async () => {
    const authConfigs: AuthConfig[] = [];
    
    // Build auth configs
    if (authMethods.no_auth) {
      authConfigs.push({ type: 'no_auth' });
    }
    
    if (authMethods.api_key) {
      authConfigs.push({
        type: 'api_key',
        location: apiKeyLocation,
        name: apiKeyName,
        prefix: apiKeyPrefix,
      });
    }
    
    if (authMethods.oauth2) {
      authConfigs.push({
        type: 'oauth2',
        authorize_url: authorizeUrl,
        access_token_url: tokenUrl,
        client_id: clientId,
        client_secret: clientSecret,
        scopes: scopes.split(',').map(s => s.trim()),
      });
    }
    
    // Create server
    await mcpService.servers.create(token, {
      ...serverDetails,
      auth_configs: authConfigs,
    });
  };
  
  // Multi-step form implementation...
}
```

### 3. Bundle Management

Location: `src/features/bundles/`

#### Bundle Creation Flow

```typescript
// src/features/bundles/components/BundleCreator.tsx

export function BundleCreator() {
  const { data: configurations } = useMCPConfigurations();
  const [selectedConfigs, setSelectedConfigs] = useState<string[]>([]);
  const [bundleDetails, setBundleDetails] = useState({
    name: '',
    description: '',
  });
  
  const createBundle = async () => {
    const bundle = await bundleService.create(token, {
      ...bundleDetails,
      mcp_server_configuration_ids: selectedConfigs,
    });
    
    toast.success('Bundle created successfully!');
    router.push(`/bundles/${bundle.id}`);
  };
  
  return (
    <div className="space-y-6">
      <BundleDetailsForm 
        details={bundleDetails}
        onChange={setBundleDetails}
      />
      
      <ConfigurationSelector
        configurations={configurations}
        selected={selectedConfigs}
        onChange={setSelectedConfigs}
      />
      
      <ToolSelector
        configurations={selectedConfigs}
        onToolsChange={handleToolSelection}
      />
      
      <Button onClick={createBundle} disabled={!canCreate}>
        Create Bundle
      </Button>
    </div>
  );
}
```

#### Bundle Connection Info

```typescript
// src/features/bundles/components/BundleConnectionInfo.tsx

export function BundleConnectionInfo({ bundle }: { bundle: Bundle }) {
  const mcpUrl = `${MCP_BASE_URL}/${bundle.bundle_key}`;
  
  const connectionConfig = {
    mcpServers: {
      [bundle.name]: {
        url: mcpUrl,
        headers: {
          Authorization: `Bearer ${bundle.bundle_key}`,
        },
      },
    },
  };
  
  return (
    <Card>
      <CardHeader>
        <CardTitle>Connection Details</CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <div>
          <Label>MCP Server URL</Label>
          <div className="flex gap-2">
            <Input value={mcpUrl} readOnly />
            <Button onClick={() => copyToClipboard(mcpUrl)}>
              <CopyIcon />
            </Button>
          </div>
        </div>
        
        <div>
          <Label>Bundle Key</Label>
          <div className="flex gap-2">
            <Input 
              value={bundle.bundle_key} 
              type="password" 
              readOnly 
            />
            <Button onClick={() => copyToClipboard(bundle.bundle_key)}>
              <CopyIcon />
            </Button>
          </div>
        </div>
        
        <div>
          <Label>Configuration (for Cursor, Claude Desktop, etc.)</Label>
          <CodeBlock language="json">
            {JSON.stringify(connectionConfig, null, 2)}
          </CodeBlock>
        </div>
      </CardContent>
    </Card>
  );
}
```

### 4. API Client

Location: `src/lib/api-client.ts`

```typescript
// Authenticated API client

export function createAuthenticatedRequest(token: string) {
  const baseURL = process.env.NEXT_PUBLIC_API_URL;
  
  async function request<T>(
    endpoint: string,
    options: RequestInit = {}
  ): Promise<T> {
    const response = await fetch(`${baseURL}${endpoint}`, {
      ...options,
      headers: {
        ...options.headers,
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
    });
    
    if (!response.ok) {
      if (response.status === 401) {
        // Handle token expiration
        await handleTokenRefresh();
        // Retry request
        return request(endpoint, options);
      }
      
      const error = await response.json();
      throw new APIError(error.message, response.status);
    }
    
    return response.json();
  }
  
  return {
    get: <T>(url: string) => request<T>(url, { method: 'GET' }),
    post: <T>(url: string, data: any) => 
      request<T>(url, { method: 'POST', body: JSON.stringify(data) }),
    patch: <T>(url: string, data: any) => 
      request<T>(url, { method: 'PATCH', body: JSON.stringify(data) }),
    delete: (url: string) => 
      request(url, { method: 'DELETE' }),
  };
}
```

### 5. State Management

#### Zustand Store Example

```typescript
// src/features/auth/store/authStore.ts

import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface AuthState {
  user: User | null;
  accessToken: string | null;
  isAuthenticated: boolean;
  
  setUser: (user: User) => void;
  setAccessToken: (token: string) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      accessToken: null,
      isAuthenticated: false,
      
      setUser: (user) => set({ user, isAuthenticated: true }),
      setAccessToken: (token) => set({ accessToken: token }),
      logout: () => set({ 
        user: null, 
        accessToken: null, 
        isAuthenticated: false 
      }),
    }),
    {
      name: 'auth-storage',
      partialize: (state) => ({ 
        user: state.user,
        isAuthenticated: state.isAuthenticated,
      }),
    }
  )
);
```

#### React Query Hooks

```typescript
// src/features/mcp/hooks/useMCPServers.ts

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

export function useMCPServers(params?: PaginationParams) {
  const { data: session } = useSession();
  
  return useQuery({
    queryKey: ['mcp-servers', params],
    queryFn: () => mcpService.servers.list(session.accessToken, params),
    enabled: !!session?.accessToken,
  });
}

export function useCreateMCPServer() {
  const queryClient = useQueryClient();
  const { data: session } = useSession();
  
  return useMutation({
    mutationFn: (data: MCPServerCreate) => 
      mcpService.servers.create(session.accessToken, data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['mcp-servers'] });
      toast.success('MCP server created successfully!');
    },
    onError: (error) => {
      toast.error(`Failed to create server: ${error.message}`);
    },
  });
}
```

## Routing

### App Router Structure

```typescript
// Dynamic routing example
// src/app/(dashboard)/mcp-servers/[serverId]/page.tsx

export default async function MCPServerDetailPage({
  params,
}: {
  params: { serverId: string };
}) {
  const server = await mcpService.servers.getById(params.serverId);
  
  return (
    <div>
      <ServerHeader server={server} />
      <ServerDetails server={server} />
      <ServerTools serverId={server.id} />
    </div>
  );
}
```

### Protected Routes

```typescript
// src/app/(dashboard)/layout.tsx

export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const { isAuthenticated } = useAuthStore();
  
  if (!isAuthenticated) {
    redirect('/login');
  }
  
  return (
    <div className="flex h-screen">
      <Sidebar />
      <main className="flex-1 overflow-auto">
        {children}
      </main>
    </div>
  );
}
```

## Component Patterns

### UI Components (shadcn/ui)

```typescript
// Example: Button component
// src/components/ui/button.tsx

const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input hover:bg-accent hover:text-accent-foreground",
        ghost: "hover:bg-accent hover:text-accent-foreground",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

export const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, ...props }, ref) => {
    return (
      <button
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    );
  }
);
```

### Data Tables

```typescript
// src/components/ui-extensions/enhanced-data-table/

export function DataTable<TData>({
  columns,
  data,
  onRowClick,
}: DataTableProps<TData>) {
  const table = useReactTable({
    data,
    columns,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
  });
  
  return (
    <div className="rounded-md border">
      <Table>
        <TableHeader>
          {table.getHeaderGroups().map((headerGroup) => (
            <TableRow key={headerGroup.id}>
              {headerGroup.headers.map((header) => (
                <TableHead key={header.id}>
                  {flexRender(
                    header.column.columnDef.header,
                    header.getContext()
                  )}
                </TableHead>
              ))}
            </TableRow>
          ))}
        </TableHeader>
        <TableBody>
          {table.getRowModel().rows.map((row) => (
            <TableRow 
              key={row.id}
              onClick={() => onRowClick?.(row.original)}
              className="cursor-pointer hover:bg-muted"
            >
              {row.getVisibleCells().map((cell) => (
                <TableCell key={cell.id}>
                  {flexRender(
                    cell.column.columnDef.cell,
                    cell.getContext()
                  )}
                </TableCell>
              ))}
            </TableRow>
          ))}
        </TableBody>
      </Table>
    </div>
  );
}
```

## Testing

### Component Tests

```typescript
// src/features/mcp/components/__tests__/MCPServerCard.test.tsx

import { render, screen } from '@testing-library/react';
import { MCPServerCard } from '../MCPServerCard';

describe('MCPServerCard', () => {
  const mockServer = {
    id: '123',
    name: 'Test Server',
    description: 'Test description',
    logo: '/logo.png',
    categories: ['Data'],
  };
  
  it('renders server details correctly', () => {
    render(<MCPServerCard server={mockServer} />);
    
    expect(screen.getByText('Test Server')).toBeInTheDocument();
    expect(screen.getByText('Test description')).toBeInTheDocument();
    expect(screen.getByAltText('Test Server logo')).toBeInTheDocument();
  });
  
  it('handles click events', async () => {
    const handleClick = vi.fn();
    render(<MCPServerCard server={mockServer} onClick={handleClick} />);
    
    await userEvent.click(screen.getByRole('button'));
    expect(handleClick).toHaveBeenCalledWith(mockServer);
  });
});
```

### API Mocking

```typescript
// src/mocks/handlers.ts

import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/v1/control-plane/mcp-servers', () => {
    return HttpResponse.json({
      data: [
        { id: '1', name: 'Server 1', description: 'Test' },
        { id: '2', name: 'Server 2', description: 'Test' },
      ],
      total: 2,
    });
  }),
  
  http.post('/v1/control-plane/bundles', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({
      id: '123',
      ...body,
      bundle_key: 'test-key-123',
    });
  }),
];
```

## Build & Deployment

### Build Configuration

```typescript
// next.config.ts

const nextConfig = {
  output: 'standalone',
  
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: `${process.env.NEXT_PUBLIC_API_URL}/:path*`,
      },
    ];
  },
  
  images: {
    domains: ['raw.githubusercontent.com'],
  },
};

export default nextConfig;
```

### Environment Variables

```bash
# .env.local

NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_ENVIRONMENT=development
NEXT_PUBLIC_GOOGLE_CLIENT_ID=your-google-client-id
```

### Production Build

```bash
# Build for production
npm run build

# Start production server
npm run start

# Run linting
npm run lint

# Run tests
npm run test
```

## Best Practices

1. **Type Safety**: Use TypeScript strictly, avoid `any`
2. **Component Composition**: Build small, reusable components
3. **Server Components**: Use Server Components where possible for better performance
4. **Error Handling**: Implement proper error boundaries
5. **Loading States**: Always show loading indicators
6. **Accessibility**: Follow WCAG guidelines
7. **Performance**: Optimize images, lazy load components
8. **Security**: Never expose sensitive data client-side

## Next Steps

- [Backend Architecture](../backend/overview.md)
- [MCP Servers Guide](../mcp-servers/overview.md)
- [API Reference](../api-reference.md)
