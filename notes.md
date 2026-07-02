# Project 2 - Prisma and NextAuth Notes

# Prisma

## What is Prisma?

Prisma is an Object Relational Mapping (ORM) tool for JavaScript and TypeScript.

It acts as a layer between your application and the database, allowing you to work with database records using JavaScript/TypeScript instead of writing raw SQL queries.

Instead of writing queries like:

```sql
SELECT * FROM users;
```

You can simply write:

```ts
const users = await prisma.user.findMany();
```

Prisma automatically converts this into the appropriate SQL query.

---

## What is an ORM?

ORM stands for **Object Relational Mapping**.

It maps database tables to JavaScript/TypeScript objects.

Instead of thinking in terms of SQL tables and joins, you work with familiar programming objects.

Example:

Database table

| id  | name | email          |
| --- | ---- | -------------- |
| 1   | Saad | saad@gmail.com |

Can be accessed as

```ts
const user = await prisma.user.findUnique({
  where: {
    id: 1,
  },
});

console.log(user.name);
```

No SQL is required.

---

# Why use an ORM?

## 1. Easier to write

Instead of SQL

```sql
SELECT * FROM posts WHERE published = true;
```

You write

```ts
const posts = await prisma.post.findMany({
  where: {
    published: true,
  },
});
```

This feels natural because you're writing JavaScript.

---

## 2. Type Safety

Prisma generates TypeScript types directly from your schema.

If a field doesn't exist, TypeScript immediately throws an error.

Example

```ts
await prisma.user.findMany({
  where: {
    emaill: "abc@gmail.com",
  },
});
```

This fails because `emaill` doesn't exist.

You catch mistakes before even running the application.

---

## 3. Protection against SQL Injection

Prisma automatically parameterizes queries.

Instead of manually concatenating strings like

```sql
SELECT * FROM users WHERE email = '${email}'
```

Prisma safely sends values to the database.

---

## 4. Less Boilerplate

CRUD operations become much shorter.

Creating

```ts
await prisma.user.create({
  data: {
    name: "Saad",
    email: "saad@gmail.com",
  },
});
```

Reading

```ts
await prisma.user.findMany();
```

Updating

```ts
await prisma.user.update({
  where: {
    id: "123",
  },
  data: {
    name: "New Name",
  },
});
```

Deleting

```ts
await prisma.user.delete({
  where: {
    id: "123",
  },
});
```

---

## 5. Database Schema Management

Prisma also manages your database structure.

Instead of manually creating tables, you define models inside

```
schema.prisma
```

Example

```prisma
model User {
    id    String @id @default(cuid())
    name  String?
    email String @unique
}
```

Then run

```bash
npx prisma migrate dev
```

Prisma creates the SQL migration and updates the database automatically.

---

# Basic Prisma Workflow

## Step 1

Create models inside

```
prisma/schema.prisma
```

Example

```prisma
model User {
    id    String @id @default(cuid())
    email String @unique
    name  String?
}
```

---

## Step 2

Generate migration

```bash
npx prisma migrate dev
```

This

- Creates SQL migrations
- Creates database tables
- Updates Prisma Client

---

## Step 3

Generate Prisma Client

```bash
npx prisma generate
```

This generates the TypeScript client used throughout the project.

---

## Step 4

Use Prisma inside your application

```ts
import { prisma } from "@/lib/prisma";

const users = await prisma.user.findMany();
```

---

# Setting up Prisma in Next.js

Installation

```bash
npm install prisma @prisma/client
```

Initialize Prisma

```bash
npx prisma init
```

This creates

```
prisma/
    schema.prisma

.env
```

Inside `.env`

```env
DATABASE_URL="..."
```

After defining models

```bash
npx prisma migrate dev
```

---

# Why do we create lib/prisma.ts?

In development mode, Next.js reloads modules frequently.

Without precautions, every reload creates a new Prisma Client instance.

Eventually you'll see

```
Too many Prisma Clients are already running.
```

To avoid this, we create a singleton.

Example

```ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as {
  prisma?: PrismaClient;
};

export const prisma = globalForPrisma.prisma ?? new PrismaClient();

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma;
}
```

Now the same Prisma Client instance is reused.

---

# NextAuth

## What is NextAuth?

NextAuth (now called Auth.js) is an authentication library built specifically for Next.js.

It handles

- Login
- Logout
- Sessions
- Cookies
- OAuth
- JWT
- Database integration

without having to build authentication from scratch.

---

# Why use NextAuth?

Without NextAuth you would have to implement

- OAuth flow
- Token verification
- Cookie management
- Session management
- CSRF protection
- Password hashing
- Login pages

yourself.

NextAuth provides all of these.

---

# OAuth

OAuth lets users log in using existing accounts instead of creating new passwords.

Examples

- GitHub
- Google
- Discord
- Microsoft
- Facebook

Flow

```
User

↓

Clicks "Continue with GitHub"

↓

Redirected to GitHub

↓

Logs into GitHub

↓

GitHub verifies identity

↓

GitHub sends user information

↓

NextAuth creates session

↓

User is logged in
```

The application never sees the user's GitHub password.

---

# Installing NextAuth

```bash
npm install next-auth
npm install @auth/prisma-adapter
```

---

# Environment Variables

Example

```env
AUTH_SECRET=your_generated_secret

AUTH_GITHUB_ID=...

AUTH_GITHUB_SECRET=...
```

The GitHub Client ID and Secret come from your GitHub OAuth App.

---

# Generating AUTH_SECRET

Generate a secure random secret

```bash
npx auth secret
```

This automatically generates something like

```env
AUTH_SECRET=long_random_secret
```

The secret is used to

- Sign JWTs
- Encrypt cookies
- Protect sessions

Never commit it to GitHub.

---

# auth.ts

This is the central configuration file for authentication.

Example

```ts
import NextAuth from "next-auth";
import Github from "next-auth/providers/github";
import { PrismaAdapter } from "@auth/prisma-adapter";
import { prisma } from "@/lib/prisma";

export const { auth, handlers, signIn, signOut } = NextAuth({
  session: {
    strategy: "jwt",
  },

  providers: [Github],

  adapter: PrismaAdapter(prisma),

  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.id = user.id;
        token.name = user.name;
      }

      return token;
    },

    async session({ session, token }) {
      if (session.user) {
        session.user.id = token.id as string;
        session.user.name = token.name as string;
      }

      return session;
    },
  },
});
```

This file controls the entire authentication system.

---

# Providers

```ts
providers: [Github];
```

A provider is simply a login method.

Examples

```ts
Github;
Google;
Discord;
Microsoft;
```

You can add multiple providers.

---

# Prisma Adapter

```ts
adapter: PrismaAdapter(prisma);
```

The adapter connects NextAuth to Prisma.

Instead of storing users only in memory, it automatically stores data inside your database.

Whenever someone signs in

NextAuth automatically

- Creates a User
- Creates an Account
- Creates Sessions (if using database sessions)

No manual SQL is required.

---

# Why are extra Prisma models created?

Installing the Prisma Adapter requires several models.

Example

```
User

Account

Session

VerificationToken
```

Each has a purpose.

## User

Stores user information.

```
id
name
email
image
```

---

## Account

Stores OAuth account information.

Example

```
GitHub Account

Google Account

Discord Account
```

A user can connect multiple providers.

---

## Session

Used only if using database sessions.

Since this project uses JWT

```ts
strategy: "jwt";
```

the Session table may not actually be used.

---

## VerificationToken

Used for email login.

---

# Session Strategy

```ts
session: {
  strategy: "jwt";
}
```

Instead of saving sessions inside the database,

NextAuth stores them inside signed JWT tokens.

Advantages

- Faster
- No session database lookup
- Stateless

---

# JWT Callback

```ts
async jwt({ token, user }) {

    if (user) {
        token.id = user.id;
        token.name = user.name;
    }

    return token;
}
```

Runs whenever a JWT is created or updated.

Purpose

Add custom fields into the token.

Without this callback

```
token.id
```

would not exist.

---

# Session Callback

```ts
async session({ session, token }) {

    if (session.user) {

        session.user.id = token.id as string;

        session.user.name = token.name as string;

    }

    return session;
}
```

Runs whenever the client requests the current session.

Copies custom JWT fields into

```
session.user
```

Now inside React

```ts
const session = await auth();

console.log(session?.user.id);
```

works correctly.

---

# middleware.ts

```ts
export { auth as middleware } from "@/auth";
```

Middleware runs before requests reach your pages.

It checks whether a user is authenticated.

You can use it to protect routes.

Flow

```
User requests page

↓

Middleware executes

↓

Checks authentication

↓

Authenticated

↓

Allow request

OR

Redirect to login
```

---

# API Route

```
app/api/auth/[...nextauth]/route.ts
```

Contents

```ts
import { handlers } from "@/auth";

export const { GET, POST } = handlers;
```

This creates all authentication API endpoints.

Examples

```
/api/auth/signin

/api/auth/signout

/api/auth/session

/api/auth/callback/github
```

You don't implement these routes manually.

NextAuth handles them.

---

# Helper Functions

```
lib/auth.ts
```

```ts
import { signIn, signOut } from "@/auth";

export const login = async () => {
  await signIn("github", {
    redirectTo: "/",
  });
};

export const logout = async () => {
  await signOut({
    redirectTo: "/auth/signin",
  });
};
```

These helper functions make authentication cleaner.

Instead of writing

```ts
await signIn("github");
```

everywhere,

you simply call

```ts
await login();
```

---

# Overall Authentication Flow

```
User clicks Login

↓

login()

↓

signIn("github")

↓

Redirect to GitHub

↓

GitHub authenticates user

↓

GitHub returns OAuth token

↓

NextAuth receives user information

↓

Prisma Adapter stores user inside database

↓

JWT callback runs

↓

JWT is generated

↓

Session callback runs

↓

Browser receives session cookie

↓

User is authenticated
```

---

# Folder Structure

```
app/
│
├── api/
│   └── auth/
│       └── [...nextauth]/
│           └── route.ts
│
├── auth/
│
├── page.tsx
│
lib/
│   ├── auth.ts
│   └── prisma.ts
│
prisma/
│   └── schema.prisma
│
auth.ts
│
middleware.ts
│
.env
```

---

# Summary

Prisma

- ORM for JavaScript and TypeScript
- Type-safe database queries
- Generates database client automatically
- Manages database schema through migrations
- Eliminates most handwritten SQL

NextAuth

- Authentication library for Next.js
- Supports OAuth providers like GitHub and Google
- Handles login, logout, sessions, cookies and JWTs
- Uses Prisma Adapter to persist users in the database
- Middleware protects routes
- API route exposes authentication endpoints
- `auth.ts` is the central configuration for authentication
- JWT callbacks allow custom data (such as user ID) to be included in tokens and sessions
