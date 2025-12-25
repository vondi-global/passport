# API endpoints и Backend сервисы

> Часть паспорта [vondi-ai-landing](./README.md)

## API Endpoints

### POST /api/chat

**Назначение:** AI демо чат с Claude Haiku

**Расположение:** `src/app/api/chat/route.ts`

#### Request

```typescript
// Method: POST
// Content-Type: application/json

interface ChatRequest {
  message: string;                    // Текст сообщения (max 500 символов)
  conversationHistory?: Array<{       // История (опционально, max 10 сообщений)
    role: 'user' | 'assistant';
    content: string;
  }>;
  locale?: string;                    // 'sr' | 'en' | 'ru' (default: 'sr')
}
```

#### Response (Success 200)

```typescript
interface ChatResponse {
  reply: string;                      // Ответ AI
  remaining: number;                  // Оставшиеся запросы в минуту
  model: string;                      // 'haiku'
  locale: string;                     // Язык ответа
}
```

#### Error Responses

| Код | Error Code | Описание |
|-----|-----------|----------|
| 400 | `INVALID_MESSAGE` | Сообщение не предоставлено |
| 400 | `MESSAGE_TOO_LONG` | Сообщение > 500 символов |
| 429 | `RATE_LIMIT_EXCEEDED` | Превышен лимит запросов |
| 503 | `AI_SERVICE_UNAVAILABLE` | API ключ не настроен |
| 503 | `AI_SERVICE_ERROR` | Ошибка Anthropic API |
| 500 | `INTERNAL_ERROR` | Внутренняя ошибка |

#### Rate Limits

| Период | Лимит |
|--------|-------|
| Минута | 10 запросов |
| Час | 50 запросов |
| День | 100 запросов |

**Идентификация:** По IP адресу (headers: `x-forwarded-for` или `x-real-ip`)

#### Пример запроса

```bash
curl -X POST http://localhost:3008/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Kako AI može pomoći u automatizaciji?",
    "locale": "sr"
  }'
```

#### Пример ответа

```json
{
  "reply": "AI može značajno poboljšati automatizaciju kroz obradu dokumenata, analizu podataka i interakciju sa korisnicima...",
  "remaining": 9,
  "model": "haiku",
  "locale": "sr"
}
```

---

### POST /api/contact

**Назначение:** Обработка контактной формы

**Расположение:** `src/app/api/contact/route.ts`

#### Request

```typescript
// Method: POST
// Content-Type: application/json

interface ContactRequest {
  name: string;                       // 2-100 символов
  email: string;                      // Валидный email
  phone?: string;                     // 6-20 символов (опционально)
  company?: string;                   // Max 200 символов (опционально)
  requestType: 'demo' | 'pricing' | 'support' | 'other';
  message: string;                    // 10-2000 символов
}
```

#### Response (Success 200)

```json
{
  "success": true,
  "message": "Poruka je uspešno poslata!"
}
```

#### Error Responses

| Код | Описание |
|-----|----------|
| 400 | Validation error (детали в `details`) |
| 500 | Failed to send email |
| 500 | Internal server error |

#### Пример запроса

```bash
curl -X POST http://localhost:3008/api/contact \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Ivan Petrović",
    "email": "ivan@example.com",
    "phone": "+381641234567",
    "company": "TechCorp d.o.o.",
    "requestType": "demo",
    "message": "Zainteresovani smo za implementaciju AI rešenja..."
  }'
```

---

## Backend сервисы

### AI Client (`src/lib/ai-client.ts`)

**Назначение:** Интеграция с Anthropic Claude API

#### Конфигурация

```typescript
const MODEL = 'claude-3-haiku-20240307';
const MAX_TOKENS = 300;
```

#### Функции

**sendMessage(messages: Message[]): Promise<string>**

```typescript
interface Message {
  role: 'user' | 'assistant';
  content: string;
}

async function sendMessage(messages: Message[]): Promise<string> {
  const anthropic = new Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY,
  });

  const response = await anthropic.messages.create({
    model: MODEL,
    max_tokens: MAX_TOKENS,
    system: SYSTEM_PROMPT,
    messages: messages,
  });

  return response.content[0].text;
}
```

**getModelInfo(): ModelInfo**

```typescript
interface ModelInfo {
  model: string;
  maxTokens: number;
  provider: string;
}

function getModelInfo(): ModelInfo {
  return {
    model: 'haiku',
    maxTokens: MAX_TOKENS,
    provider: 'anthropic',
  };
}
```

#### System Prompt

```
Ti si VONDI AI Asistent - demo verzija AI chatbot-a kompanije VONDI GLOBAL D.O.O. iz Beograda.

Tvoja uloga:
- Predstaviti mogućnosti VONDI AI rešenja za automatizaciju
- Odgovarati na pitanja o AI za državnu upravu i biznis
- Biti profesionalan, prijatan i informativan

Ograničenja:
- Ne obećavaj zakazivanje demo prezentacija ili poziva
- Ne prikupljaj lične podatke korisnika
- Ne daj medicinske, pravne ili finansijske savete
- Odgovori kratko (3-4 rečenice)

Jezici: Srpski, Engleski, Ruski - odgovaraj na jeziku pitanja.
```

---

### Zoho Mail Integration (`src/lib/zoho-mail.ts`)

**Назначение:** Отправка email через Zoho Mail API

#### Конфигурация (Environment Variables)

```bash
ZOHO_CLIENT_ID=<client_id>
ZOHO_CLIENT_SECRET=<client_secret>
ZOHO_REFRESH_TOKEN=<refresh_token>
ZOHO_ACCOUNT_ID=<account_id>
ZOHO_ACCOUNTS_DOMAIN=accounts.zoho.eu
ZOHO_MAIL_DOMAIN=mail.zoho.eu
ZOHO_FROM_EMAIL=mail@vondi.rs
CHAT_NOTIFY_EMAIL=mail@vondi.rs
```

#### OAuth2 Flow

```typescript
// Token caching
let cachedToken: { token: string; expiresAt: number } | null = null;

async function getAccessToken(): Promise<string> {
  // Check cache (with 60s buffer)
  if (cachedToken && Date.now() < cachedToken.expiresAt - 60000) {
    return cachedToken.token;
  }

  // Refresh token
  const response = await fetch(
    `https://${ZOHO_ACCOUNTS_DOMAIN}/oauth/v2/token`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams({
        refresh_token: ZOHO_REFRESH_TOKEN,
        client_id: ZOHO_CLIENT_ID,
        client_secret: ZOHO_CLIENT_SECRET,
        grant_type: 'refresh_token',
      }),
    }
  );

  const data = await response.json();
  cachedToken = {
    token: data.access_token,
    expiresAt: Date.now() + data.expires_in * 1000,
  };

  return cachedToken.token;
}
```

#### Функции

**sendEmail(options): Promise<boolean>**

```typescript
interface SendEmailOptions {
  to: string;
  subject: string;
  body: string;
  isHtml?: boolean;
}

async function sendEmail(options: SendEmailOptions): Promise<boolean> {
  const token = await getAccessToken();

  const response = await fetch(
    `https://${ZOHO_MAIL_DOMAIN}/api/accounts/${ZOHO_ACCOUNT_ID}/messages`,
    {
      method: 'POST',
      headers: {
        Authorization: `Zoho-oauthtoken ${token}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        fromAddress: ZOHO_FROM_EMAIL,
        toAddress: options.to,
        subject: options.subject,
        content: options.body,
        mailFormat: options.isHtml ? 'html' : 'plaintext',
      }),
    }
  );

  return response.ok;
}
```

**notifyAdminAboutChatActivity(params): Promise<boolean>**

```typescript
interface ChatNotifyParams {
  userIP: string;
  firstMessage: string;
  locale: string;
  userAgent?: string;
}

async function notifyAdminAboutChatActivity(
  params: ChatNotifyParams
): Promise<boolean> {
  const { userIP, firstMessage, locale, userAgent } = params;

  const subject = `[VONDI AI Demo] Nova chat sesija - ${locale.toUpperCase()}`;

  const body = `
Nova chat sesija započeta na solutions.vondi.rs

========================================
DETALJI SESIJE
========================================

IP Adresa: ${userIP}
Jezik: ${locale}
User Agent: ${userAgent || 'N/A'}
Vreme: ${new Date().toISOString()}

========================================
PRVO PITANJE
========================================

${firstMessage}

---
Ova notifikacija je automatski generisana.
  `.trim();

  return sendEmail({
    to: CHAT_NOTIFY_EMAIL,
    subject,
    body,
    isHtml: false,
  });
}
```

---

### Email Service (`src/lib/email.ts`)

**Назначение:** Wrapper для отправки email с fallback на console.log

#### Функции

**sendContactEmail(data): Promise<EmailResult>**

```typescript
interface EmailData {
  name: string;
  email: string;
  phone?: string;
  company?: string;
  requestType: string;
  message: string;
}

interface EmailResult {
  success: boolean;
  message: string;
}

async function sendContactEmail(data: EmailData): Promise<EmailResult> {
  // В development без Zoho credentials - логируем в консоль
  if (!process.env.ZOHO_CLIENT_ID && process.env.NODE_ENV !== 'production') {
    console.log('[EMAIL] Contact form submission:', data);
    return { success: true, message: 'Email logged (dev mode)' };
  }

  const typeLabels: Record<string, string> = {
    demo: 'Zahtev za demo',
    pricing: 'Pitanje o cenama',
    support: 'Tehnička podrška',
    other: 'Ostalo',
  };

  const body = `
Nova poruka sa sajta solutions.vondi.rs

========================================
KONTAKT INFORMACIJE
========================================

Ime: ${data.name}
Email: ${data.email}
Telefon: ${data.phone || 'Nije navedeno'}
Kompanija: ${data.company || 'Nije navedeno'}
Tip zahteva: ${typeLabels[data.requestType] || data.requestType}

========================================
PORUKA
========================================

${data.message}
  `.trim();

  const success = await sendEmail({
    to: 'mail@vondi.rs',
    subject: `[Contact Form] ${typeLabels[data.requestType]} - ${data.name}`,
    body,
    isHtml: false,
  });

  return {
    success,
    message: success ? 'Email sent' : 'Failed to send email',
  };
}
```

---

### Rate Limiter (`src/lib/rate-limiter.ts`)

**Назначение:** In-memory rate limiting по IP адресу

#### Конфигурация

```typescript
const LIMITS = {
  perMinute: parseInt(process.env.RATE_LIMIT_PER_MINUTE || '10'),
  perHour: parseInt(process.env.RATE_LIMIT_PER_HOUR || '50'),
  perDay: parseInt(process.env.RATE_LIMIT_PER_DAY || '100'),
};

const MAX_MESSAGE_LENGTH = 500;
```

#### Хранилище

```typescript
// In-memory store (для production рекомендуется Redis)
const store = new Map<string, { count: number; resetAt: number }>();

// Автоочистка каждые 5 минут
setInterval(() => {
  const now = Date.now();
  for (const [key, value] of store.entries()) {
    if (value.resetAt < now) {
      store.delete(key);
    }
  }
}, 5 * 60 * 1000);
```

#### Функции

**rateLimit(ip: string): Promise<RateLimitResult>**

```typescript
interface RateLimitResult {
  success: boolean;
  remaining: number;
  resetAt: number;
  error?: string;
}

async function rateLimit(ip: string): Promise<RateLimitResult> {
  const now = Date.now();
  const periods = [
    { key: `rate:minute:${ip}`, limit: LIMITS.perMinute, ttl: 60 * 1000 },
    { key: `rate:hour:${ip}`, limit: LIMITS.perHour, ttl: 60 * 60 * 1000 },
    { key: `rate:day:${ip}`, limit: LIMITS.perDay, ttl: 24 * 60 * 60 * 1000 },
  ];

  for (const { key, limit, ttl } of periods) {
    const record = store.get(key);

    if (!record || record.resetAt < now) {
      store.set(key, { count: 1, resetAt: now + ttl });
      continue;
    }

    if (record.count >= limit) {
      return {
        success: false,
        remaining: 0,
        resetAt: record.resetAt,
        error: 'RATE_LIMIT_EXCEEDED',
      };
    }

    record.count++;
  }

  const minuteRecord = store.get(`rate:minute:${ip}`);
  return {
    success: true,
    remaining: LIMITS.perMinute - (minuteRecord?.count || 0),
    resetAt: minuteRecord?.resetAt || now + 60 * 1000,
  };
}
```

**validateMessage(message: string): boolean**

```typescript
function validateMessage(message: string): boolean {
  return typeof message === 'string' &&
         message.length > 0 &&
         message.length <= MAX_MESSAGE_LENGTH;
}
```

**getLimits(): LimitConfig**

```typescript
interface LimitConfig {
  perMinute: number;
  perHour: number;
  perDay: number;
  maxMessageLength: number;
}

function getLimits(): LimitConfig {
  return {
    ...LIMITS,
    maxMessageLength: MAX_MESSAGE_LENGTH,
  };
}
```

---

### Analytics (`src/lib/analytics.ts`)

**Назначение:** Интеграция с Plausible Analytics

#### Функции

```typescript
// Client-side only
declare global {
  interface Window {
    plausible?: (event: string, options?: { props?: Record<string, string> }) => void;
  }
}

function trackEvent(eventName: string, props?: Record<string, string>): void {
  if (typeof window === 'undefined') return;

  if (window.plausible) {
    window.plausible(eventName, { props });
  } else {
    console.log('[Analytics]', eventName, props);
  }
}

// Специализированные функции
function trackChatOpened(): void {
  trackEvent('Chat Opened');
}

function trackChatMessageSent(): void {
  trackEvent('Chat Message Sent');
}

function trackChatDemoCompleted(): void {
  trackEvent('Chat Demo Completed');
}

function trackChatCTAClicked(ctaType: string): void {
  trackEvent('Chat CTA Clicked', { type: ctaType });
}
```

---

## TypeScript Types (`src/types/index.ts`)

```typescript
// Message для чата
export interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
}

// Данные контактной формы
export interface ContactFormData {
  name: string;
  email: string;
  phone?: string;
  company?: string;
  requestType: 'demo' | 'pricing' | 'support' | 'other';
  message: string;
}

// Общий API response
export interface APIResponse<T = unknown> {
  success: boolean;
  data?: T;
  message?: string;
  error?: string;
  details?: unknown;
}

// Локаль
export type Locale = 'sr' | 'en' | 'ru';

// Chat API response
export interface ChatAPIResponse {
  reply: string;
  remaining: number;
  model: string;
  locale: string;
}

// Rate limit result
export interface RateLimitResult {
  success: boolean;
  remaining: number;
  resetAt: number;
  error?: string;
}
```

---

## Data Flow диаграммы

### Chat Flow

```
┌─────────┐     POST /api/chat     ┌──────────┐
│  User   │ ──────────────────────▶│  API     │
│ Browser │                        │  Route   │
└─────────┘                        └────┬─────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
              ┌──────────┐       ┌──────────┐       ┌──────────┐
              │  Rate    │       │  Validate │       │  Notify  │
              │  Limiter │       │  Message  │       │  Admin   │
              └────┬─────┘       └────┬─────┘       │  (async) │
                   │                  │             └──────────┘
                   ▼                  ▼
              ┌───────────────────────────────┐
              │        Anthropic API          │
              │     (Claude 3 Haiku)          │
              └───────────────────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │  JSON Response   │
                    │  { reply, ...}   │
                    └──────────────────┘
```

### Contact Form Flow

```
┌─────────┐     POST /api/contact   ┌──────────┐
│  User   │ ──────────────────────▶│  API     │
│ Browser │                        │  Route   │
└─────────┘                        └────┬─────┘
                                        │
                              ┌─────────┴─────────┐
                              ▼                   ▼
                        ┌──────────┐       ┌──────────┐
                        │   Zod    │       │  Format  │
                        │ Validate │       │  Email   │
                        └────┬─────┘       └────┬─────┘
                             │                  │
                             ▼                  ▼
                       ┌───────────────────────────────┐
                       │         Zoho Mail API         │
                       │   (OAuth2 → Send Email)       │
                       └───────────────────────────────┘
                                      │
                                      ▼
                            ┌──────────────────┐
                            │  JSON Response   │
                            │  { success }     │
                            └──────────────────┘
```

### Chat Notification Flow (Background)

```
┌──────────────────┐
│  First message   │
│  from new user   │
└────────┬─────────┘
         │ async (not blocking response)
         ▼
┌──────────────────┐
│  Get Zoho token  │
│  (from cache)    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Format notify   │
│  email body      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Send to         │
│  CHAT_NOTIFY_    │
│  EMAIL           │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Silently handle │
│  errors (no      │
│  user impact)    │
└──────────────────┘
```

---

## Внешние интеграции

| Сервис | Назначение | Endpoint | Auth |
|--------|-----------|----------|------|
| **Anthropic API** | AI Chat | api.anthropic.com | API Key |
| **Zoho Mail** | Email sending | mail.zoho.eu | OAuth2 |
| **Plausible** | Analytics | Client-side JS | None |

### Fallback поведение

| Сервис | Fallback |
|--------|----------|
| Anthropic API | 503 error response |
| Zoho Mail | Console.log в dev mode |
| Plausible | Console.log |
