---
layout: post
title:  "NextJS Server Login"
date:   2025-03-10 21:00:00 -0300
categories: react init
comments: true
---

# Server Login

While a login feature was previously implemented on the client-side, we are now migrating the authentication logic to the server-side for enhanced security and a more robust architecture.

Here's what I'll do

1. [Create API routes for authentication](#create-api-routes-for-authentication)
2. [Update the auth library to use API calls instead of client-side validation](#update-the-auth-library-to-use-api-calls-instead-of-client-side-validation)
3. [Implement JWT token-based authentication](#implement-jwt-token-based-authentication)
4. [Update the middleware to validate JWT tokens](#update-the-auth-library-to-use-api-calls-instead-of-client-side-validation)

# Create API routes for authentication

Install necessary dependencies for JWT authentication:

```
$ npm install jsonwebtoken cookies-next
```

Install the type declarations for jsonwebtoken

```
$ npm install --save-dev @types/jsonwebtoken
```

Create a configuration file for our JWT secret

```
// src/lib/config.ts
// Configuration settings for the application

// JWT settings
export const JWT_SECRET = process.env.JWT_SECRET || 'pokemon-challenge-default-jwt-secret';
export const JWT_EXPIRATION = '24h';  // Token expiration time

// Hardcoded credentials for development - in production, use a database
export const VALID_USERNAME = 'admin';
export const VALID_PASSWORD = 'admin'; 
```

# Update the auth library to use API calls instead of client-side validation

Create our API endpoints for authentication:

```
// src/app/api/auth/login/route.ts
import { NextResponse } from 'next/server';
import jwt from 'jsonwebtoken';
import { VALID_USERNAME, VALID_PASSWORD, JWT_SECRET, JWT_EXPIRATION } from '@/lib/config';

// Set runtime to Node.js to support the crypto module used by jsonwebtoken
export const runtime = 'nodejs';

export async function POST(request: Request) {
  try {
    // Parse request body
    const body = await request.json();
    const { username, password } = body;

    // Validate required fields
    if (!username || !password) {
      return NextResponse.json(
        { error: 'Username and password are required' },
        { status: 400 }
      );
    }

    // Validate credentials
    // In a real app, you would check against a database
    if (username !== VALID_USERNAME || password !== VALID_PASSWORD) {
      return NextResponse.json(
        { error: 'Invalid username or password' },
        { status: 401 }
      );
    }

    // Generate JWT token
    const token = jwt.sign(
      { 
        username,
        // Add any additional user data you want in the token
      },
      JWT_SECRET,
      { expiresIn: JWT_EXPIRATION }
    );

    // Return success response with token
    return NextResponse.json(
      { 
        success: true,
        token,
        user: { username }
      },
      { 
        status: 200,
        headers: {
          'Set-Cookie': `token=${token}; Path=/; HttpOnly; SameSite=Strict; Max-Age=${60 * 60 * 24}` // 24 hours
        }
      }
    );
  } catch (error) {
    console.error('Login error:', error);
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
} 
```

Create a logout endpoint as well

```
// src/app/api/auth/logout/route.ts
import { NextResponse } from 'next/server';

// Set runtime to Node.js for consistency with other auth routes
export const runtime = 'nodejs';

export async function POST() {
  try {
    // Clear the auth token by setting an expired cookie
    return NextResponse.json(
      { success: true },
      {
        status: 200,
        headers: {
          'Set-Cookie': `token=; Path=/; HttpOnly; SameSite=Strict; Max-Age=0`
        }
      }
    );
  } catch (error) {
    console.error('Logout error:', error);
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
} 
```

# Implement JWT token-based authentication

Create an endpoint to check the current user

```
// src/app/api/auth/user/route.ts
import { NextResponse } from 'next/server';
import { cookies } from 'next/headers';
import jwt from 'jsonwebtoken';
import { JWT_SECRET } from '@/lib/config';

// Set runtime to Node.js to support the crypto module used by jsonwebtoken
export const runtime = 'nodejs';

export async function GET() {
  try {
    // Get the token from cookies
    const cookieStore = await cookies();
    const token = cookieStore.get('token')?.value;

    if (!token) {
      return NextResponse.json(
        { user: null },
        { status: 401 }
      );
    }

    try {
      // Verify and decode the JWT token
      const decoded = jwt.verify(token, JWT_SECRET) as { username: string };
      
      // Return the user data
      return NextResponse.json(
        { 
          user: { 
            username: decoded.username
          } 
        },
        { status: 200 }
      );
    } catch (jwtError) {
      // Invalid token
      console.error('JWT verification error:', jwtError);
      return NextResponse.json(
        { user: null },
        { status: 401 }
      );
    }
  } catch (error) {
    console.error('Get user error:', error);
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
} 
```

4. Update the middleware to validate JWT tokens

Update our middleware to properly check for JWT authentication

```
// src/middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { JWT_SECRET } from './lib/config';
import * as jose from 'jose';

export async function middleware(request: NextRequest) {
  // Skip middleware for login page, public assets, and API routes
  if (
    request.nextUrl.pathname.startsWith('/login') ||
    request.nextUrl.pathname.startsWith('/_next') ||
    request.nextUrl.pathname.startsWith('/api') ||
    request.nextUrl.pathname.startsWith('/favicon.ico')
  ) {
    return NextResponse.next();
  }

  try {
    // Get auth token from cookies
    const token = request.cookies.get('token')?.value;

    // If no token exists, redirect to login
    if (!token) {
      const url = new URL('/login', request.url);
      return NextResponse.redirect(url);
    }

    // For middleware, we'll need to use jose instead of jsonwebtoken
    // since middleware runs in the Edge Runtime which doesn't support
    // Node.js crypto module that jsonwebtoken depends on
    try {
      // We'll just check if the JWT is valid, not extracting the payload here
      // This is a simple verification that the token is properly signed
      // Convert string to Uint8Array for jose
      const secretKey = new TextEncoder().encode(JWT_SECRET);
      await jose.jwtVerify(token, secretKey);
      
      // Token is valid, continue
      return NextResponse.next();
    } catch (jwtError) {
      // Token is invalid, redirect to login
      console.error('Invalid token:', jwtError);
      const url = new URL('/login', request.url);
      return NextResponse.redirect(url);
    }
  } catch (error) {
    console.error('Auth middleware error:', error);
    // On error, redirect to login as a fallback
    const url = new URL('/login', request.url);
    return NextResponse.redirect(url);
  }
}

export const config = {
  matcher: [
    /*
     * Match all request paths except for:
     * - _next/static (static files)
     * - _next/image (image optimization files)
     * - favicon.ico (favicon file)
     * - public directory
     */
    '/((?!_next/static|_next/image|favicon.ico|public).*)',
  ],
}; 
```

Update the auth.ts library to use our new backend API endpoints

```
// src/lib/auth.ts
// Auth utilities for API-based authentication
import { setCookie, deleteCookie } from 'cookies-next';

interface User {
  username: string;
}

// Hardcoded credentials for development
const VALID_USERNAME = 'admin';
const VALID_PASSWORD = 'admin';

// Check if credentials are valid
export function validateCredentials(username: string, password: string): boolean {
  return username === VALID_USERNAME && password === VALID_PASSWORD;
}

// Get the current user by checking with the API
export async function getCurrentUser(): Promise<User | null> {
  try {
    const response = await fetch('/api/auth/user', {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
      },
      credentials: 'include', // Include cookies in the request
    });

    if (!response.ok) {
      return null;
    }

    const data = await response.json();
    return data.user;
  } catch (error) {
    console.error('Error getting current user:', error);
    return null;
  }
}

// Login the user via API
export async function login(username: string, password: string): Promise<boolean> {
  try {
    // Call login API endpoint
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ username, password }),
      credentials: 'include', // Include cookies in the response
    });

    if (!response.ok) {
      return false;
    }

    const data = await response.json();
    
    // Store token in client-side cookie as a backup (this will be set by the server too)
    // This is just for redundancy
    if (data.token) {
      setCookie('token', data.token, { 
        path: '/',
        maxAge: 60 * 60 * 24, // 1 day in seconds
        sameSite: 'strict' 
      });
    }
    
    return true;
  } catch (error) {
    console.error('Login error:', error);
    return false;
  }
}

// Logout the user via API
export async function logout(): Promise<void> {
  try {
    // Call logout API endpoint
    await fetch('/api/auth/logout', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      credentials: 'include', // Include cookies in the request
    });
    
    // Delete the token from client-side cookies (redundant, as the server will do this too)
    deleteCookie('token', { path: '/' });
  } catch (error) {
    console.error('Logout error:', error);
  }
} 
```

Update the AuthProvider to use the async auth functions

```
// src/components/AuthProvider.tsx
'use client';

import { createContext, useContext, useEffect, useState, ReactNode } from 'react';
import { useRouter } from 'next/navigation';
import { getCurrentUser, login as loginUser, logout as logoutUser } from '@/lib/auth';

interface User {
  username: string;
}

interface AuthContextType {
  user: User | null;
  login: (username: string, password: string) => Promise<boolean>;
  logout: () => Promise<void>;
  loading: boolean;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const router = useRouter();

  useEffect(() => {
    // Check if user is logged in on component mount
    const checkAuth = async () => {
      try {
        const currentUser = await getCurrentUser();
        setUser(currentUser);
      } catch (error) {
        console.error('Auth check error:', error);
      } finally {
        setLoading(false);
      }
    };
    
    checkAuth();
  }, []);

  const login = async (username: string, password: string): Promise<boolean> => {
    try {
      const success = await loginUser(username, password);
      if (success) {
        setUser({ username });
        router.push('/');
      }
      return success;
    } catch (error) {
      console.error('Login error in context:', error);
      return false;
    }
  };

  const logout = async (): Promise<void> => {
    try {
      await logoutUser();
      setUser(null);
      router.push('/login');
    } catch (error) {
      console.error('Logout error in context:', error);
    }
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
} 
```

Update the login page to handle async login

```
// src/app/login/page.tsx
'use client';

import { useState, useEffect } from 'react';
import { useAuth } from '@/components/AuthProvider';
import { useRouter } from 'next/navigation';
import Image from 'next/image';

export default function Login() {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const { login, user, loading } = useAuth();
  const router = useRouter();

  // Move the redirect logic to useEffect instead of during render
  useEffect(() => {
    // Redirect if already logged in
    if (!loading && user) {
      router.push('/');
    }
  }, [user, loading, router]);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');
    
    if (!username || !password) {
      setError('Username and password are required');
      return;
    }
    
    setIsLoading(true);
    
    try {
      const success = await login(username, password);
      if (!success) {
        setError('Invalid username or password');
      }
    } catch (error) {
      console.error('Login error:', error);
      setError('An error occurred. Please try again.');
    } finally {
      setIsLoading(false);
    }
  };

  // Remove the conditional rendering with router.push()
  // Just render the login form if we reach this point
  return (
    <div className="min-h-screen flex items-center justify-center p-8">
      <div className="w-full max-w-md bg-white dark:bg-black/20 p-8 rounded-lg shadow">
        <div className="flex justify-center mb-8">
          <Image
            className="dark:invert"
            src="/next.svg"
            alt="Next.js logo"
            width={120}
            height={25}
            priority
          />
        </div>
        
        <h1 className="text-2xl font-bold text-center mb-6">Login</h1>
        
        {error && (
          <div className="bg-red-100 dark:bg-red-900/20 text-red-700 dark:text-red-300 p-3 rounded mb-4">
            {error}
          </div>
        )}
        
        <form onSubmit={handleSubmit}>
          <div className="mb-4">
            <label className="block text-sm font-medium mb-2" htmlFor="username">
              Username
            </label>
            <input
              id="username"
              type="text"
              value={username}
              onChange={(e) => setUsername(e.target.value)}
              className="w-full p-3 border rounded focus:outline-none focus:ring-2 focus:ring-blue-500 dark:bg-gray-800 dark:border-gray-700"
              disabled={isLoading}
            />
          </div>
          
          <div className="mb-6">
            <label className="block text-sm font-medium mb-2" htmlFor="password">
              Password
            </label>
            <input
              id="password"
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              className="w-full p-3 border rounded focus:outline-none focus:ring-2 focus:ring-blue-500 dark:bg-gray-800 dark:border-gray-700"
              disabled={isLoading}
            />
          </div>
          
          <button
            type="submit"
            className="w-full py-3 px-4 bg-foreground text-background rounded-full hover:bg-[#383838] dark:hover:bg-[#ccc] transition-colors"
            disabled={isLoading}
          >
            {isLoading ? 'Logging in...' : 'Login'}
          </button>
        </form>
      </div>
    </div>
  );
} 
```

Create a .env.local file to store our secret

```
# .env.local
JWT_SECRET=pokemon-challenge-production-secret 
```

Add `.env` files to gitignore

```
# .gitignore
...
.env
.env.*
```