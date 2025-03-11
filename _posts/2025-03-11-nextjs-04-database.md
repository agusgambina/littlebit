---
layout: post
title:  "NextJS Migrating to Database Authentication with Prisma"
date:   2025-03-11 08:00:00 -0300
categories: react init
comments: true
---

# Migrating to Database Authentication with Prisma

1. [Install Dependencies](#install-dependencies)
2. [Initialize Prisma with SQLite](#initialize-prisma-with-sqlite)
3. [Define the User Model](#define-the-user-model)
4. [Update Environment Variables](#update-environment-variables)
5. [Generate the Prisma Client and Create DB](#generate-the-prisma-client-and-create-db)
6. [Create a User Service](#create-a-user-service)
7. [Create a DB Seed Script](#create-a-db-seed-script)
8. [Update the Login Route](#update-the-login-route)
9. [Creating a Prisma Client Instance](#creating-a-prisma-client-instance)

## Install dependencies

```
$ npm install @prisma/client bcrypt
$ npm install prisma --save-dev
$ npm i --save-dev @types/bcrypt
```

## Initialize Prisma with SQLite

```
$ npx prisma init --datasource-provider sqlite
```

From Prisma initialization

> 1. Set the DATABASE_URL in the .env file to point to your existing database. If your database has no tables yet, read https://pris.ly/d/getting-started
> 2. Run prisma db pull to turn your database schema into a Prisma schema.
> 3. Run prisma generate to generate the Prisma Client. You can then start querying your database.
> 4. Tip: Explore how you can extend the ORM with scalable connection pooling, global caching, and real-time database events. Read: https://pris.ly/cli/beyond-orm

## Define the User Model

Append to `prisma/schema.prisma`
```
...existing code...
model User {
  id        String   @id @default(uuid())
  username  String   @unique
  password  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

## Update Environment Variables

Append to the `.env` file

```
...existing code...
DATABASE_URL="file:./dev.db"
```

## Generate the Prisma Client and Create DB

```
$ npx prisma migrate dev --name init
```

From the previous command the output is something like

> SQLite database dev.db created at file:./dev.db
>
>Applying migration `20250311125033_init`
>
>The following migration(s) have been created and applied from new schema changes:
>
>migrations/
>  └─ 20250311125033_init/
>    └─ migration.sql
>
>Your database is now in sync with your schema.

## Create a User Service

Install

```
// src/lib/services/userService.ts
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcrypt';

const prisma = new PrismaClient();

export async function findUserByUsername(username: string) {
  return prisma.user.findUnique({
    where: { username },
  });
}

export async function verifyPassword(plainPassword: string, hashedPassword: string) {
  return bcrypt.compare(plainPassword, hashedPassword);
}

export async function createUser(username: string, password: string) {
  const hashedPassword = await bcrypt.hash(password, 10);
  
  return prisma.user.create({
    data: {
      username,
      password: hashedPassword,
    },
  });
} 
```

## Create a DB Seed Script

```
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcrypt';

const prisma = new PrismaClient();

async function main() {
  // Delete existing users to avoid duplicates
  await prisma.user.deleteMany({});

  const {
    SEED_ADMIN_USERNAME: seedAdminUsernameRaw,
    SEED_ADMIN_PASSWORD: seedAdminPasswordRaw,
  } = process.env;

  // Sanitize environment variables by removing quotes, semicolons and trimming whitespace
  const sanitizeEnvVar = (value: string | undefined) => 
    value?.replace(/['"]/g, '').replace(';', '').trim();

  const seedAdminUsername = sanitizeEnvVar(seedAdminUsernameRaw);
  const seedAdminPassword = sanitizeEnvVar(seedAdminPasswordRaw);

  if (!seedAdminUsername || !seedAdminPassword) {
    throw new Error('SEED_ADMIN_USERNAME or SEED_ADMIN_PASSWORD is not set');
  }

  // Create your default user
  const hashedPassword = await bcrypt.hash(seedAdminPassword, 10);

  const user = await prisma.user.create({
    data: {
      username: seedAdminUsername,
      password: hashedPassword,
    },
  });

  console.log({ user });
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

Add the seed script to your package.json:

```
...existing code...
  "prisma": {
    "seed": "ts-node --compiler-options {\"module\":\"CommonJS\"} prisma/seed.ts"
  }
...existing code...
```

Run the seed

```
$ npx prisma db seed
```

The output after running the seed

> Running seed command `ts-node --compiler-options {"module":"CommonJS"} prisma/seed.ts` ...
> {
>  user: {
>     id: 'f435f483-232c-48cd-8f7f-a2c13667a840',
>     username: 'admin',
>     password: '$2b$10$hWA1lLppobPUS2U.cduAgOHxJRfNjrzOTWZVfckx8/OL7lSvF3aVG',
>     createdAt: 2025-03-11T12:55:28.559Z,
>     updatedAt: 2025-03-11T12:55:28.559Z
>   }
> }
>
> 🌱  The seed command has been executed.

## Update the Login Route

```
// src/app/api/auth/login/route.ts
import { NextResponse } from 'next/server';
import jwt from 'jsonwebtoken';
import { JWT_SECRET, JWT_EXPIRATION } from '@/lib/config';
import { findUserByUsername, verifyPassword } from '@/lib/services/userService';

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

    // Find user in database
    const user = await findUserByUsername(username);
    
    // Check if user exists
    if (!user) {
      return NextResponse.json(
        { error: 'Invalid username or password' },
        { status: 401 }
      );
    }
    
    // Verify password
    const isValidPassword = await verifyPassword(password, user.password);
    
    if (!isValidPassword) {
      return NextResponse.json(
        { error: 'Invalid username or password' },
        { status: 401 }
      );
    }

    // Generate JWT token
    const token = jwt.sign(
      { 
        username: user.username,
        userId: user.id
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
        user: { 
          username: user.username,
          id: user.id 
        }
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
## Creating a Prisma Client Instance

For better performance, create a singleton Prisma client

```
// src/lib/prisma.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = global as unknown as { prisma: PrismaClient };

export const prisma =
  globalForPrisma.prisma ||
  new PrismaClient({
    log: ['query'],
  });

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma; 
```

Then update your userService:

```
// src/lib/services/userService.ts
import { prisma } from '@/lib/prisma';
import bcrypt from 'bcrypt';

export async function findUserByUsername(username: string) {
  return prisma.user.findUnique({
    where: { username },
  });
}

export async function verifyPassword(plainPassword: string, hashedPassword: string) {
  return bcrypt.compare(plainPassword, hashedPassword);
}

export async function createUser(username: string, password: string) {
  const hashedPassword = await bcrypt.hash(password, 10);
  
  return prisma.user.create({
    data: {
      username,
      password: hashedPassword,
    },
  });
} 
```