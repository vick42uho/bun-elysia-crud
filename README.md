# Bun + Elysia + Prisma 7 + Supabase CRUD API

โปรเจกต์ตัวอย่างการสร้าง High-Performance RESTful API ด้วย Tech Stack ใหม่ล่าสุด:
- **Runtime:** Bun
- **Framework:** ElysiaJS
- **ORM:** Prisma 7 (พร้อม Driver Adapter & PrismaBox)
- **Database:** Supabase (PostgreSQL + PgBouncer)
- **Deployment:** Vercel

---

## 🛠️ Prerequisites (สิ่งที่ต้องเตรียม)

1.  **Bun:** ติดตั้ง Bun ([คู่มือติดตั้ง](https://bun.sh/))
2.  **Supabase:** สร้าง Project ใหม่และเตรียม Connection String (Transaction Pooler & Session/Direct)

---

## 🚀 Step-by-Step Setup Guide

### Step 1: สร้างโปรเจกต์และติดตั้ง Dependencies

1. สร้างโฟลเดอร์และเริ่มโปรเจกต์
```bash
mkdir bun-elysia-crud
cd bun-elysia-crud
bun init -y
```
2. ติดตั้ง Core Dependencies
```bash
bun add elysia @elysiajs/swagger
bun add @prisma/client @prisma/adapter-pg pg
```
3. ติดตั้ง Dev Dependencies
```bash
bun add -d prisma prismabox typescript @types/pg
```

### Step 2: ตั้งค่าฐานข้อมูล (Database Configuration)
1.เริ่มระบบ Prisma
```bash
bunx prisma init
```
2.ตั้งค่า Environment Variables ในไฟล์ .env
สำคัญ: ใส่ทั้ง 2 ค่าที่ได้จาก Supabase เพื่อรองรับ Prisma 7 Workflow
```bash
# ใช้สำหรับ App connect ปกติ (Transaction Mode / Port 6543)
DATABASE_URL="postgresql://postgres:[password]@[aws-0-xx-xx-xx.pooler.supabase.com:6543/postgres?pgbouncer=true](https://aws-0-xx-xx-xx.pooler.supabase.com:6543/postgres?pgbouncer=true)"

# ใช้สำหรับ Migrations/Push DB (Session Mode / Port 5432)
DIRECT_URL="postgresql://postgres:[password]@[aws-0-xx-xx-xx.pooler.supabase.com:5432/postgres](https://aws-0-xx-xx-xx.pooler.supabase.com:5432/postgres)"
```
3.แก้ไข prisma/schema.prisma เพิ่ม Generator สำหรับ Prisma Client และ PrismaBox (Validation)
```bash
generator client {
  provider = "prisma-client-js"
  output   = "../generated/prisma"
}

generator prismabox {
  provider = "prismabox"
  typeboxImportDependencyName = "elysia"
  typeboxImportVariableName   = "t"
  inputModel = true
  output     = "../generated/prismabox"
}

datasource db {
  provider = "postgresql"
  // ไม่ต้องใส่ url ตรงนี้ใน Prisma 7 (เราจะไป config ในไฟล์แยก)
}

model Product {
  id     String   @id @default(cuid())
  name   String   @unique
  detail String
  price  Decimal?
}
```
4.สร้างไฟล์ prisma.config.ts ที่ root folder เพื่อให้คำสั่ง CLI รู้จัก Direct URL
```bash
import "dotenv/config";
import { defineConfig } from "@prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  datasource: {
    url: process.env.DIRECT_URL, 
  },
});
```
5.อัปเดตฐานข้อมูลและสร้างโค้ด
```bash
bunx prisma generate
bunx prisma db push
```
### Step 3: เขียนโค้ด API
แก้ไขไฟล์ src/index.ts โดยใส่โค้ดดังนี้:
```bash
import { Elysia, t } from 'elysia'
import { swagger } from '@elysiajs/swagger'
import { PrismaPg } from '@prisma/adapter-pg'
import { Pool } from 'pg'

import { PrismaClient } from '../generated/prisma/client'
import {
  ProductPlain,
  ProductPlainInputCreate,
  ProductPlainInputUpdate
} from '../generated/prismabox/Product'

const pool = new Pool({ connectionString: process.env.DATABASE_URL })
const adapter = new PrismaPg(pool)

const prisma = new PrismaClient({ adapter })

// แปลง Decimal เป็น number สำหรับ JSON response
const toResponse = (product: { id: string; name: string; detail: string; price: { toNumber: () => number } | null }) => ({
  id: product.id,
  name: product.name,
  detail: product.detail,
  price: product.price?.toNumber() ?? null
})

const app = new Elysia()

// Swagger
  .use(swagger())


// CREATE - สร้างสินค้าใหม่
  .post(
    '/products',
    async ({ body }) => {
      const product = await prisma.product.create({ data: body })
      return toResponse(product)
    },
    {
      body: ProductPlainInputCreate,
      response: ProductPlain
    }
  )

// READ - ดึงสินค้าทั้งหมด
  .get(
    '/products',
    async () => {
      const products = await prisma.product.findMany()
      return products.map(toResponse)
    },
    {
      response: t.Array(ProductPlain)
    }
  )

// READ - ดึงสินค้าตาม ID
  .get(
    '/products/:id',
    async ({ params: { id }, status }) => {
      const product = await prisma.product.findUnique({ where: { id } })
      if (!product) return status(404, 'Product not found')
      return toResponse(product)
    },
    {
      response: {
        200: ProductPlain,
        404: t.String()
      }
    }
  )

// UPDATE - อัปเดตสินค้า
  .patch(
    '/products/:id',
    async ({ params: { id }, body, set }) => {
      try {
        const product = await prisma.product.update({
          where: { id },
          data: body
        })
        return toResponse(product)
      } catch {
        set.status = 404
        return 'Product not found'
      }
    },
    {
      body: ProductPlainInputUpdate,
      response: {
        200: ProductPlain,
        404: t.String()
      }
    }
  )

// DELETE - ลบสินค้า
  .delete(
    '/products/:id',
    async ({ params: { id }, set }) => {
      try {
        await prisma.product.delete({ where: { id } })
        return { message: 'Product deleted' }
      } catch {
        set.status = 404
        return { message: 'Product not found' }
      }
    },
    {
      response: t.Object({ message: t.String() })
    }
  )

// For Vercel: export as default
export default app

// For local dev: start server
if (import.meta.main || process.env.NODE_ENV !== 'production') {
  app.listen(3000)
  console.log(
    `🦊 Elysia is running at ${app.server?.hostname}:${app.server?.port}`
  )
}
```
### Step 4: ทดสอบระบบ (Testing)
1. เพิ่ม Script ใน package.json
```JSON
"scripts": {
  "dev": "bun run --watch src/index.ts",
  "build": "bunx prisma generate"
}
```
2. รัน Server
```bash
bun run dev
```
3. ทดสอบ

เปิด Browser ไปที่: http://localhost:3000/swagger เพื่อดูเอกสาร API

ยิง API ผ่าน Postman/Thunder Client ไปที่ http://localhost:3000/products

### Step 5: นำขึ้นออนไลน์ (Deploy to Vercel)
1. สร้างไฟล์ vercel.json
```JSON
{
  "buildCommand": "bunx prisma generate",
  "bunVersion": "1.x",
  "rewrites": [{ "source": "/(.*)", "destination": "/src" }]
}
```
2. สร้างไฟล์ tsconfig.json (Optimized for Vercel)
```JSON
{
  "compilerOptions": {
    "target": "ES2021",
    "module": "ES2022",
    "moduleResolution": "node",
    "types": ["bun-types"],
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "baseUrl": "."
  }
}
```
3.Deploy:

Push Code ขึ้น GitHub

เชื่อมต่อกับ Vercel

สำคัญ: ไปที่ Settings -> Environment Variables บน Vercel แล้วเพิ่ม DATABASE_URL (ใช้ค่าเดียวกับใน .env ที่มี pgbouncer=true)

## 📚 References

- [Bun Documentation](https://bun.sh/)
- [ElysiaJS Documentation](https://elysiajs.com/)
- [Prisma Documentation](https://prisma.io/docs/)
- [Vercel Documentation](https://vercel.com/docs)
- [Supabase Documentation](https://supabase.com/docs)
