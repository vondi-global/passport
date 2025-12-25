# SEO оптимизация и интернационализация

> Часть паспорта [vondi-ai-landing](./README.md)

## Интернационализация (i18n)

### Поддерживаемые языки

| Код | Язык | Регион | Статус |
|-----|------|--------|--------|
| `sr` | Сербский | Сербия, Черногория, Босния | **Default** |
| `en` | Английский | Международный | Доступен |
| `ru` | Русский | СНГ, русскоязычные | Доступен |

### Конфигурация next-intl

**src/i18n/routing.ts**
```typescript
import { defineRouting } from 'next-intl/routing';

export const routing = defineRouting({
  locales: ['sr', 'en', 'ru'],
  defaultLocale: 'sr',
  localePrefix: 'always',
});
```

**src/i18n/navigation.ts**
```typescript
import { createNavigation } from 'next-intl/navigation';
import { routing } from './routing';

export const { Link, redirect, usePathname, useRouter } = createNavigation(routing);
```

**src/i18n.ts**
```typescript
import { getRequestConfig } from 'next-intl/server';

export default getRequestConfig(async ({ locale }) => ({
  messages: (await import(`./messages/${locale}.json`)).default,
}));
```

### Middleware

**src/middleware.ts**
```typescript
import createMiddleware from 'next-intl/middleware';
import { routing } from './i18n/routing';

export default createMiddleware(routing);

export const config = {
  matcher: ['/', '/(sr|en|ru)/:path*'],
};
```

---

## Структура переводов

### Файлы переводов

```
src/messages/
├── sr.json    # Сербский (7 KB)
├── en.json    # Английский
└── ru.json    # Русский
```

### Namespace структура (sr.json)

```json
{
  "nav": {
    "features": "Mogućnosti",
    "solutions": "Rešenja",
    "pricing": "Cene",
    "contact": "Kontakt",
    "demo": "Demo"
  },

  "hero": {
    "title": "AI Rešenja za Srbiju i Balkan",
    "subtitle": "Napredni AI asistenti za automatizaciju",
    "description": "VONDI AI pomaže državnoj upravi i biznisu...",
    "cta_demo": "Zakažite Demo",
    "cta_learn": "Saznajte Više"
  },

  "features": {
    "title": "Mogućnosti",
    "subtitle": "Šta VONDI AI može da uradi za vas",
    "items": {
      "search": {
        "title": "Pametna Pretraga",
        "description": "AI pretraga dokumenata i baza podataka"
      },
      "documents": {
        "title": "Obrada Dokumenata",
        "description": "Automatizacija administrativnih zadataka"
      },
      "chat": {
        "title": "AI Razgovori",
        "description": "Chatbot za korisnike i građane"
      },
      "analytics": {
        "title": "Analitika",
        "description": "Uvidi i izveštaji u realnom vremenu"
      }
    }
  },

  "how_it_works": {
    "title": "Kako Funkcioniše",
    "steps": {
      "1": { "title": "Integracija", "description": "Povezivanje sa vašim sistemima" },
      "2": { "title": "Konfiguracija", "description": "Prilagođavanje vašim potrebama" },
      "3": { "title": "Pokretanje", "description": "Lansiranje AI asistenta" },
      "4": { "title": "Podrška", "description": "Kontinuirana pomoć i unapređenja" }
    }
  },

  "solutions": {
    "title": "Rešenja",
    "government": {
      "title": "Za Državu",
      "description": "AI rešenja za opštine i javne službe",
      "features": ["Portal za građane", "Automatizacija zahteva", "Smart redovi"]
    },
    "enterprise": {
      "title": "Za Biznis",
      "description": "Korporativna AI automatizacija",
      "features": ["Korisnička podrška", "Interna automatizacija", "Analitika"]
    }
  },

  "pricing": {
    "title": "Cene",
    "subtitle": "Izaberite plan koji vam odgovara",
    "plans": {
      "starter": {
        "name": "Starter",
        "price": "€499",
        "period": "/mesec",
        "description": "Za male timove",
        "features": ["Do 1000 poruka/mesec", "Email podrška", "1 AI asistent"]
      },
      "business": {
        "name": "Business",
        "price": "€1499",
        "period": "/mesec",
        "popular": "POPULARAN",
        "description": "Za srednje kompanije",
        "features": ["Do 10000 poruka/mesec", "Prioritetna podrška", "5 AI asistenata"]
      },
      "enterprise": {
        "name": "Enterprise",
        "price": "Po zahtevu",
        "description": "Za velike organizacije",
        "features": ["Neograničeno poruka", "Dedicirani menadžer", "Custom integracije"]
      }
    }
  },

  "faq": {
    "title": "Česta Pitanja",
    "items": {
      "1": {
        "question": "Koliko vremena treba za implementaciju?",
        "answer": "Tipična implementacija traje 2-4 nedelje..."
      },
      "2": {
        "question": "Koje jezike podržavate?",
        "answer": "Podržavamo srpski, engleski i ruski..."
      },
      "3": {
        "question": "Gde se čuvaju podaci?",
        "answer": "Svi podaci se čuvaju u EU data centrima..."
      },
      "4": {
        "question": "Da li može da se integriše sa našim sistemom?",
        "answer": "Da, nudimo API i webhooks za integraciju..."
      }
    }
  },

  "cta": {
    "title": "Spremni za AI Transformaciju?",
    "subtitle": "Kontaktirajte nas danas",
    "button": "Pošaljite upit"
  },

  "try_ai": {
    "badge": "Probajte AI Demo",
    "title": "Isprobajte VONDI AI",
    "subtitle": "Interaktivni demo",
    "description": "Postavite pitanje i vidite kako AI radi",
    "expand": "Proširite na ceo ekran"
  },

  "chat": {
    "welcome": "Zdravo! Ja sam VONDI AI Asistent...",
    "placeholder": "Unesite vaše pitanje...",
    "send": "Pošalji",
    "thinking": "AI razmišlja...",
    "limit_reached": "Dostignut dnevni limit",
    "remaining": "Preostalo poruka"
  },

  "contact": {
    "title": "Kontakt",
    "subtitle": "Pošaljite nam poruku"
  },

  "form": {
    "name": "Ime i prezime",
    "email": "Email adresa",
    "phone": "Telefon",
    "company": "Kompanija",
    "request_type": "Tip zahteva",
    "message": "Poruka",
    "submit": "Pošalji",
    "submitting": "Šaljem...",
    "success": "Poruka je uspešno poslata!",
    "error": "Greška pri slanju"
  },

  "validation": {
    "name_required": "Ime je obavezno",
    "name_min": "Ime mora imati najmanje 2 karaktera",
    "email_required": "Email je obavezan",
    "email_invalid": "Unesite validan email",
    "phone_invalid": "Unesite validan broj telefona",
    "message_required": "Poruka je obavezna",
    "message_min": "Poruka mora imati najmanje 10 karaktera"
  },

  "footer": {
    "company": "VONDI GLOBAL D.O.O.",
    "location": "Beograd, Srbija",
    "description": "AI rešenja za automatizaciju",
    "copyright": "© {year} VONDI GLOBAL D.O.O. Sva prava zadržana."
  }
}
```

### Использование в компонентах

```typescript
'use client';

import { useTranslations } from 'next-intl';

export function HeroSection() {
  const t = useTranslations('hero');

  return (
    <section>
      <h1>{t('title')}</h1>
      <p>{t('subtitle')}</p>
      <p>{t('description')}</p>
      <button>{t('cta_demo')}</button>
    </section>
  );
}
```

### Динамические переводы

```typescript
// Для вложенных ключей
t('features.items.search.title')

// С параметрами
t('footer.copyright', { year: new Date().getFullYear() })

// Списки
const features = t.raw('solutions.government.features') as string[];
```

---

## SEO оптимизация

### Метаданные Layout

**src/app/[locale]/layout.tsx**

```typescript
import { Metadata } from 'next';

const seoConfig = {
  sr: {
    title: 'VONDI AI - AI Rešenja za Srbiju i Balkan',
    description: 'Napredni AI chatboti i automatizacija za državnu upravu i biznis. VONDI AI Solutions - vaš partner za digitalnu transformaciju.',
    keywords: [
      'AI chatbot Srbija',
      'AI automatizacija',
      'e-Uprava AI',
      'AI rešenja Beograd',
      'chatbot za biznis',
      'automatizacija dokumenata',
      'AI asistent srpski',
      'digitalna transformacija',
    ],
  },
  en: {
    title: 'VONDI AI - AI Solutions for Serbia and Balkans',
    description: 'Advanced AI chatbots and automation for government and business. VONDI AI Solutions - your partner for digital transformation.',
    keywords: [
      'AI chatbot Serbia',
      'AI automation Balkans',
      'e-Government AI',
      'AI solutions Belgrade',
      'business chatbot',
      'document automation',
      'AI assistant',
      'digital transformation',
    ],
  },
  ru: {
    title: 'VONDI AI - ИИ Решения для Сербии и Балкан',
    description: 'Продвинутые ИИ чатботы и автоматизация для государственных органов и бизнеса. VONDI AI Solutions.',
    keywords: [
      'ИИ чатбот Сербия',
      'автоматизация ИИ',
      'электронное правительство',
      'ИИ решения Белград',
      'чатбот для бизнеса',
      'автоматизация документов',
    ],
  },
};

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { locale } = await params;
  const seo = seoConfig[locale as keyof typeof seoConfig];
  const baseUrl = 'https://solutions.vondi.rs';

  return {
    title: {
      default: seo.title,
      template: `%s | VONDI AI`,
    },
    description: seo.description,
    keywords: seo.keywords,

    metadataBase: new URL(baseUrl),

    openGraph: {
      title: seo.title,
      description: seo.description,
      url: `${baseUrl}/${locale}`,
      siteName: 'VONDI AI Solutions',
      locale: locale === 'sr' ? 'sr_RS' : locale === 'ru' ? 'ru_RU' : 'en_US',
      type: 'website',
      images: [
        {
          url: `${baseUrl}/images/og-image-${locale}.png`,
          width: 1200,
          height: 630,
          alt: seo.title,
        },
      ],
    },

    twitter: {
      card: 'summary_large_image',
      title: seo.title,
      description: seo.description,
      images: [`${baseUrl}/images/og-image-${locale}.png`],
    },

    alternates: {
      canonical: `${baseUrl}/${locale}`,
      languages: {
        'sr': `${baseUrl}/sr`,
        'en': `${baseUrl}/en`,
        'ru': `${baseUrl}/ru`,
        'x-default': `${baseUrl}/sr`,
      },
    },

    robots: {
      index: true,
      follow: true,
      googleBot: {
        index: true,
        follow: true,
        'max-video-preview': -1,
        'max-image-preview': 'large',
        'max-snippet': -1,
      },
    },

    verification: {
      google: 'GOOGLE_VERIFICATION_CODE',
      yandex: 'YANDEX_VERIFICATION_CODE',
    },

    other: {
      'geo.region': 'RS',
      'geo.placename': 'Belgrade',
      'geo.position': '44.7866;20.4489',
      'ICBM': '44.7866, 20.4489',
    },
  };
}
```

---

### Structured Data (JSON-LD)

**src/components/seo/StructuredData.tsx**

#### Organization Schema

```json
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "VONDI GLOBAL D.O.O.",
  "legalName": "VONDI GLOBAL D.O.O. Beograd",
  "url": "https://solutions.vondi.rs",
  "logo": "https://solutions.vondi.rs/images/vondi-logo.png",
  "description": "AI rešenja za automatizaciju državne uprave i biznisa na Balkanu",
  "foundingDate": "2024",
  "address": {
    "@type": "PostalAddress",
    "addressCountry": "RS",
    "addressLocality": "Beograd",
    "addressRegion": "Srbija"
  },
  "contactPoint": {
    "@type": "ContactPoint",
    "email": "mail@vondi.rs",
    "contactType": "sales",
    "availableLanguage": ["Serbian", "English", "Russian"]
  },
  "areaServed": [
    { "@type": "Country", "name": "Serbia" },
    { "@type": "Country", "name": "Montenegro" },
    { "@type": "Country", "name": "Bosnia and Herzegovina" },
    { "@type": "Country", "name": "Croatia" },
    { "@type": "Country", "name": "Slovenia" },
    { "@type": "Country", "name": "North Macedonia" },
    { "@type": "Country", "name": "Albania" },
    { "@type": "Country", "name": "Kosovo" }
  ],
  "knowsLanguage": ["sr", "en", "ru"]
}
```

#### Website Schema

```json
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": "VONDI AI Solutions",
  "url": "https://solutions.vondi.rs",
  "description": "AI chatboti i automatizacija za Srbiju i Balkan",
  "inLanguage": ["sr", "en", "ru"],
  "potentialAction": {
    "@type": "SearchAction",
    "target": {
      "@type": "EntryPoint",
      "urlTemplate": "https://solutions.vondi.rs/search?q={search_term_string}"
    },
    "query-input": "required name=search_term_string"
  }
}
```

#### SoftwareApplication Schema

```json
{
  "@context": "https://schema.org",
  "@type": "SoftwareApplication",
  "name": "VONDI AI Assistant",
  "applicationCategory": "BusinessApplication",
  "applicationSubCategory": "AI Chatbot",
  "operatingSystem": "Web",
  "offers": [
    {
      "@type": "Offer",
      "name": "Starter",
      "price": "499",
      "priceCurrency": "EUR",
      "priceValidUntil": "2025-12-31"
    },
    {
      "@type": "Offer",
      "name": "Business",
      "price": "1499",
      "priceCurrency": "EUR",
      "priceValidUntil": "2025-12-31"
    }
  ],
  "featureList": [
    "AI document processing",
    "Multi-language support",
    "Government integration",
    "Enterprise automation"
  ]
}
```

#### LocalBusiness Schema

```json
{
  "@context": "https://schema.org",
  "@type": "LocalBusiness",
  "name": "VONDI GLOBAL D.O.O.",
  "image": "https://solutions.vondi.rs/images/vondi-logo.png",
  "url": "https://solutions.vondi.rs",
  "telephone": "",
  "email": "mail@vondi.rs",
  "address": {
    "@type": "PostalAddress",
    "streetAddress": "",
    "addressLocality": "Beograd",
    "addressRegion": "Srbija",
    "postalCode": "11000",
    "addressCountry": "RS"
  },
  "geo": {
    "@type": "GeoCoordinates",
    "latitude": 44.7866,
    "longitude": 20.4489
  },
  "openingHours": "Mo-Fr 09:00-17:00",
  "currenciesAccepted": "EUR, RSD",
  "paymentAccepted": "Bank Transfer, Credit Card"
}
```

#### FAQ Schema

```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Koliko vremena treba za implementaciju VONDI AI?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Tipična implementacija VONDI AI rešenja traje 2-4 nedelje, u zavisnosti od kompleksnosti zahteva i potrebnih integracija."
      }
    },
    {
      "@type": "Question",
      "name": "Koje jezike podržava VONDI AI?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "VONDI AI podržava srpski, engleski i ruski jezik, sa mogućnošću dodavanja dodatnih jezika po potrebi."
      }
    },
    {
      "@type": "Question",
      "name": "Gde se čuvaju podaci?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Svi podaci se čuvaju u EU data centrima sa punom usklađenošću sa GDPR regulativom."
      }
    },
    {
      "@type": "Question",
      "name": "Koliko košta VONDI AI?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "VONDI AI nudi tri plana: Starter od €499/mesec, Business od €1499/mesec, i Enterprise sa prilagođenim cenama."
      }
    }
  ]
}
```

---

### Sitemap

**src/app/sitemap.ts**

```typescript
import { MetadataRoute } from 'next';

export default function sitemap(): MetadataRoute.Sitemap {
  const baseUrl = 'https://solutions.vondi.rs';
  const locales = ['sr', 'en', 'ru'];

  const pages = [
    '',
    '/features',
    '/solutions',
    '/pricing',
    '/contact',
    '/about',
    '/demo',
    '/privacy',
    '/terms',
  ];

  const sitemap: MetadataRoute.Sitemap = [];

  for (const locale of locales) {
    for (const page of pages) {
      sitemap.push({
        url: `${baseUrl}/${locale}${page}`,
        lastModified: new Date(),
        changeFrequency: page === '' ? 'weekly' : 'monthly',
        priority: page === '' ? 1.0 : page === '/pricing' ? 0.9 : 0.8,
        alternates: {
          languages: {
            sr: `${baseUrl}/sr${page}`,
            en: `${baseUrl}/en${page}`,
            ru: `${baseUrl}/ru${page}`,
          },
        },
      });
    }
  }

  return sitemap;
}
```

---

### Open Graph изображения

**Файлы:**
```
public/images/
├── og-image-sr.png     # 1200x630, сербский текст
├── og-image-en.png     # 1200x630, английский текст
├── og-image-ru.png     # 1200x630, русский текст
└── og-image-final.png  # 1200x630, универсальный
```

**Спецификация:**
- Размер: 1200 × 630 px
- Формат: PNG
- Содержание: Логотип VONDI AI + заголовок + описание

---

### PWA Manifest

**public/manifest.json**

```json
{
  "name": "VONDI AI Solutions",
  "short_name": "VONDI AI",
  "description": "AI rešenja za automatizaciju",
  "start_url": "/sr",
  "display": "standalone",
  "background_color": "#1a1a2e",
  "theme_color": "#4a6cf7",
  "icons": [
    {
      "src": "/images/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png"
    },
    {
      "src": "/images/icon-96x96.png",
      "sizes": "96x96",
      "type": "image/png"
    },
    {
      "src": "/images/icon-128x128.png",
      "sizes": "128x128",
      "type": "image/png"
    },
    {
      "src": "/images/icon-144x144.png",
      "sizes": "144x144",
      "type": "image/png"
    },
    {
      "src": "/images/icon-152x152.png",
      "sizes": "152x152",
      "type": "image/png"
    },
    {
      "src": "/images/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/images/icon-384x384.png",
      "sizes": "384x384",
      "type": "image/png"
    },
    {
      "src": "/images/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

---

### Favicon

**Файлы:**
```
public/
├── favicon.ico          # 16x16, 32x32, 48x48 (multi-size)
├── favicon-16x16.png
├── favicon-32x32.png
└── apple-touch-icon.png # 180x180
```

**Подключение в layout:**
```typescript
export const metadata: Metadata = {
  icons: {
    icon: [
      { url: '/favicon-16x16.png', sizes: '16x16', type: 'image/png' },
      { url: '/favicon-32x32.png', sizes: '32x32', type: 'image/png' },
    ],
    apple: '/apple-touch-icon.png',
  },
};
```

---

## SEO Checklist

### Technical SEO

- [x] HTTPS enabled (Let's Encrypt)
- [x] Mobile-responsive design
- [x] Fast loading (< 3s LCP)
- [x] Proper heading hierarchy (H1 → H6)
- [x] Alt text for images
- [x] Sitemap.xml generated
- [x] robots.txt configured
- [x] Canonical URLs set
- [x] Hreflang tags for multi-language

### Content SEO

- [x] Unique title tags per page
- [x] Meta descriptions (150-160 chars)
- [x] Keywords in content
- [x] Internal linking
- [x] External linking (where appropriate)
- [x] Fresh content (FAQ, blog if added)

### Local SEO (Balkans)

- [x] LocalBusiness schema
- [x] Geo meta tags (Belgrade, Serbia)
- [x] Local keywords (srpski AI, AI Srbija)
- [x] Multi-language support (SR/EN/RU)
- [x] Region-specific content

### Structured Data

- [x] Organization schema
- [x] Website schema
- [x] SoftwareApplication schema
- [x] LocalBusiness schema
- [x] FAQ schema
- [x] BreadcrumbList schema

### Social Media

- [x] Open Graph tags
- [x] Twitter Card tags
- [x] OG images per locale
- [x] Social sharing buttons (if needed)

---

## Региональное SEO

### Целевые регионы

| Страна | Код | Язык | Приоритет |
|--------|-----|------|-----------|
| Сербия | RS | sr | Высший |
| Черногория | ME | sr | Высокий |
| Босния и Герцеговина | BA | sr | Высокий |
| Хорватия | HR | sr/en | Средний |
| Словения | SI | en | Средний |
| Северная Македония | MK | sr/en | Средний |
| Албания | AL | en | Низкий |
| Косово | XK | sr/en | Низкий |

### Ключевые слова по регионам

**Сербский (основной):**
- AI chatbot Srbija
- AI automatizacija Beograd
- e-Uprava AI rešenja
- chatbot za državnu upravu
- automatizacija dokumenata AI
- AI asistent srpski jezik

**Английский:**
- AI chatbot Serbia
- AI automation Balkans
- e-Government AI solutions
- AI business automation
- document processing AI

**Русский:**
- ИИ чатбот Сербия
- автоматизация ИИ Балканы
- искусственный интеллект Белград
- чатбот для бизнеса
