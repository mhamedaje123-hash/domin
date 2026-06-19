index.html

## 1. قاعدة البيانات - Schema (drizzle/schema.ts)

```typescript
import {
  int,
  mysqlEnum,
  mysqlTable,
  text,
  timestamp,
  varchar,
  decimal,
  boolean,
} from "drizzle-orm/mysql-core";

export const users = mysqlTable("users", {
  id: int("id").autoincrement().primaryKey(),
  openId: varchar("openId", { length: 64 }).notNull().unique(),
  name: text("name"),
  email: varchar("email", { length: 320 }),
  loginMethod: varchar("loginMethod", { length: 64 }),
  role: mysqlEnum("role", ["user", "admin"]).default("user").notNull(),
  walletBalance: decimal("walletBalance", { precision: 10, scale: 2 }).default("0.00").notNull(),
  createdAt: timestamp("createdAt").defaultNow().notNull(),
  updatedAt: timestamp("updatedAt").defaultNow().onUpdateNow().notNull(),
  lastSignedIn: timestamp("lastSignedIn").defaultNow().notNull(),
});

export type User = typeof users.$inferSelect;
export type InsertUser = typeof users.$inferInsert;

export const categories = mysqlTable("categories", {
  id: int("id").autoincrement().primaryKey(),
  name: varchar("name", { length: 128 }).notNull(),
  nameAr: varchar("nameAr", { length: 128 }).notNull(),
  type: mysqlEnum("type", ["games", "apps", "social_media", "other"]).notNull(),
  icon: text("icon"),
  imageUrl: text("imageUrl"),
  sortOrder: int("sortOrder").default(0).notNull(),
  isActive: boolean("isActive").default(true).notNull(),
  createdAt: timestamp("createdAt").defaultNow().notNull(),
});

export type Category = typeof categories.$inferSelect;
export type InsertCategory = typeof categories.$inferInsert;

export const products = mysqlTable("products", {
  id: int("id").autoincrement().primaryKey(),
  categoryId: int("categoryId").notNull(),
  name: varchar("name", { length: 256 }).notNull(),
  nameAr: varchar("nameAr", { length: 256 }).notNull(),
  description: text("description"),
  price: decimal("price", { precision: 10, scale: 2 }).notNull(),
  imageUrl: text("imageUrl"),
  isActive: boolean("isActive").default(true).notNull(),
  sortOrder: int("sortOrder").default(0).notNull(),
  createdAt: timestamp("createdAt").defaultNow().notNull(),
  updatedAt: timestamp("updatedAt").defaultNow().onUpdateNow().notNull(),
});

export type Product = typeof products.$inferSelect;
export type InsertProduct = typeof products.$inferInsert;

export const paymentMethods = mysqlTable("paymentMethods", {
  id: int("id").autoincrement().primaryKey(),
  name: varchar("name", { length: 128 }).notNull(),
  nameAr: varchar("nameAr", { length: 128 }).notNull(),
  instructions: text("instructions"),
  instructionsAr: text("instructionsAr"),
  accountInfo: text("accountInfo"),
  logoUrl: text("logoUrl"),
  isActive: boolean("isActive").default(true).notNull(),
  sortOrder: int("sortOrder").default(0).notNull(),
  createdAt: timestamp("createdAt").defaultNow().notNull(),
  updatedAt: timestamp("updatedAt").defaultNow().onUpdateNow().notNull(),
});

export type PaymentMethod = typeof paymentMethods.$inferSelect;
export type InsertPaymentMethod = typeof paymentMethods.$inferInsert;

export const walletTopupRequests = mysqlTable("walletTopupRequests", {
  id: int("id").autoincrement().primaryKey(),
  userId: int("userId").notNull(),
  paymentMethodId: int("paymentMethodId").notNull(),
  amount: decimal("amount", { precision: 10, scale: 2 }).notNull(),
  proofNote: text("proofNote"),
  proofImage: text("proofImage"),
  status: mysqlEnum("status", ["pending", "accepted", "rejected"]).default("pending").notNull(),
  adminNote: text("adminNote"),
  processedAt: timestamp("processedAt"),
  createdAt: timestamp("createdAt").defaultNow().notNull(),
  updatedAt: timestamp("updatedAt").defaultNow().onUpdateNow().notNull(),
});

export type WalletTopupRequest = typeof walletTopupRequests.$inferSelect;
export type InsertWalletTopupRequest = typeof walletTopupRequests.$inferInsert;

export const serviceOrders = mysqlTable("serviceOrders", {
  id: int("id").autoincrement().primaryKey(),
  userId: int("userId").notNull(),
  productId: int("productId").notNull(),
  gameId: varchar("gameId", { length: 256 }).notNull(),
  quantity: int("quantity").default(1).notNull(),
  totalPrice: decimal("totalPrice", { precision: 10, scale: 2 }).notNull(),
  status: mysqlEnum("status", ["pending", "accepted", "rejected"]).default("pending").notNull(),
  adminNote: text("adminNote"),
  processedAt: timestamp("processedAt"),
  createdAt: timestamp("createdAt").defaultNow().notNull(),
  updatedAt: timestamp("updatedAt").defaultNow().onUpdateNow().notNull(),
});

export type ServiceOrder = typeof serviceOrders.$inferSelect;
export type InsertServiceOrder = typeof serviceOrders.$inferInsert;

export const balanceAdjustments = mysqlTable("balanceAdjustments", {
  id: int("id").autoincrement().primaryKey(),
  userId: int("userId").notNull(),
  adminId: int("adminId").notNull(),
  amount: decimal("amount", { precision: 10, scale: 2 }).notNull(),
  type: mysqlEnum("type", ["add", "subtract", "set"]).notNull(),
  note: text("note"),
  createdAt: timestamp("createdAt").defaultNow().notNull(),
});

export type BalanceAdjustment = typeof balanceAdjustments.$inferSelect;
```

---

## 2. قاعدة البيانات - Query Helpers (server/db.ts)

انظر الملف الكامل في المشروع: `/home/ubuntu/topup-store/server/db.ts`

يحتوي على:
- دوال إدارة المستخدمين (users)
- دوال إدارة الفئات (categories)
- دوال إدارة المنتجات (products)
- دوال إدارة طرق الدفع (paymentMethods)
- دوال إدارة طلبات المحفظة (walletTopupRequests)
- دوال إدارة طلبات الخدمات (serviceOrders)
- دوال إدارة تعديلات الرصيد (balanceAdjustments)

---

## 3. API الخلفي - tRPC Routers (server/routers.ts)

### الـ Routers الرئيسية:

#### Auth Router
```typescript
auth: router({
  me: publicProcedure.query((opts) => opts.ctx.user),
  logout: publicProcedure.mutation(({ ctx }) => {
    const cookieOptions = getSessionCookieOptions(ctx.req);
    ctx.res.clearCookie(COOKIE_NAME, { ...cookieOptions, maxAge: -1 });
    return { success: true } as const;
  }),
}),
```

#### Categories Router
- `list` - عرض الفئات النشطة (عام)
- `listAll` - عرض جميع الفئات (أدمن)
- `create` - إنشاء فئة (أدمن)
- `update` - تحديث فئة (أدمن)
- `delete` - حذف فئة (أدمن)

#### Products Router
- `list` - عرض المنتجات النشطة (عام)
- `listAll` - عرض جميع المنتجات (أدمن)
- `getById` - الحصول على منتج محدد (عام)
- `create` - إنشاء منتج (أدمن)
- `update` - تحديث منتج (أدمن)
- `delete` - حذف منتج (أدمن)

#### Payment Methods Router
- `list` - عرض طرق الدفع النشطة (عام)
- `listAll` - عرض جميع طرق الدفع (أدمن)
- `create` - إنشاء طريقة دفع (أدمن)
- `update` - تحديث طريقة دفع (أدمن)
- `delete` - حذف طريقة دفع (أدمن)

#### Wallet Router
```typescript
wallet: router({
  getBalance: protectedProcedure.query(async ({ ctx }) => {
    const user = await getUserById(ctx.user.id);
    return { balance: user?.walletBalance ?? "0.00" };
  }),
  requestTopup: protectedProcedure
    .input(z.object({
      paymentMethodId: z.number(),
      amount: z.string(),
      proofNote: z.string().optional(),
      proofImage: z.string().optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      await createWalletTopupRequest({
        userId: ctx.user.id,
        paymentMethodId: input.paymentMethodId,
        amount: input.amount,
        proofNote: input.proofNote,
        proofImage: input.proofImage,
        status: "pending",
      });
      return { success: true };
    }),
  topupHistory: protectedProcedure.query(async ({ ctx }) => {
    return getWalletTopupRequestsByUserWithMethod(ctx.user.id);
  }),
}),
```

#### Orders Router
```typescript
orders: router({
  create: protectedProcedure
    .input(z.object({
      productId: z.number(),
      gameId: z.string().min(1),
      quantity: z.number().min(1).default(1),
    }))
    .mutation(async ({ ctx, input }) => {
      // التحقق من المنتج والرصيد
      // خصم الرصيد فوراً
      // إنشاء الطلب
    }),
  myOrders: protectedProcedure.query(async ({ ctx }) => {
    return getServiceOrdersByUser(ctx.user.id);
  }),
}),
```

#### Admin Router
```typescript
admin: router({
  users: router({
    list: adminProcedure.query(() => getAllUsers()),
    adjustBalance: adminProcedure
      .input(z.object({
        userId: z.number(),
        type: z.enum(["add", "subtract", "set"]),
        amount: z.string(),
        note: z.string().optional(),
      }))
      .mutation(async ({ ctx, input }) => {
        // تعديل رصيد المستخدم
      }),
  }),
  topupRequests: router({
    list: adminProcedure.query(() => getAllWalletTopupRequests()),
    approve: adminProcedure.input(z.object({ id: z.number(), adminNote: z.string().optional() })).mutation(...),
    reject: adminProcedure.input(z.object({ id: z.number(), adminNote: z.string().optional() })).mutation(...),
  }),
  serviceOrders: router({
    list: adminProcedure.query(() => getAllServiceOrders()),
    approve: adminProcedure.input(z.object({ id: z.number(), adminNote: z.string().optional() })).mutation(...),
    reject: adminProcedure.input(z.object({ id: z.number(), adminNote: z.string().optional() })).mutation(...),
  }),
}),
```

---

## 4. الواجهة الأمامية - الصفحات الرئيسية

### Home.tsx - الصفحة الرئيسية
- عرض الفئات الرئيسية (ألعاب، تطبيقات، سوشيال ميديا، أخرى)
- عرض الإحصائيات (24/7 دعم، طلبات مكتملة، إلخ)
- عرض خطوات الاستخدام
- تصميم بكسل آرت ريترو

### Category.tsx - صفحة الفئة
- عرض المنتجات في الفئة المختارة
- عرض السعر بالليرة السورية (ل.س)
- زر الشحن لكل منتج

### Order.tsx - صفحة الطلب
- إدخال الآيدي
- اختيار الكمية
- عرض السعر الإجمالي
- التحقق من الرصيد
- خصم الرصيد فوراً عند الطلب

### Dashboard.tsx - لوحة العميل
- عرض الرصيد الحالي
- **قسم طلب شحن المحفظة:**
  - اختيار طريقة الدفع
  - إدخال المبلغ
  - إضافة ملاحظة
  - **رفع صورة الإيصال** (base64)
  - إرسال الطلب
- **سجل طلبات المحفظة:**
  - التاريخ
  - المبلغ
  - طريقة الدفع
  - الحالة (قيد الانتظار / مقبول / مرفوض)
- **سجل طلبات الخدمات:**
  - اسم الخدمة
  - الآيدي المستخدم
  - المبلغ
  - الحالة

### Admin.tsx - لوحة التحكم الكاملة

#### 1. نظرة عامة
- عدد طلبات المحفظة المعلقة
- عدد طلبات الخدمات المعلقة
- عدد المستخدمين
- آخر الطلبات

#### 2. إدارة الفئات
- إضافة فئة جديدة
- تعديل الفئات الموجودة
- حذف الفئات
- تفعيل/تعطيل الفئات

#### 3. إدارة المنتجات
- إضافة منتج جديد
- تعديل المنتجات
- حذف المنتجات
- عرض جدول بجميع المنتجات

#### 4. طلبات المحفظة
- عرض جميع الطلبات المعلقة
- **عرض صورة الإيصال** المرفقة
- قبول الطلب (إضافة الرصيد تلقائياً)
- رفض الطلب (مع ملاحظة)

#### 5. طلبات الخدمات
- عرض جميع الطلبات
- قبول الطلب
- رفض الطلب (استرجاع الرصيد تلقائياً)

#### 6. إدارة المستخدمين
- عرض جميع المستخدمين
- عرض رصيد كل مستخدم
- إضافة رصيد
- تعديل الرصيد
- حذف الرصيد

#### 7. طرق الدفع
- إضافة طريقة دفع جديدة
- تعديل طرق الدفع
- حذف طرق الدفع
- تفعيل/تعطيل الطرق

---

## 5. التصميم - CSS (client/src/index.css)

### الألوان الرئيسية:
```css
:root {
  --neon-green: oklch(0.85 0.25 140);
  --neon-cyan: oklch(0.85 0.2 195);
  --neon-magenta: oklch(0.7 0.3 320);
  --neon-yellow: oklch(0.92 0.22 100);
  --navy-bg: #0a0e27;
  --text-primary: oklch(0.9 0.03 200);
  --text-secondary: oklch(0.65 0.05 250);
}
```

### الخطوط:
- `Press Start 2P` - للعناوين الرئيسية (البكسل آرت)
- `Orbitron` - للأرقام والأسعار
- `Rajdhani` - للنصوص العادية

### المكونات الرئيسية:
- `.pixel-card` - بطاقة بإطار بكسل
- `.pixel-btn` - زر بتأثير بكسل
- `.neon-green`, `.neon-cyan`, etc. - ألوان نيون
- `.pixel-blink` - تأثير وميض

---

## 6. المكونات المخصصة

### PixelBackground.tsx
- أشكال هندسية عائمة
- تأثيرات glow
- خلفية متحركة

### StatusBadge.tsx
- عرض حالة الطلب (قيد الانتظار / مقبول / مرفوض)
- ألوان مختلفة لكل حالة

### Navbar.tsx
- عرض اسم المستخدم
- عرض الرصيد الحالي
- روابط التنقل
- زر تسجيل الخروج

---

## 7. البيئة والإعدادات

### متغيرات البيئة المطلوبة:
```
DATABASE_URL=mysql://...
JWT_SECRET=...
VITE_APP_ID=...
OAUTH_SERVER_URL=...
VITE_OAUTH_PORTAL_URL=...
OWNER_OPEN_ID=...
OWNER_NAME=...
```

### الأوامر الرئيسية:
```bash
# التطوير
pnpm dev

# البناء
pnpm build

# الاختبار
pnpm test

# التحقق من TypeScript
pnpm check
```

---

## 8. الميزات الرئيسية

✅ **نظام المحفظة الإلكترونية**
- طلب شحن المحفظة
- رفع صورة الإيصال
- تتبع حالة الطلب

✅ **نظام الطلبات**
- طلب خدمات الشحن
- خصم الرصيد فوراً
- تتبع الطلبات

✅ **لوحة التحكم الكاملة**
- إدارة المنتجات والفئات
- إدارة طرق الدفع
- مراجعة وقبول/رفض الطلبات
- إدارة أرصدة المستخدمين
- عرض المستخدمين

✅ **التصميم**
- بكسل آرت ريترو
- نيون ألوان زاهية
- واجهة سهلة الاستخدام
- دعم اللغة العربية

✅ **الأمان**
- مصادقة OAuth
- تحكم الأدمن
- التحقق من الرصيد

---

## 9. هيكل المشروع

```
topup-store/
├── client/
│   ├── src/
│   │   ├── pages/
│   │   │   ├── Home.tsx
│   │   │   ├── Category.tsx
│   │   │   ├── Order.tsx
│   │   │   ├── Dashboard.tsx
│   │   │   └── Admin.tsx
│   │   ├── components/
│   │   │   ├── Navbar.tsx
│   │   │   ├── PixelBackground.tsx
│   │   │   ├── StatusBadge.tsx
│   │   │   └── ui/
│   │   ├── App.tsx
│   │   ├── index.css
│   │   └── lib/trpc.ts
│   └── index.html
├── server/
│   ├── db.ts
│   ├── routers.ts
│   ├── topup.test.ts
│   └── _core/
├── drizzle/
│   ├── schema.ts
│   └── migrations/
├── shared/
├── package.json
└── tsconfig.json
```

---

## 10. ملاحظات مهمة

1. **العملة:** جميع الأسعار والأرصدة بالليرة السورية (ل.س)
2. **صور الإيصال:** يتم تخزينها كـ base64 في قاعدة البيانات
3. **الرصيد:** يتم خصمه فوراً عند الطلب (ليس عند القبول)
4. **الاسترجاع:** يتم استرجاع الرصيد فقط عند رفض طلب الخدمة
5. **الأدمن:** يتم تحديد الأدمن من خلال `OWNER_OPEN_ID`

