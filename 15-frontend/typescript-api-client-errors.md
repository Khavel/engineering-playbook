# TypeScript API Client with Custom Error Classes

## Problem
API calls need consistent error handling with type safety. Errors should carry status codes for different handling (401 = redirect to login, 429 = show retry message).

## Solution
Custom error class with type guard, typed request/response interfaces, and consistent fetch wrapper.

## Custom Error Class

`	ypescript
export class ApiError extends Error {
  constructor(
    public status: number,
    message: string
  ) {
    super(message);
    this.name = 'ApiError';
  }

  // Type guard for safe error handling
  static isApiError(error: unknown): error is ApiError {
    return error instanceof ApiError;
  }

  // Helper methods for common checks
  get isUnauthorized(): boolean {
    return this.status === 401;
  }

  get isRateLimited(): boolean {
    return this.status === 429;
  }

  get isNotFound(): boolean {
    return this.status === 404;
  }
}
`

## API Function Pattern

`	ypescript
import { API_BASE_URL } from '@/lib/config';

interface CreateFeedbackRequest {
  profileId: string;
  outcome: 'positive' | 'neutral' | 'caution';
  comment?: string;
}

interface FeedbackResponse {
  id: string;
  createdAt: string;
}

export async function createFeedback(
  token: string,
  request: CreateFeedbackRequest
): Promise<FeedbackResponse> {
  const response = await fetch(${API_BASE_URL}/api/feedback, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': Bearer ,
    },
    body: JSON.stringify(request),
  });

  const data = await response.json();

  if (!response.ok) {
    throw new ApiError(
      response.status,
      data.error || 'Failed to submit feedback'
    );
  }

  return data as FeedbackResponse;
}
`

## Component Usage

`	sx
import { ApiError, createFeedback } from '@/lib/api/feedback';
import { useRouter } from 'next/navigation';

function FeedbackForm() {
  const router = useRouter();
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (data: FormData) => {
    try {
      await createFeedback(token, {
        profileId: data.profileId,
        outcome: data.outcome,
      });
      router.push('/success');
    } catch (err) {
      if (ApiError.isApiError(err)) {
        if (err.isUnauthorized) {
          router.push('/login');
          return;
        }
        if (err.isRateLimited) {
          setError('Too many requests. Please wait and try again.');
          return;
        }
        setError(err.message);
      } else {
        setError('An unexpected error occurred');
      }
    }
  };
}
`

## Generic Fetch Helper (Optional)

`	ypescript
async function apiFetch<T>(
  endpoint: string,
  options: RequestInit = {}
): Promise<T> {
  const response = await fetch(${API_BASE_URL}, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });

  const data = await response.json();

  if (!response.ok) {
    throw new ApiError(response.status, data.error || 'Request failed');
  }

  return data as T;
}

// Usage
const profile = await apiFetch<ProfileResponse>('/api/profile/123');
`

## Benefits

- **Type safety**: Responses are typed, errors are structured
- **Consistent handling**: Same error pattern across all API calls
- **Status-aware**: Different UX for 401 vs 429 vs 500
- **Testable**: Easy to mock ApiError in tests
