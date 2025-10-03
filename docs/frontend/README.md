# Frontend Documentation

## Overview

The SurfSense web application is a modern Next.js-based frontend providing a rich user interface for knowledge management and AI-powered search.

**Location**: `surfsense_web/`

**Key Technologies**:
- Next.js 15.2.3 (App Router)
- React 19.0.0
- TypeScript
- Tailwind CSS 4.x
- Shadcn/ui components
- Vercel AI SDK

## Directory Structure

```
surfsense_web/
├── app/                     # Next.js App Router pages
│   ├── (home)/             # Marketing/public pages
│   │   ├── login/
│   │   ├── register/
│   │   ├── pricing/
│   │   └── ...
│   ├── api/                # API routes (server-side)
│   ├── auth/               # Authentication callbacks
│   ├── dashboard/          # Main application
│   │   ├── [search_space_id]/  # Dynamic search space routes
│   │   │   ├── researcher/     # AI research interface
│   │   │   ├── chats/          # Chat management
│   │   │   ├── documents/      # Document management
│   │   │   ├── podcasts/       # Podcast generation
│   │   │   ├── connectors/     # External integrations
│   │   │   └── logs/           # Task logs
│   │   ├── api-key/        # API key management
│   │   └── searchspaces/   # Search space management
│   ├── docs/               # Documentation (Fumadocs)
│   ├── db/                 # Database utilities
│   ├── onboard/            # Onboarding flow
│   └── settings/           # User settings
├── components/             # React components
│   ├── ui/                # Shadcn/ui components
│   ├── homepage/          # Marketing components
│   └── ...
├── hooks/                 # Custom React hooks
├── lib/                   # Utility functions
├── public/                # Static assets
├── content/               # MDX content for docs
└── contracts/             # TypeScript interfaces
```

## Core Features

### 1. App Router Architecture

Next.js 15 App Router with file-based routing:

```
app/
├── page.tsx                    → /
├── dashboard/
│   ├── page.tsx               → /dashboard
│   ├── searchspaces/
│   │   └── page.tsx          → /dashboard/searchspaces
│   └── [search_space_id]/
│       ├── page.tsx           → /dashboard/:id
│       ├── researcher/
│       │   └── [[...chat_id]]/
│       │       └── page.tsx   → /dashboard/:id/researcher
│       │                        /dashboard/:id/researcher/:chat_id
│       └── documents/
│           ├── (manage)/
│           │   └── page.tsx   → /dashboard/:id/documents
│           ├── upload/
│           │   └── page.tsx   → /dashboard/:id/documents/upload
│           └── youtube/
│               └── page.tsx   → /dashboard/:id/documents/youtube
```

**Key Patterns**:
- `[param]` - Dynamic route parameter
- `[[...param]]` - Optional catch-all route
- `(folder)` - Route group (doesn't affect URL)

### 2. Authentication Flow

**Authentication Hook** (`hooks/use-auth.ts`):
```typescript
export function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('surfsense_bearer_token');
    if (token) {
      verifyToken(token);
    }
  }, []);

  const verifyToken = async (token: string) => {
    const response = await fetch('/verify-token', {
      headers: { Authorization: `Bearer ${token}` }
    });
    // Handle response
  };

  return { user, loading, login, logout };
}
```

**Protected Routes**:
```typescript
// app/dashboard/layout.tsx
export default function DashboardLayout({ children }) {
  const { user, loading } = useAuth();

  if (loading) return <LoadingSpinner />;
  if (!user) redirect('/login');

  return <>{children}</>;
}
```

### 3. State Management

**Local State** (useState):
- Component-level UI state
- Form inputs
- Modal visibility

**URL State** (useSearchParams, useParams):
- Current search space ID
- Pagination
- Filters

**Server State** (Custom hooks):
- Documents
- Chats
- Connectors
- Search spaces

### 4. Custom Hooks

#### useDocuments (`hooks/use-documents.ts`)

Fetch and manage documents for a search space.

```typescript
export function useDocuments(
  searchSpaceId: number,
  options?: UseDocumentsOptions
) {
  const [documents, setDocuments] = useState<Document[]>([]);
  const [total, setTotal] = useState(0);
  const [loading, setLoading] = useState(false);

  const fetchDocuments = useCallback(async (page?: number, pageSize?: number) => {
    setLoading(true);
    const params = new URLSearchParams({
      search_space_id: searchSpaceId.toString(),
      page: page?.toString() || '1',
      page_size: pageSize?.toString() || '300',
    });

    const response = await fetch(
      `${process.env.NEXT_PUBLIC_FASTAPI_BACKEND_URL}/api/v1/documents/?${params}`,
      {
        headers: {
          Authorization: `Bearer ${localStorage.getItem('surfsense_bearer_token')}`,
        },
      }
    );

    const data = await response.json();
    setDocuments(data.items);
    setTotal(data.total);
    setLoading(false);
  }, [searchSpaceId]);

  useEffect(() => {
    if (!options?.lazy) {
      fetchDocuments();
    }
  }, [searchSpaceId]);

  return { documents, total, loading, fetchDocuments, refetch: fetchDocuments };
}
```

**Usage**:
```typescript
function DocumentsList() {
  const { documents, loading } = useDocuments(searchSpaceId);

  if (loading) return <Skeleton />;

  return (
    <div>
      {documents.map(doc => (
        <DocumentCard key={doc.id} document={doc} />
      ))}
    </div>
  );
}
```

#### useSearchSpaces (`hooks/use-search-spaces.ts`)

Manage search spaces.

```typescript
export function useSearchSpaces() {
  const [searchSpaces, setSearchSpaces] = useState<SearchSpace[]>([]);
  const [loading, setLoading] = useState(true);

  const fetchSearchSpaces = async () => {
    // Fetch from API
  };

  const createSearchSpace = async (data: SearchSpaceCreate) => {
    const response = await fetch('/api/v1/searchspaces/', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${getToken()}`,
      },
      body: JSON.stringify(data),
    });
    
    const newSpace = await response.json();
    setSearchSpaces([...searchSpaces, newSpace]);
    return newSpace;
  };

  const deleteSearchSpace = async (id: number) => {
    await fetch(`/api/v1/searchspaces/${id}`, {
      method: 'DELETE',
      headers: { Authorization: `Bearer ${getToken()}` },
    });
    
    setSearchSpaces(searchSpaces.filter(s => s.id !== id));
  };

  return {
    searchSpaces,
    loading,
    fetchSearchSpaces,
    createSearchSpace,
    deleteSearchSpace,
  };
}
```

### 5. Key Pages

#### Research Page (`app/dashboard/[search_space_id]/researcher/[[...chat_id]]/page.tsx`)

The main AI research interface with streaming responses.

**Features**:
- Real-time streaming chat
- Multiple research modes (QNA, General, Deep, Deeper)
- Citation tracking
- Follow-up questions
- Connector selection

**Streaming Implementation**:
```typescript
const handleSubmit = async (message: string) => {
  const response = await fetch('/chat', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${getToken()}`,
    },
    body: JSON.stringify({
      messages: [...history, { role: 'user', content: message }],
      data: {
        search_space_id: searchSpaceId,
        research_mode: researchMode,
        search_mode: 'CHUNKS',
        selected_connectors: connectors,
      },
    }),
  });

  const reader = response.body!.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    const text = decoder.decode(value);
    const lines = text.split('\n');

    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const data = line.slice(6);
        
        if (data === '[DONE]') {
          setStreaming(false);
          break;
        }

        const event = JSON.parse(data);
        handleStreamEvent(event);
      }
    }
  }
};

const handleStreamEvent = (event: any) => {
  switch (event.type) {
    case 'section_title':
      appendToCurrentMessage(`\n## ${event.content}\n`);
      break;
    case 'section_content':
      appendToCurrentMessage(event.content);
      break;
    case 'citation':
      addCitation(event);
      break;
    case 'followup_questions':
      setFollowUpQuestions(event.questions);
      break;
  }
};
```

#### Document Upload Page (`app/dashboard/[search_space_id]/documents/upload/page.tsx`)

File upload interface with drag-and-drop.

**Features**:
- Drag-and-drop upload
- Multiple file selection
- File type validation
- Upload progress
- File preview

**Implementation**:
```typescript
const { getRootProps, getInputProps, isDragActive } = useDropzone({
  onDrop: (acceptedFiles) => {
    setFiles([...files, ...acceptedFiles]);
  },
  accept: {
    'application/pdf': ['.pdf'],
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document': ['.docx'],
    'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet': ['.xlsx'],
    'image/*': ['.png', '.jpg', '.jpeg'],
    // ... more types
  },
});

const handleUpload = async () => {
  const formData = new FormData();
  files.forEach(file => formData.append('files', file));
  formData.append('search_space_id', searchSpaceId);

  const response = await fetch('/api/v1/documents/fileupload', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${getToken()}`,
    },
    body: formData,
  });

  if (response.ok) {
    toast.success('Files uploaded successfully');
    router.push(`/dashboard/${searchSpaceId}/documents`);
  }
};
```

#### Documents Management (`app/dashboard/[search_space_id]/documents/(manage)/page.tsx`)

List and manage documents with search and filtering.

**Features**:
- Paginated document list
- Search by title
- Delete documents
- View document details

**Data Table** (using @tanstack/react-table):
```typescript
const columns: ColumnDef<Document>[] = [
  {
    accessorKey: 'title',
    header: 'Title',
  },
  {
    accessorKey: 'document_type',
    header: 'Type',
    cell: ({ row }) => {
      const type = row.getValue('document_type');
      return <Badge>{type}</Badge>;
    },
  },
  {
    accessorKey: 'created_at',
    header: 'Created',
    cell: ({ row }) => {
      const date = new Date(row.getValue('created_at'));
      return <span>{date.toLocaleDateString()}</span>;
    },
  },
  {
    id: 'actions',
    cell: ({ row }) => {
      return (
        <DropdownMenu>
          <DropdownMenuTrigger asChild>
            <Button variant="ghost">...</Button>
          </DropdownMenuTrigger>
          <DropdownMenuContent>
            <DropdownMenuItem onClick={() => viewDocument(row.original.id)}>
              View
            </DropdownMenuItem>
            <DropdownMenuItem onClick={() => deleteDocument(row.original.id)}>
              Delete
            </DropdownMenuItem>
          </DropdownMenuContent>
        </DropdownMenu>
      );
    },
  },
];
```

### 6. UI Components

#### Shadcn/ui Components (`components/ui/`)

Pre-built, customizable components:
- Button, Input, Select, Checkbox
- Card, Dialog, Sheet, Popover
- Table, DataTable
- Toast notifications (Sonner)
- Form components

**Example Usage**:
```typescript
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

function MyComponent() {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Upload Documents</CardTitle>
      </CardHeader>
      <CardContent>
        <Input type="file" />
        <Button>Upload</Button>
      </CardContent>
    </Card>
  );
}
```

#### Custom Components

**DocumentCard** (`components/document-card.tsx`):
```typescript
interface DocumentCardProps {
  document: Document;
  onDelete?: (id: number) => void;
}

export function DocumentCard({ document, onDelete }: DocumentCardProps) {
  return (
    <Card>
      <CardHeader>
        <div className="flex items-start justify-between">
          <div>
            <CardTitle className="text-lg">{document.title}</CardTitle>
            <p className="text-sm text-muted-foreground">
              {new Date(document.created_at).toLocaleDateString()}
            </p>
          </div>
          <Badge>{document.document_type}</Badge>
        </div>
      </CardHeader>
      <CardContent>
        <p className="line-clamp-3 text-sm">{document.content}</p>
      </CardContent>
      <CardFooter>
        <Button variant="ghost" size="sm" onClick={() => onDelete?.(document.id)}>
          Delete
        </Button>
      </CardFooter>
    </Card>
  );
}
```

### 7. Styling

**Tailwind CSS Configuration** (`tailwind.config.js`):
```javascript
module.exports = {
  content: [
    './pages/**/*.{ts,tsx}',
    './components/**/*.{ts,tsx}',
    './app/**/*.{ts,tsx}',
  ],
  theme: {
    extend: {
      colors: {
        border: 'hsl(var(--border))',
        input: 'hsl(var(--input))',
        ring: 'hsl(var(--ring))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
        },
        // ... more colors
      },
    },
  },
  plugins: [require('tailwindcss-animate')],
};
```

**CSS Variables** (`app/globals.css`):
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --primary: 222.2 47.4% 11.2%;
    --primary-foreground: 210 40% 98%;
    /* ... more variables */
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    /* ... dark mode variables */
  }
}
```

### 8. API Integration

**API Client** (`lib/api.ts`):
```typescript
const BASE_URL = process.env.NEXT_PUBLIC_FASTAPI_BACKEND_URL;

export class ApiClient {
  private getHeaders() {
    const token = localStorage.getItem('surfsense_bearer_token');
    return {
      'Content-Type': 'application/json',
      ...(token && { Authorization: `Bearer ${token}` }),
    };
  }

  async get<T>(endpoint: string): Promise<T> {
    const response = await fetch(`${BASE_URL}${endpoint}`, {
      headers: this.getHeaders(),
    });
    
    if (!response.ok) {
      throw new Error(await response.text());
    }
    
    return response.json();
  }

  async post<T>(endpoint: string, data: any): Promise<T> {
    const response = await fetch(`${BASE_URL}${endpoint}`, {
      method: 'POST',
      headers: this.getHeaders(),
      body: JSON.stringify(data),
    });
    
    if (!response.ok) {
      throw new Error(await response.text());
    }
    
    return response.json();
  }

  // ... put, delete methods
}

export const api = new ApiClient();
```

### 9. Error Handling

**Error Boundaries**:
```typescript
'use client';

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen">
      <h2 className="text-2xl font-bold mb-4">Something went wrong!</h2>
      <p className="text-muted-foreground mb-4">{error.message}</p>
      <Button onClick={reset}>Try again</Button>
    </div>
  );
}
```

**Toast Notifications** (Sonner):
```typescript
import { toast } from 'sonner';

// Success
toast.success('Document uploaded successfully');

// Error
toast.error('Failed to upload document', {
  description: error.message,
});

// Loading
const toastId = toast.loading('Uploading...');
// Later:
toast.success('Upload complete', { id: toastId });
```

### 10. Performance Optimization

**Code Splitting**:
- Automatic with Next.js App Router
- Dynamic imports for heavy components

**Lazy Loading**:
```typescript
import dynamic from 'next/dynamic';

const HeavyComponent = dynamic(() => import('./HeavyComponent'), {
  loading: () => <Skeleton />,
  ssr: false, // Disable SSR for this component
});
```

**Image Optimization**:
```typescript
import Image from 'next/image';

<Image
  src="/hero.png"
  alt="Hero"
  width={1200}
  height={600}
  priority // Load immediately
/>
```

**Memoization**:
```typescript
const memoizedValue = useMemo(() => {
  return expensiveComputation(data);
}, [data]);

const memoizedCallback = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

## Development Workflow

### Running Locally

1. **Install dependencies**:
```bash
npm install
# or
pnpm install
```

2. **Set environment variables** (`.env.local`):
```bash
NEXT_PUBLIC_FASTAPI_BACKEND_URL=http://localhost:8000
```

3. **Run dev server**:
```bash
npm run dev
# or
pnpm dev
```

4. **Open browser**: http://localhost:3000

### Building for Production

```bash
npm run build
npm run start
```

### Linting

```bash
npm run lint
```

## Testing

Currently minimal test coverage. Recommended structure:

```
__tests__/
├── components/
│   ├── DocumentCard.test.tsx
│   └── SearchSpaces.test.tsx
├── hooks/
│   ├── useDocuments.test.ts
│   └── useAuth.test.ts
└── pages/
    └── researcher.test.tsx
```

**Example Test** (Jest + React Testing Library):
```typescript
import { render, screen } from '@testing-library/react';
import { DocumentCard } from '@/components/document-card';

describe('DocumentCard', () => {
  it('renders document title', () => {
    const doc = {
      id: 1,
      title: 'Test Document',
      content: 'Content',
      document_type: 'FILE',
      created_at: '2024-01-01',
      search_space_id: 1,
    };

    render(<DocumentCard document={doc} />);
    expect(screen.getByText('Test Document')).toBeInTheDocument();
  });
});
```

## Deployment

### Vercel (Recommended)

1. Connect GitHub repository
2. Configure environment variables
3. Deploy automatically on push

### Docker

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

EXPOSE 3000

CMD ["npm", "start"]
```

### Self-Hosted

```bash
npm run build
pm2 start npm --name surfsense -- start
```

## Browser Extension Integration

The web app communicates with the browser extension via:

1. **Extension saves page** → Posts to `/api/v1/documents/`
2. **Extension authenticates** → Shares token from web app

See [Extension Documentation](../extension/README.md) for details.

## Troubleshooting

**Issue**: API calls fail with CORS errors
**Solution**: Check backend CORS configuration, ensure correct API URL

**Issue**: Authentication fails after refresh
**Solution**: Token stored in localStorage. Check if token is expired or invalid.

**Issue**: Streaming chat not working
**Solution**: Ensure proper SSE handling, check browser console for errors

**Issue**: Build fails
**Solution**: Clear `.next` folder and rebuild. Check for TypeScript errors.

## Further Reading

- [Next.js Documentation](https://nextjs.org/docs)
- [React Documentation](https://react.dev)
- [Tailwind CSS](https://tailwindcss.com)
- [Shadcn/ui](https://ui.shadcn.com)
- [Vercel AI SDK](https://sdk.vercel.ai)
- [Architecture Documentation](../ARCHITECTURE.md)
- [API Reference](../API_REFERENCE.md)
