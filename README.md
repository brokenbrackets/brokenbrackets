<!---
brokenbrackets/brokenbrackets is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
// package.json
{
  "name": "tennis-bracket-app",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "@prisma/client": "^5.x",
    "@stripe/stripe-js": "^2.x",
    "next": "14.x",
    "next-auth": "^4.x",
    "react": "^18.x",
    "react-dom": "^18.x",
    "stripe": "^14.x",
    "zod": "^3.x"
  },
  "devDependencies": {
    "prisma": "^5.x",
    "typescript": "^5.x"
  }
}

// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  name          String?
  brackets      Bracket[]
  subscription  String?   // "free", "premium"
  stripeCustomerId String?
}

model Bracket {
  id        String   @id @default(cuid())
  name      String
  data      Json     // Store the bracket data
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

// pages/api/auth/[...nextauth].ts
import NextAuth from 'next-auth';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { prisma } from '@/lib/prisma';
import GoogleProvider from 'next-auth/providers/google';

export default NextAuth({
  adapter: PrismaAdapter(prisma),
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    }),
  ],
  callbacks: {
    session: async ({ session, user }) => {
      session.user.id = user.id;
      return session;
    },
  },
});

// pages/api/create-checkout-session.ts
import { stripe } from '@/lib/stripe';
import { getSession } from 'next-auth/react';

export default async function handler(req, res) {
  const session = await getSession({ req });
  
  if (!session?.user) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  const checkoutSession = await stripe.checkout.sessions.create({
    mode: 'subscription',
    payment_method_types: ['card'],
    line_items: [
      {
        price: process.env.STRIPE_PRICE_ID,
        quantity: 1,
      },
    ],
    success_url: `${process.env.NEXTAUTH_URL}/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${process.env.NEXTAUTH_URL}/pricing`,
    customer_email: session.user.email,
  });

  return res.json({ url: checkoutSession.url });
}
