# Authentication Guide for Next.js App Router with Auth.js and Mongoose

This guide provides a step-by-step approach to implementing authentication in a Next.js (App Router) application using Auth.js (NextAuth) with Mongoose for database operations and support for Credentials, Google, and GitHub providers. The guide is based on the provided codebase, with optimizations and best practices for a robust authentication flow.

## Table of Contents
1. [Project Setup](#project-setup)
2. [Environment Variables](#environment-variables)
3. [Database Configuration](#database-configuration)
4. [Mongoose User Model](#mongoose-user-model)
5. [Auth.js Configuration](#authjs-configuration)
6. [Middleware Setup](#middleware-setup)
7. [Session Management](#session-management)
8. [User Actions (Register/Login)](#user-actions-registerlogin)
9. [Login Page Implementation](#login-page-implementation)
10. [Registration Page Implementation](#registration-page-implementation)
11. [Authentication Flow](#authentication-flow)
    - [Register with Credentials](#register-with-credentials)
    - [Login with Credentials](#login-with-credentials)
    - [Login with Google](#login-with-google)
    - [Login with GitHub](#login-with-github)
12. [Best Practices and Optimizations](#best-practices-and-optimizations)
13. [Troubleshooting](#troubleshooting)

---

## Project Setup
Ensure your Next.js project is set up with the following dependencies installed:

```bash
npm install next-auth mongoose bcryptjs clsx tailwind-merge @tabler/icons-react
```

- `next-auth`: For authentication.
- `mongoose`: For MongoDB integration.
- `bcryptjs`: For password hashing.
- `clsx` and `tailwind-merge`: For class name utilities.
- `@tabler/icons-react`: For GitHub/Google icons.

Create the following file structure:
```
├── lib/
│   ├── db.ts
│   ├── getSession.ts
│   ├── utils.ts
├── models/
│   ├── User.ts
├── app/
│   ├── login/
│   │   ├── page.tsx
│   ├── register/
│   │   ├── page.tsx
│   ├── middleware.ts
│   ├── api/auth/[...nextauth]/
│   │   ├── route.ts
├── auth.ts
```

---

## Environment Variables
Create a `.env.local` file in the root directory to store environment variables for MongoDB and OAuth providers.

**`.env.local`**
```env
MONGO_URI=mongodb://localhost:27017/your-database-name
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-secret-key-here
GITHUB_CLIENT_ID=your-github-client-id
GITHUB_CLIENT_SECRET=your-github-client-secret
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
```

- `MONGO_URI`: MongoDB connection string.
- `NEXTAUTH_URL`: Base URL of your app.
- `NEXTAUTH_SECRET`: Secret for signing JWT tokens (generate a strong random key).
- Obtain `GITHUB_CLIENT_ID` and `GITHUB_CLIENT_SECRET` from GitHub OAuth Apps.
- Obtain `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` from Google Cloud Console.

---

## Database Configuration
Set up MongoDB connection using Mongoose.

**`lib/db.ts`**
```typescript
import mongoose from "mongoose";

const connectDB = async () => {
  try {
    if (mongoose.connection.readyState >= 1) {
      console.log("Already connected to MongoDB");
      return;
    }
    await mongoose.connect(process.env.MONGO_URI!, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log("Successfully connected to MongoDB 🥂");
  } catch (error: any) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

export default connectDB;
```

- Checks for existing connection to avoid reconnecting.
- Uses modern Mongoose connection options.

---

## Mongoose User Model
Define the User schema for MongoDB.

**`models/User.ts`**
```typescript
import mongoose from "mongoose";

const userSchema = new mongoose.Schema({
  firstName: { type: String, required: true },
  lastName: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, select: false }, // Hidden by default
  role: { type: String, default: "user" },
  image: { type: String },
  authProviderId: { type: String }, // For OAuth providers
  createdAt: { type: Date, default: Date.now },
});

export const User = mongoose.models?.User || mongoose.model("User", userSchema);
```

- `email` is unique to prevent duplicate accounts.
- `password` is hidden by default (`select: false`) for security.
- `authProviderId` stores OAuth provider IDs (e.g., Google or GitHub user ID).

---

## Auth.js Configuration
Configure Auth.js with Credentials, Google, and GitHub providers.

**`auth.ts`**
```typescript
import NextAuth, { CredentialsSignin } from "next-auth";
import Credentials from "next-auth/providers/credentials";
import Github from "next-auth/providers/github";
import Google from "next-auth/providers/google";
import connectDB from "./lib/db";
import { User } from "./models/User";
import { compare } from "bcryptjs";

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    Github({
      clientId: process.env.GITHUB_CLIENT_ID,
      clientSecret: process.env.GITHUB_CLIENT_SECRET,
    }),
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    }),
    Credentials({
      name: "Credentials",
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      authorize: async (credentials) => {
        const email = credentials.email as string | undefined;
        const password = credentials.password as string | undefined;

        if (!email || !password) {
          throw new CredentialsSignin("Please provide both email and password");
        }

        await connectDB();
        const user = await User.findOne({ email }).select("+password +role");

        if (!user || !user.password) {
          throw new Error("Invalid email or password");
        }

        const isMatched = await compare(password, user.password);
        if (!isMatched) {
          throw new Error("Invalid email or password");
        }

        return {
          id: user._id.toString(),
          email: user.email,
          firstName: user.firstName,
          lastName: user.lastName,
          role: user.role,
        };
      },
    }),
  ],
  pages: {
    signIn: "/login",
    error: "/error",
  },
  callbacks: {
    async session({ session, token }) {
      if (token?.sub && token?.role) {
        session.user.id = token.sub;
        session.user.role = token.role;
      }
      return session;
    },
    async jwt({ token, user }) {
      if (user) {
        token.sub = user.id;
        token.role = user.role;
      }
      return token;
    },
    async signIn({ user, account }) {
      await connectDB();
      if (account?.provider === "google" || account?.provider === "github") {
        const { email, id } = user;
        const name = user.name?.split(" ") || ["", ""];
        const firstName = name[0] || "Unknown";
        const lastName = name[1] || "User";
        const image = user.image || "";

        const existingUser = await User.findOne({ email });
        if (!existingUser) {
          await User.create({
            email,
            firstName,
            lastName,
            image,
            authProviderId: id,
            role: "user",
          });
        } else if (!existingUser.authProviderId) {
          await User.updateOne({ email }, { authProviderId: id });
        }
      }
      return true;
    },
  },
  session: {
    strategy: "jwt",
  },
});
```

- **Providers**: Configures GitHub, Google, and Credentials providers.
- **Credentials Provider**: Validates email/password against MongoDB.
- **Callbacks**:
  - `session`: Adds user ID and role to session.
  - `jwt`: Stores user ID and role in JWT.
  - `signIn`: Handles OAuth user creation/update in MongoDB.
- **Pages**: Custom login and error pages.
- **Session**: Uses JWT strategy for session management.

---

## Middleware Setup
Protect routes using Auth.js middleware.

**`root/middleware.ts`**
```typescript
export { auth as middleware } from "@/auth";
```

- Applies Auth.js middleware to protect routes.
- Add specific routes to protect in `matcher` if needed (e.g., `matcher: ["/dashboard"]`).

---

## Session Management
Cache session data for performance.

**`lib/getSession.ts`**
```typescript
import { auth } from "@/auth";
import { cache } from "react";

export const getSession = cache(async () => {
  const session = await auth();
  return session;
});
```

- Uses React's `cache` to memoize session data.
- Ensures session is fetched efficiently across components.

---

## User Actions (Register/Login)
Define server actions for registration and login.

**`lib/user.ts`**
```typescript
"use server";

import connectDB from "@/lib/db";
import { User } from "@/models/User";
import { redirect } from "next/navigation";
import { hash } from "bcryptjs";
import { signIn } from "@/auth";

export const login = async (formData: FormData) => {
  const email = formData.get("email") as string;
  const password = formData.get("password") as string;

  try {
    await signIn("credentials", {
      email,
      password,
      redirect: false,
    });
  } catch (error) {
    throw new Error("Invalid email or password");
  }
  redirect("/");
};

export const register = async (formData: FormData) => {
  const firstName = formData.get("firstName") as string;
  const lastName = formData.get("lastName") as string;
  const email = formData.get("email") as string;
  const password = formData.get("password") as string;

  if (!firstName || !lastName || !email || !password) {
    throw new Error("Please fill all fields");
  }

  await connectDB();
  const existingUser = await User.findOne({ email });
  if (existingUser) {
    throw new Error("User already exists");
  }

  const hashedPassword = await hash(password, 12);
  await User.create({
    firstName,
    lastName,
    email,
    password: hashedPassword,
    role: "user",
  });

  redirect("/login");
};

export const fetchAllUsers = async () => {
  await connectDB();
  const users = await User.find({});
  return users.map((user) => ({
    id: user._id.toString(),
    firstName: user.firstName,
    lastName: user.lastName,
    email: user.email,
    role: user.role,
  }));
};
```

- **login**: Uses Auth.js `signIn` for credentials-based login.
- **register**: Validates input, hashes password, and creates a user.
- **fetchAllUsers**: Utility function to retrieve all users (for admin purposes).

---

## Login Page Implementation
Create a login page with credentials and OAuth options.

**`app/login/page.tsx`**
```typescript
import { login } from "@/lib/user";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { IconBrandGithub, IconBrandGoogle } from "@tabler/icons-react";
import { signIn } from "@/auth";
import Link from "next/link";
import { redirect } from "next/navigation";
import { getSession } from "@/lib/getSession";

const Login = async () => {
  const session = await getSession();
  if (session?.user) redirect("/");

  return (
    <div className="mt-10 max-w-md w-full mx-auto rounded-none md:rounded-2xl p-4 md:p-8 shadow-input bg-white border border-[#121212] dark:bg-black">
      <h2 className="font-bold text-xl text-neutral-800 dark:text-neutral-200">
        Login
      </h2>
      <form className="my-8" action={login}>
        <div className="mb-4">
          <Label htmlFor="email">Email Address</Label>
          <Input
            id="email"
            placeholder="projectmayhem@fc.com"
            type="email"
            name="email"
            className="mt-1"
          />
        </div>
        <div className="mb-6">
          <Label htmlFor="password">Password</Label>
          <Input
            id="password"
            placeholder="*************"
            type="password"
            name="password"
            className="mt-1"
          />
        </div>
        <button
          className="bg-gradient-to-br from-black to-neutral-600 block w-full text-white rounded-md h-10 font-medium shadow-[0px_1px_0px_0px_#ffffff40_inset,0px_-1px_0px_0px_#ffffff40_inset] dark:shadow-[0px_1px_0px_0px_var(--zinc-800)_inset,0px_-1px_0px_0px_var(--zinc-800)_inset]"
          type="submit"
        >
          Login &rarr;
        </button>
        <p className="text-right text-neutral-600 text-sm mt-4 dark:text-neutral-300">
          Don't have an account? <Link href="/register">Register</Link>
        </p>
      </form>
      <div className="bg-gradient-to-r from-transparent via-neutral-300 dark:via-neutral-700 to-transparent my-8 h-[1px] w-full" />
      <form
        action={async () => {
          "use server";
          await signIn("github", { redirectTo: "/" });
        }}
        className="mb-4"
      >
        <button
          className="flex space-x-2 items-center justify-center w-full text-black rounded-md h-10 font-medium shadow-input bg-gray-50 dark:bg-zinc-900 dark:shadow-[0px_0px_1px_1px_var(--neutral-800)]"
          type="submit"
        >
          <IconBrandGithub className="h-4 w-4 text-neutral-800 dark:text-neutral-300" />
          <span className="text-neutral-700 dark:text-neutral-300 text-sm">
            Sign in with GitHub
          </span>
        </button>
      </form>
      <form
        action={async () => {
          "use server";
          await signIn("google", { redirectTo: "/" });
        }}
      >
        <button
          className="flex space-x-2 items-center justify-center w-full text-black rounded-md h-10 font-medium shadow-input bg-gray-50 dark:bg-zinc-900 dark:shadow-[0px_0px_1px_1px_var(--neutral-800)]"
          type="submit"
        >
          <IconBrandGoogle className="h-4 w-4 text-neutral-800 dark:text-neutral-300" />
          <span className="text-neutral-700 dark:text-neutral-300 text-sm">
            Sign in with Google
          </span>
        </button>
      </form>
    </div>
  );
};

export default Login;
```

- Redirects authenticated users to the homepage.
- Supports credentials login and OAuth (GitHub/Google).
- Uses server actions for OAuth sign-in with redirect.

---

## Registration Page Implementation
Create a registration page for credentials-based signup.

**`app/register/page.tsx`**
```typescript
import { register } from "@/lib/user";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import Link tolerances="next/link";
import { redirect } from "next/navigation";
import { getSession } from "@/lib/getSession";

const Register = async () => {
  const session = await getSession();
  if (session?.user) redirect("/");

  return (
    <div className="mt-10 max-w-md w-full mx-auto rounded-none md:rounded-2xl p-4 md:p-8 shadow-input bg-white border border-[#121212] dark:bg-black">
      <h2 className="font-bold text-xl text-neutral-800 dark:text-neutral-200">
        Register
      </h2>
      <form className="my-8" action={register}>
        <div className="mb-4">
          <Label htmlFor="firstName">First Name</Label>
          <Input
            id="firstName"
            placeholder="Tyler"
            type="text"
            name="firstName"
            className="mt-1"
          />
        </div>
        <div className="mb-4">
          <Label htmlFor="lastName">Last Name</Label>
          <Input
            id="lastName"
            placeholder="Durden"
            type="text"
            name="lastName"
            className="mt-1"
          />
        </div>
        <div className="mb-4">
          <Label htmlFor="email">Email Address</Label>
          <Input
            id="email"
            placeholder="projectmayhem@fc.com"
            type="email"
            name="email"
            className="mt-1"
          />
        </div>
        <div className="mb-6">
          <Label htmlFor="password">Password</Label>
          <Input
            id="password"
            placeholder="*************"
            type="password"
            name="password"
            className="mt-1"
          />
        </div>
        <button
          className="bg-gradient-to-br from-black to-neutral-600 block w-full text-white rounded-md h-10 font-medium shadow-[0px_1px_0px_0px_#ffffff40_inset,0px_-1px_0px_0px_#ffffff40_inset] dark:shadow-[0px_1px_0px_0px_var(--zinc-800)_inset,0px_-1px_0px_0px_var(--zinc-800)_inset]"
          type="submit"
        >
          Register &rarr;
        </button>
        <p className="text-right text-neutral-600 text-sm mt-4 dark:text-neutral-300">
          Already have an account? <Link href="/login">Login</Link>
        </p>
      </form>
    </div>
  );
};

export default Register;
```

- Collects first name, last name, email, and password.
- Submits to the `register` server action.
- Redirects authenticated users to the homepage.

---

## Authentication Flow

### Register with Credentials
1. User navigates to `/register`.
2. Fills out the form with first name, last name, email, and password.
3. Form submits to the `register` server action (`lib/user.ts`).
4. The action:
   - Validates input fields.
   - Connects to MongoDB.
   - Checks for existing user by email.
   - Hashes the password with `bcryptjs`.
   - Creates a new user in MongoDB.
   - Redirects to `/login`.

### Login with Credentials
1. User navigates to `/login`.
2. Enters email and password.
3. Form submits to the `login` server action (`lib/user.ts`).
4. The action:
   - Calls `signIn("credentials", ...)` from Auth.js.
   - Auth.js `authorize` function (in `auth.ts`) verifies credentials:
     - Connects to MongoDB.
     - Finds user by email.
     - Compares password using `bcryptjs`.
     - Returns user data if valid.
   - On success, redirects to `/`.

### Login with Google
1. User clicks "Sign in with Google" on `/login`.
2. Form submits to a server action that calls `signIn("google")`.
3. Auth.js redirects to Google's OAuth flow.
4. After authorization, the `signIn` callback (in `auth.ts`):
   - Connects to MongoDB.
   - Checks if the user exists by email.
   - Creates a new user if not found, using Google profile data.
   - Updates `authProviderId` if the user exists.
   - Stores user data in the session.
5. Redirects to `/`.

### Login with GitHub
1. User clicks "Sign in with GitHub" on `/login`.
2. Form submits to a server action that calls `signIn("github")`.
3. Auth.js redirects to GitHub's OAuth flow.
4. After authorization, the `signIn` callback (in `auth.ts`):
   - Connects to MongoDB.
   - Checks if the user exists by email.
   - Creates a new user if not found, using GitHub profile data.
   - Updates `authProviderId` if the user exists.
   - Stores user data in the session.
5. Redirects to `/`.

---

## Best Practices and Optimizations
1. **Error Handling**:
   - Add client-side validation for forms.
   - Display error messages on the login/register pages using `useFormState` or `useFormStatus` from `react-dom`.
2. **Security**:
   - Ensure `NEXTAUTH_SECRET` is a strong, random string.
   - Use HTTPS in production.
   - Sanitize user input to prevent injection attacks.
3. **Performance**:
   - Cache MongoDB connections using `mongoose.connection.readyState`.
   - Use React's `cache` for session data.
4. **User Experience**:
   - Add loading states for form submissions.
   - Implement a password confirmation field for registration.
   - Add a "Forgot Password" feature if needed.
5. **Database**:
   - Add indexes to `email` and `authProviderId` in the User schema for faster queries.
   - Handle duplicate email errors gracefully in OAuth flows.

---

## Troubleshooting
- **MongoDB Connection Issues**:
  - Verify `MONGO_URI` is correct and MongoDB is running.
  - Check network access if using a remote database.
- **OAuth Errors**:
  - Ensure `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `GOOGLE_CLIENT_ID`, and `GOOGLE_CLIENT_SECRET` are correct.
  - Set `NEXTAUTH_URL` to match your deployment URL.
- **Credentials Login Fails**:
  - Verify password hashing/comparison logic.
  - Ensure the user exists in MongoDB with a password.
- **Session Issues**:
  - Check `auth.ts` callbacks for correct token/session handling.
  - Ensure middleware is applied to protected routes.

---

This guide should provide a clear, reproducible authentication flow for your Next.js app. Refer to the code examples above for implementation details, and ensure all environment variables are properly configured before running the app.
