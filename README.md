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
