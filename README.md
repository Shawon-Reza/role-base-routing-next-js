# role-base-routing-next-js
### 1. Central Route Config (Scalable Version)
```jsx
import type { Role } from './roles';

export type RoutePermission = {
  path: string;
  allowedRoles: Role[];
  // Optional: For more granular control in future
  requiredPermission?: string;
};

export const protectedRoutes: RoutePermission[] = [
  {
    path: '/dashboard',
    allowedRoles: ['user', 'manager', 'admin'],
  },
  {
    path: '/profile',
    allowedRoles: ['user', 'manager', 'admin'],
  },
  {
    path: '/settings',
    allowedRoles: ['user', 'manager'],           // user can access basic settings
  },
  {
    path: '/projects',
    allowedRoles: ['user', 'manager', 'admin'],
  },
  {
    path: '/admin',
    allowedRoles: ['admin'],
  },
  // Dynamic routes are handled via prefix matching in middleware
  {
    path: '/projects/',
    allowedRoles: ['user', 'manager', 'admin'],  // covers /projects/[id], /projects/new etc.
  },
];

// Helper to check if a path needs protection
export function getRoutePermissions(pathname: string): RoutePermission | undefined {
  // Exact match first
  let route = protectedRoutes.find(r => r.path === pathname);
  
  // Prefix match for dynamic/nested routes (e.g., /projects/123)
  if (!route) {
    route = protectedRoutes
      .filter(r => r.path.endsWith('/'))
      .find(r => pathname.startsWith(r.path));
  }
  
  return route;
}
```
### 2. Middleware (src/middleware.ts)
```jsx
import { auth } from '@/lib/auth';
import { getRoutePermissions } from '@/config/routeConfig';
import type { Role } from '@/config/roles';

export default auth((req) => {
  const { nextUrl } = req;
  const session = req.auth;

  // Skip public routes
  if (nextUrl.pathname.startsWith('/login') || 
      nextUrl.pathname.startsWith('/register') ||
      nextUrl.pathname === '/') {
    return;
  }

  if (!session?.user?.role) {
    return Response.redirect(new URL('/login', req.url));
  }

  const route = getRoutePermissions(nextUrl.pathname);

  if (route) {
    const userRole = session.user.role as Role;
    if (!route.allowedRoles.includes(userRole)) {
      return Response.redirect(new URL('/unauthorized', req.url));
    }
  }
});

export const config = {
  matcher: [
    '/dashboard/:path*',
    '/profile/:path*',
    '/settings/:path*',
    '/projects/:path*',
    '/admin/:path*',
  ],
};
```

# Way 2
### 1. Create the Proxy File


```jsx
// proxy.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { getSession } from '@/lib/auth'; // Your auth helper (Auth.js, custom JWT, etc.)

// Define protected routes and required roles
const roleBasedAccess: Record<string, string[]> = {
  '/admin': ['admin'],
  '/dashboard': ['admin', 'manager', 'user'],
  '/manager': ['admin', 'manager'],
  '/profile': ['admin', 'manager', 'user'],
  // Add more as needed
};

// Public routes (no auth required)
const publicRoutes = ['/', '/login', '/signup', '/api/auth'];

export async function proxy(request: NextRequest) {
  const url = request.nextUrl;
  const pathname = url.pathname;

  // Skip public routes and static assets
  if (
    publicRoutes.some((route) => pathname.startsWith(route)) ||
    pathname.startsWith('/_next') ||
    pathname.startsWith('/favicon') ||
    pathname.match(/\.(png|jpg|jpeg|gif|svg|ico|css|js)$/)
  ) {
    return NextResponse.next();
  }

  // Get user session (with role)
  const session = await getSession(request); // Implement this based on your auth

  if (!session?.user) {
    // Redirect unauthenticated users
    const loginUrl = new URL('/login', request.url);
    loginUrl.searchParams.set('callbackUrl', pathname);
    return NextResponse.redirect(loginUrl);
  }

  const userRole = session.user.role; // e.g., 'admin', 'manager', 'user'

  // Check RBAC for protected routes
  for (const [protectedPath, allowedRoles] of Object.entries(roleBasedAccess)) {
    if (pathname.startsWith(protectedPath)) {
      if (!allowedRoles.includes(userRole)) {
        // Forbidden - redirect or return 403
        return NextResponse.redirect(new URL('/unauthorized', request.url));
        // Or return JSON for API routes:
        // return NextResponse.json({ error: 'Forbidden: Insufficient permissions' }, { status: 403 });
      }
      break;
    }
  }

  // Optional: Add security headers or modify request
  const response = NextResponse.next();

  // Example: Add custom headers
  response.headers.set('X-User-Role', userRole);
  response.headers.set('X-Frame-Options', 'DENY');

  return response;
}

// Optional: Configure matcher to limit where proxy runs
export const config = {
  matcher: [
    /*
     * Match all request paths except:
     * - _next/static (static files)
     * - _next/image (image optimization files)
     * - favicon.ico, etc.
     */
    '/((?!_next/static|_next/image|favicon.ico).*)',
  ],
};
```

### 2. Create Session Helper (lib/auth.ts)
```jsx
// lib/auth.ts  (or wherever you handle auth)
import { cookies } from 'next/headers';
import { decrypt } from './session'; // Your JWT decrypt function

export async function getSession(request?: NextRequest) {
  const cookieStore = cookies();
  const sessionCookie = cookieStore.get('session')?.value; // or 'next-auth.session-token'

  if (!sessionCookie) return null;

  try {
    const session = await decrypt(sessionCookie);
    return session;
  } catch (error) {
    return null;
  }
}
```
