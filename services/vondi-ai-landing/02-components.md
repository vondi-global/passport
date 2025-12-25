# Frontend компоненты

> Часть паспорта [vondi-ai-landing](./README.md)

## Обзор архитектуры компонентов

```
src/components/
├── layout/           # Глобальные компоненты макета
│   ├── Header.tsx
│   ├── Footer.tsx
│   └── LanguageSwitcher.tsx
│
├── sections/         # Секции главной страницы
│   ├── HeroSection.tsx
│   ├── FeaturesSection.tsx
│   ├── HowItWorksSection.tsx
│   ├── TryAISection.tsx
│   ├── SolutionsSection.tsx
│   ├── PricingSection.tsx
│   ├── FAQSection.tsx
│   ├── CTASection.tsx
│   ├── TestimonialsSection.tsx
│   ├── UseCasesSection.tsx
│   └── index.ts
│
├── chat/             # AI чат компоненты
│   ├── ChatWindow.tsx
│   ├── ChatMessage.tsx
│   ├── ChatInput.tsx
│   ├── SuggestedQuestions.tsx
│   └── AIDemoWidget.tsx
│
├── forms/            # Формы
│   └── ContactForm.tsx
│
├── ui/               # Базовые UI компоненты
│   ├── Button.tsx
│   ├── Card.tsx
│   ├── Badge.tsx
│   ├── Input.tsx
│   └── index.ts
│
└── seo/              # SEO компоненты
    └── StructuredData.tsx
```

---

## Layout компоненты

### Header.tsx

**Тип:** Client Component (`'use client'`)
**Назначение:** Главная навигация сайта

**Функционал:**
- Sticky позиционирование с blur-эффектом при скролле
- Логотип VONDI AI с градиентом
- Навигационное меню (desktop/mobile)
- Переключатель языков (SR/EN/RU)
- CTA кнопка "Demo"

**State:**
```typescript
const [mobileMenuOpen, setMobileMenuOpen] = useState(false);
const isScrolled = useScrollAnimation(50); // порог 50px
```

**Навигационные ссылки:**
- Features (`/{locale}/features`)
- Solutions (`/{locale}/solutions`)
- Pricing (`/{locale}/pricing`)
- Contact (`/{locale}/contact`)

**Стили при скролле:**
```css
/* isScrolled === true */
background: rgba(26, 26, 46, 0.95);
backdrop-filter: blur(8px);
border-bottom: 1px solid rgba(74, 108, 247, 0.1);
```

---

### Footer.tsx

**Тип:** Client Component
**Назначение:** Подвал сайта с информацией и навигацией

**Структура (4 колонки):**
1. **VONDI AI** - логотип, описание компании
2. **Solutions** - Government AI, Enterprise AI, Custom Solutions
3. **Company** - About, Contact, Privacy, Terms
4. **Resources** - Documentation, API Reference, Support

**Нижняя панель:**
- Copyright: © {year} VONDI GLOBAL D.O.O.
- Privacy Policy | Terms of Service
- Geo-тег: Beograd, Srbija

---

### LanguageSwitcher.tsx

**Тип:** Client Component
**Назначение:** Переключение языка UI

**Props:** Нет

**Поддерживаемые языки:**
| Код | Название | Флаг |
|-----|----------|------|
| `sr` | Serbian | 🇷🇸 |
| `en` | English | 🇬🇧 |
| `ru` | Russian | 🇷🇺 |

**Реализация:**
```typescript
const currentLocale = useLocale();
const pathname = usePathname();

// Переключение сохраняет текущий путь
<Link href={pathname} locale="en">EN</Link>
```

---

## Section компоненты

### HeroSection.tsx

**Назначение:** Главная привлекающая секция с ключевым предложением

**Структура:**
```
┌─────────────────────────────────────────┐
│  Background Image (hero-background.jpg) │
│  ┌─────────────────────────────────┐    │
│  │  Заголовок (gradient text)      │    │
│  │  Подзаголовок                   │    │
│  │  Описание                       │    │
│  │                                 │    │
│  │  [Request Demo] [Learn More]    │    │
│  │                                 │    │
│  │  ┌────┐ ┌────┐ ┌────┐          │    │
│  │  │Stat│ │Stat│ │Stat│          │    │
│  │  └────┘ └────┘ └────┘          │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

**Статистика:**
- Projects (TrendingUp icon)
- SLA 99.9% (Shield icon)
- 24/7 Support (Headphones icon)

**Анимации (Framer Motion):**
```typescript
const containerVariants = {
  hidden: { opacity: 0 },
  visible: {
    opacity: 1,
    transition: { staggerChildren: 0.2 }
  }
};

const itemVariants = {
  hidden: { opacity: 0, y: 30 },
  visible: { opacity: 1, y: 0 }
};
```

---

### FeaturesSection.tsx

**Назначение:** Презентация 4 ключевых возможностей AI

**Функции:**
1. **Smart Search** (Search icon) - blue→cyan gradient
2. **Document Processing** (FileText icon) - purple→pink gradient
3. **AI Conversations** (MessageSquare icon) - green→emerald gradient
4. **Analytics** (BarChart3 icon) - orange→red gradient

**Card структура:**
```
┌──────────────────┐
│  ┌──┐            │
│  │🔍│  Icon      │
│  └──┘            │
│  Title           │
│  Description     │
│  text...         │
└──────────────────┘
```

**Hover эффект:**
- Shadow: `md` → `xl`
- Transform: `translateY(-4px)`

---

### TryAISection.tsx

**Назначение:** Интерактивная демонстрация AI чата

**Layout (2 колонки на desktop):**
```
┌─────────────────┬─────────────────┐
│  Text Content   │   ChatWindow    │
│                 │                 │
│  Badge          │   ┌─────────┐   │
│  Title          │   │ Chat    │   │
│  Description    │   │ UI      │   │
│                 │   │         │   │
│  [Full Screen]  │   └─────────┘   │
└─────────────────┴─────────────────┘
```

**Background:** `accent-500/85` с декоративными сферами

**Компоненты:**
- `<ChatWindow showHeader={false} />`
- Button "Expand to Full Screen" → `/{locale}/demo`

---

### HowItWorksSection.tsx

**Назначение:** 4-шаговый процесс внедрения

**Шаги:**
1. **Integration** (Link icon) - blue
2. **Configuration** (Settings icon) - purple
3. **Launch** (Rocket icon) - green
4. **Support** (Headphones icon) - orange

**Timeline визуализация:**
```
  ①───────②───────③───────④
  │       │       │       │
 Step    Step    Step    Step
```

---

### SolutionsSection.tsx

**Назначение:** Решения для разных секторов

**Карточки:**

1. **Government AI** (Building2 icon)
   - Gradient: blue→indigo
   - Функции: Citizen portals, Document automation, Smart queuing
   - CTA: "Learn More" → `/{locale}/solutions#government`

2. **Enterprise AI** (Briefcase icon)
   - Gradient: purple→pink
   - Функции: Customer support, Internal automation, Analytics
   - CTA: "Learn More" → `/{locale}/solutions#enterprise`

---

### PricingSection.tsx

**Назначение:** Тарифные планы

**Планы:**

| План | Цена | Особенности |
|------|------|-------------|
| **Starter** | €499/mo | Basic features, Email support |
| **Business** | €1499/mo | Advanced features, Priority support, **ПОПУЛЯРЕН** |
| **Enterprise** | Custom | Full customization, Dedicated manager |

**Business план выделен:**
- Badge "ПОПУЛЯРЕН"
- Scale: `scale-105`
- Border: `border-2 border-accent-500`
- Shadow: `shadow-xl`

---

### FAQSection.tsx

**Назначение:** Аккордион с FAQ

**State:**
```typescript
const [openIndex, setOpenIndex] = useState<number | null>(0);
```

**Вопросы (примеры):**
1. Сколько времени занимает внедрение?
2. Какие языки поддерживаются?
3. Где хранятся данные?
4. Можно ли интегрировать с существующими системами?

**Анимация раскрытия:**
```typescript
<motion.div
  initial={{ height: 0 }}
  animate={{ height: 'auto' }}
  exit={{ height: 0 }}
  transition={{ duration: 0.3 }}
>
```

---

### CTASection.tsx

**Назначение:** Финальный призыв к действию

**Структура:** Аналогична HeroSection с фоновым изображением

**CTA кнопка:** "Get in Touch" → `/{locale}/contact`

---

## Chat компоненты

### ChatWindow.tsx

**Тип:** Client Component
**Назначение:** Главный контейнер AI чата

**Props:**
```typescript
interface ChatWindowProps {
  onClose?: () => void;
  showHeader?: boolean; // default: true
}
```

**State:**
```typescript
const [messages, setMessages] = useState<Message[]>([]);
const [isLoading, setIsLoading] = useState(false);
const [remaining, setRemaining] = useState(10);
```

**API взаимодействие:**
```typescript
const response = await fetch('/api/chat', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    message: userMessage,
    conversationHistory: messages.slice(-10),
    locale: currentLocale,
  }),
});
```

**Приветственные сообщения:**
```typescript
const welcomeMessages = {
  sr: 'Здраво! Ја сам VONDI AI Асистент...',
  en: 'Hello! I am VONDI AI Assistant...',
  ru: 'Здравствуйте! Я VONDI AI Assistant...',
};
```

**Компоненты:**
- `<ChatMessage />` - для каждого сообщения
- `<ChatInput />` - поле ввода
- `<SuggestedQuestions />` - предложенные вопросы (если ≤2 сообщений)

---

### ChatMessage.tsx

**Назначение:** Отображение одного сообщения

**Props:**
```typescript
interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
}
```

**Стили по роли:**

| Role | Выравнивание | Фон | Аватар |
|------|-------------|-----|--------|
| `user` | Справа | `accent-500` | User icon |
| `assistant` | Слева | `gray-100` | Bot icon |

**Анимация появления:**
```typescript
<motion.div
  initial={{ opacity: 0, y: 10 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: 0.3 }}
>
```

---

### ChatInput.tsx

**Назначение:** Поле ввода сообщения

**Props:**
```typescript
interface ChatInputProps {
  onSend: (message: string) => void;
  disabled?: boolean;
  remaining?: number; // default: 10
}
```

**Валидация:**
- Max length: 500 символов
- Empty check
- Disabled state при лимите

**Keyboard handling:**
- Enter → отправка
- Shift+Enter → новая строка (если поддерживается)

**Placeholder:**
```typescript
placeholder={remaining <= 0
  ? t('chat.limit_reached')
  : t('chat.placeholder')}
```

---

### SuggestedQuestions.tsx

**Назначение:** Предложенные вопросы для начала диалога

**Props:**
```typescript
interface SuggestedQuestionsProps {
  onSelect: (question: string) => void;
}
```

**Вопросы по языкам:**

**Serbian (sr):**
- "Kako AI može pomoći u obradi zahteva građana?"
- "Koje su prednosti automatizacije dokumenata?"
- "Koliko vremena treba za implementaciju?"

**English (en):**
- "How can AI help with citizen requests?"
- "What are the benefits of document automation?"
- "How long does implementation take?"

**Russian (ru):**
- "Как AI может помочь с обращениями граждан?"
- "Какие преимущества автоматизации документов?"
- "Сколько времени занимает внедрение?"

---

### AIDemoWidget.tsx

**Назначение:** Плавающая кнопка чата (появляется на всех страницах)

**State:**
```typescript
const [isOpen, setIsOpen] = useState(false);
```

**Кнопка (закрыто):**
- Position: `fixed bottom-6 right-6 z-50`
- Background: `accent-500`
- Icon: MessageCircle
- Text: "Try AI Demo" (скрыт на мобильных)

**Chat окно (открыто):**
- Size: `400px × 600px` (desktop)
- Size: `100vw × 100vh` (mobile)
- Shadow: `shadow-2xl`
- Border radius: `2xl`

**Анимации:**
```typescript
// Кнопка
whileHover={{ scale: 1.05 }}
whileTap={{ scale: 0.95 }}

// Окно
initial={{ opacity: 0, scale: 0.95, y: 20 }}
animate={{ opacity: 1, scale: 1, y: 0 }}
exit={{ opacity: 0, scale: 0.95, y: 20 }}
```

---

## Form компоненты

### ContactForm.tsx

**Назначение:** Форма обратной связи

**Библиотеки:**
- React Hook Form
- Zod для валидации
- @hookform/resolvers

**Поля:**

| Поле | Тип | Валидация |
|------|-----|-----------|
| `name` | text | Required, 2-100 chars |
| `email` | email | Required, valid email |
| `phone` | tel | Optional, regex pattern |
| `company` | text | Optional, max 200 chars |
| `requestType` | select | Required, enum |
| `message` | textarea | Required, 10-2000 chars |

**Request Types:**
- `demo` - Запросить демо
- `pricing` - Узнать цены
- `support` - Техподдержка
- `other` - Другое

**Zod Schema:**
```typescript
const contactSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
  phone: z.string().optional().refine(
    (val) => !val || /^[\d\s\-+()]+$/.test(val),
    'Invalid phone format'
  ),
  company: z.string().optional().max(200),
  requestType: z.enum(['demo', 'pricing', 'support', 'other']),
  message: z.string().min(10).max(2000),
});
```

**State:**
```typescript
const [isSubmitting, setIsSubmitting] = useState(false);
const [submitStatus, setSubmitStatus] = useState<{
  type: 'success' | 'error' | null;
  message: string;
}>({ type: null, message: '' });
```

---

## UI компоненты

### Button.tsx

**Props:**
```typescript
interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'outline' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
}
```

**Варианты стилей:**

| Variant | Background | Text | Border |
|---------|-----------|------|--------|
| `primary` | accent-500 | white | none |
| `secondary` | primary-700 | white | none |
| `outline` | transparent | white | white/30 |
| `ghost` | transparent | gray-700 | none |

**Размеры:**

| Size | Padding | Font |
|------|---------|------|
| `sm` | px-3 py-1.5 | text-sm |
| `md` | px-6 py-3 | text-base |
| `lg` | px-8 py-4 | text-lg |

---

### Card.tsx

**Props:**
```typescript
interface CardProps {
  children: ReactNode;
  className?: string;
  hover?: boolean; // default: false
}
```

**Базовые стили:**
```css
bg-white rounded-lg shadow-md p-6
```

**Hover эффект (hover=true):**
```css
hover:shadow-xl hover:-translate-y-1 hover:border-accent-500
transition-all duration-300
```

---

### Badge.tsx

**Props:**
```typescript
interface BadgeProps {
  children: ReactNode;
  variant?: 'default' | 'success' | 'warning' | 'info';
  className?: string;
}
```

**Варианты:**

| Variant | Background | Text |
|---------|-----------|------|
| `default` | gray-100 | gray-800 |
| `success` | green-100 | green-800 |
| `warning` | orange-100 | orange-800 |
| `info` | blue-100 | blue-800 |

---

### Input.tsx

**Props:**
```typescript
interface InputProps extends InputHTMLAttributes<HTMLInputElement> {
  label?: string;
  error?: string;
}
```

**Стили:**
```css
/* Базовые */
w-full px-4 py-2 border border-gray-300 rounded-lg

/* Focus */
focus:ring-2 focus:ring-accent-500 focus:border-transparent

/* Error */
border-red-500 focus:ring-red-500
```

**ForwardRef:** Поддерживает ref для интеграции с React Hook Form

---

## SEO компоненты

### StructuredData.tsx

**Тип:** Server Component
**Назначение:** JSON-LD structured data для поисковых систем

**Props:**
```typescript
interface StructuredDataProps {
  locale: string;
}
```

**Генерируемые схемы:**

1. **Organization** - информация о компании VONDI GLOBAL D.O.O.
2. **Website** - информация о сайте с SearchAction
3. **SoftwareApplication** - VONDI AI Assistant как продукт
4. **LocalBusiness** - локальный бизнес в Белграде
5. **FAQ** - часто задаваемые вопросы
6. **BreadcrumbList** - навигационная цепочка

**Пример вывода:**
```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "VONDI GLOBAL D.O.O.",
  "url": "https://solutions.vondi.rs",
  "logo": "https://solutions.vondi.rs/images/vondi-logo.png",
  ...
}
</script>
```

---

## Хуки

### useScrollAnimation.ts

**Назначение:** Отслеживание скролла для анимаций

```typescript
function useScrollAnimation(threshold: number = 50): boolean {
  const [isScrolled, setIsScrolled] = useState(false);

  useEffect(() => {
    const handleScroll = () => {
      setIsScrolled(window.scrollY > threshold);
    };
    window.addEventListener('scroll', handleScroll);
    return () => window.removeEventListener('scroll', handleScroll);
  }, [threshold]);

  return isScrolled;
}
```

**Использование:**
```typescript
const isScrolled = useScrollAnimation(50);
// true если scrollY > 50px
```

---

### useIntersectionObserver.ts

**Назначение:** Отслеживание видимости элементов для lazy-анимаций

```typescript
function useIntersectionObserver(
  options?: IntersectionObserverInit
): [RefObject<HTMLElement>, boolean] {
  const ref = useRef<HTMLElement>(null);
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        setIsVisible(true);
        observer.disconnect();
      }
    }, options);

    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, [options]);

  return [ref, isVisible];
}
```

**Использование:**
```typescript
const [ref, isVisible] = useIntersectionObserver({ threshold: 0.1 });
// isVisible = true когда 10% элемента в viewport
```

---

## Экспорт компонентов

### sections/index.ts
```typescript
export { HeroSection } from './HeroSection';
export { FeaturesSection } from './FeaturesSection';
export { HowItWorksSection } from './HowItWorksSection';
export { TryAISection } from './TryAISection';
export { SolutionsSection, UseCasesSection } from './SolutionsSection';
export { PricingSection } from './PricingSection';
export { FAQSection } from './FAQSection';
export { CTASection } from './CTASection';
export { TestimonialsSection } from './TestimonialsSection';
```

### ui/index.ts
```typescript
export { Button } from './Button';
export { Card } from './Card';
export { Badge } from './Badge';
export { Input } from './Input';
```
