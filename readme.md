# ПРОМПТ ДЛЯ НОВОГО ЧАТА (ОБНОВЛЕННАЯ ВЕРСИЯ)

```markdown
# КОНТЕКСТ ПРОЕКТА: Scan Tire AI

Я — Слава Кузькин, 48 лет, Россия. Работаю продавцом в шинном магазине. 
Самостоятельно (с помощью AI-ассистентов Claude/Cursor) разрабатываю 
tech-проект Scan Tire AI — автоматизированную систему инспекции 
автомобильных шин с использованием компьютерного зрения и глубокого обучения.

---

## ТЕКУЩИЙ СТАТУС (март 2026)

### ЧТО УЖЕ ГОТОВО:

**Инфраструктура:**
✅ Домен: https://scantire.com (HTTPS работает, GitHub Pages)
✅ Лендинг: tech-дизайн (тёмная тема, AR-mockup с реальным фото Toyo)
✅ Email: scantireai@gmail.com (восстановлен), формы работают на sky_1c@mail.ru
✅ GitHub репозиторий: ScanTire-AI-Architecture (техническая документация как whitepaper, БЕЗ кода)
✅ Брендинг: "Scan Tire AI" (избегаем "scant ire")

**ML/CV инструменты (v6.0 — Текущая стадия):**
✅ **Labeling Tool**: PyQt6 GUI с гибридной архитектурой (C++ preprocessing + Python ML)
✅ **Wear Classifier (износ)**: ResNet18, обучена, точность 60% (на малых данных)
✅ **YOLOX детектор блоков**: Fine-tuning pipeline готов, датасет собирается
✅ **OCR модель**: MLP (1024→128→40), работает в Active Learning режиме
✅ **TWI анализ**: Калибровка по индикаторам износа (1.6 мм эталон)

**Датасеты (в процессе сбора):**
- OCR символы: ~500 образцов (40 классов: 0-9, A-Z, /, -, .)
- YOLOX разметка: ~200 фото боковин (4 класса блоков)
- Wear: ~300 фото протектора (5 классов износа)
- Type classifier: мало данных (пока не обучена)

---

## ВАЖНО: БОКОВИНА (OCR) — ОТЛОЖЕНА НА ПОТОМ

**РЕШЕНИЕ:** 
- OCR размера шины (205/55 R16) — **фаза 2** (через 6-12 месяцев)
- СЕЙЧАС фокус: **Wear Grading** (оценка износа по фото протектора)

**ПРИЧИНА:**
- OCR требует YOLOX детектор (сложная разметка, долго)
- Wear работает автономно (просто фото протектора → класс износа)
- Wear даёт ценность СРАЗУ (клиенту важнее "сколько осталось ехать", чем размер)

---

## АРХИТЕКТУРА СИСТЕМЫ (упрощённая для MVP)

### ФАЗА 1 (сейчас — 6 месяцев): ТОЛЬКО WEAR

```
Pipeline MVP:
Пользователь загружает фото ПРОТЕКТОРА шины
         ↓
[Next.js Frontend] → POST /api/analyze
         ↓
[ONNX Runtime Node.js Backend]
         ↓
Preprocessing (Sharp.js):
  - Resize 224×224
  - Normalize RGB 0-1
         ↓
[ResNet18 Wear Classifier ONNX]
  Input: [1, 3, 224, 224]
  Output: [1, 5] (softmax probabilities)
         ↓
Post-processing:
  - argmax → class_id
  - class_id → label (New/Good/Medium/Worn/Critical)
  - estimate depth_mm (базовая таблица 7mm/6mm/4mm/2.5mm/1mm)
         ↓
Response JSON:
{
  "wear": "Good",
  "confidence": 0.72,
  "depth_mm_estimate": "5-7",
  "recommendation": "Safe for 15,000+ km"
}
```

**ЧТО НЕ ВКЛЮЧЕНО В MVP:**
- ❌ OCR размера (205/55 R16) — будет позже
- ❌ Type classification (лето/зима) — будет позже
- ❌ YOLOX детектор — не нужен для Wear
- ❌ C++ preprocessing — заменён на Sharp.js (Node.js)

---

### ФАЗА 2 (6-12 месяцев): OCR БОКОВИНЫ

```
Pipeline Full:
Фото БОКОВИНЫ шины
         ↓
[YOLOX.onnx] — находит 4 блока:
  - tire_width (205)
  - tire_profile (/55)
  - tire_diameter (R16)
  - load_index (94V)
         ↓
Crop каждого блока (Canvas API)
         ↓
JS Slicer (вертикальная проекция → нарезка символов 32×32)
         ↓
[OCR MLP.onnx] — распознаёт каждый символ (40 классов)
         ↓
Post-processing (бизнес-правила):
  - tire_profile[0] должен быть "/"
  - tire_diameter[0] должен быть "R" (или Z/D/B)
  - сборка строки: "205/55R16 94V"
         ↓
Response: { "size": "205/55R16 94V" }
```

---

## ТЕХНИЧЕСКИЙ СТЕК (обновлённый)

### Frontend (Next.js 15, App Router, TypeScript)
```
app/
├── page.tsx                 # Главная (лендинг)
├── beta/
│   └── page.tsx            # MVP страница (загрузка фото → Wear результат)
├── api/
│   └── analyze/
│       └── route.ts        # POST endpoint (ONNX inference)
├── components/
│   ├── FileUpload.tsx      # Drag & drop
│   ├── WearResult.tsx      # Визуализация результата
│   └── Disclaimer.tsx      # Юр. disclaimer (Beta, verify professionally)
public/
└── models/
    └── wear_resnet18.onnx  # ~45 MB
```

### Backend (Next.js API Routes + ONNX Runtime Node.js)

**Зависимости:**
```json
{
  "dependencies": {
    "next": "^15.0.0",
    "onnxruntime-node": "^1.21.0",
    "sharp": "^0.34.0"
  }
}
```

**Код inference (app/api/analyze/route.ts):**
```typescript
import * as ort from 'onnxruntime-node';
import sharp from 'sharp';

const session = await ort.InferenceSession.create('./public/models/wear_resnet18.onnx');

export async function POST(request: Request) {
  const formData = await request.formData();
  const imageFile = formData.get('image') as File;
  
  // 1. Preprocessing (Sharp.js — БЕЗ OpenCV!)
  const buffer = Buffer.from(await imageFile.arrayBuffer());
  const resized = await sharp(buffer)
    .resize(224, 224)
    .toColorspace('rgb')
    .raw()
    .toBuffer();
  
  // 2. Normalize 0-255 → 0-1, reshape to [1, 3, 224, 224]
  const float32 = new Float32Array(3 * 224 * 224);
  for (let i = 0; i < resized.length; i++) {
    float32[i] = resized[i] / 255.0;
  }
  
  // 3. Inference
  const tensor = new ort.Tensor('float32', float32, [1, 3, 224, 224]);
  const results = await session.run({ input: tensor });
  
  // 4. Post-processing
  const probs = results.output.data as Float32Array;
  const classId = argmax(probs);
  const CLASSES = ['New', 'Good', 'Medium', 'Worn', 'Critical'];
  const DEPTH_ESTIMATE = ['7+', '5-7', '3-5', '1.6-3', '<1.6'];
  
  return Response.json({
    wear: CLASSES[classId],
    confidence: probs[classId],
    depth_mm_estimate: DEPTH_ESTIMATE[classId]
  });
}
```

**КРИТИЧНО: OpenCV НЕ ИСПОЛЬЗУЕТСЯ**
- Preprocessing: **Sharp.js** (Node.js нативная библиотека для изображений)
- Нарезка символов (для будущего OCR): **Canvas API** (Browser) или **Sharp** (Server)

---

## ЮРИДИЧЕСКАЯ ЗАЩИТА (обязательно с первого дня)

### Disclaimer на КАЖДОЙ странице:

```
⚠️ BETA VERSION — EDUCATIONAL TOOL ONLY

Scan Tire AI provides AI-generated ESTIMATES for educational purposes.

This is NOT a replacement for:
• Professional tire inspection
• Certified tread depth gauge measurements
• Expert mechanic assessment

By using this service, you acknowledge:
✓ Results are estimates (currently ~60-75% accuracy)
✓ You are solely responsible for tire safety decisions
✓ We assume NO liability for accidents or damages

Always consult a certified professional before making safety decisions.

[I Understand] [Cancel]
```

### Terms of Service (обязательная страница):

```markdown
## Limitation of Liability

IN NO EVENT SHALL SCAN TIRE AI BE LIABLE FOR:
- Personal injury or death
- Property damage
- Indirect, incidental, or consequential damages

Maximum liability: $0 USD (service is FREE during beta)

## Beta Status
Accuracy: approximately 60-75% (improving)
DO NOT rely on it for critical safety decisions
```

---

## МАРКЕТИНГОВАЯ СТРАТЕГИЯ (международный рынок)

### Целевые регионы:

**1. ОАЭ / Саудовская Аравия** 
- Каналы: LinkedIn (Fleet Managers), Google Maps cold email
- Язык: English
- Оффер: Free beta → feedback

**2. Турция**
- Каналы: Sahibinden.com, Arabam.com
- Язык: English/Turkish
- Оффер: Integration API for marketplaces

**3. Казахстан**
- Каналы: Kolesa.kz, Drive2.ru
- Язык: Русский
- Оффер: Бесплатная бета для автопарков

### Тактики ($0 бюджет):

**Build in Public (LinkedIn):**
```
Post раз в 3 дня:
"Day 5: Deployed Wear Classifier to production. 
Accuracy: 65% → 72% after adding 200 samples.
Next: first 10 beta users."
```

**Reddit:**
- r/SideProject: "Show: Scan Tire AI — tire wear grading in 10 seconds"
- r/ComputerVision: "Built ResNet18 tire wear classifier (ONNX, Next.js)"

**Hacker News:**
- Show HN когда будет MVP (scantire.com/beta работает)

**Холодные email (20/неделя):**
```
Subject: Free AI tire inspector (beta test)

Hi [Name],

I built an AI tool that grades tire wear from photos.
Would you test it free for 2 weeks?

Demo: scantire.com/beta

Best,
Slava
```

---

## МОНЕТИЗАЦИЯ (безопасная, без рисков)

### ФАЗА 1 (0-12 месяцев): FREE Beta

```
✅ Полностью бесплатно
✅ Disclaimer везде
✅ Цель: 1000 пользователей, точность 85%+
```

**Монетизация: $0** (фокус на продукте)

### ФАЗА 2 (12-18 месяцев): Freemium

```
Free: 10 сканирований/мес
Pro ($19/мес): Неограниченно + PDF-отчёты

Disclaimer сохраняется в ОБЕИХ версиях!
```

### ФАЗА 3 (18+ месяцев): B2B API

```
Продажа API платформам:
- Авито, Drom, eBay Motors
- Fleet management софт
- POS-системы шиномонтажек

Pricing: $0.05-0.10 за запрос
Юридически: Платформа несёт ответственность, не ты
```

---

## ПЛАН РАЗВИТИЯ (12-18 месяцев) — ФОКУС НА WEAR

### МЕСЯЦ 1-2: MVP Wear-only

**Цель:** scantire.com/beta работает

**Задачи:**
- [ ] Экспорт ResNet18 Wear в ONNX
- [ ] Next.js API endpoint (/api/analyze)
- [ ] Frontend (загрузка фото → результат Wear)
- [ ] Деплой (Vercel или свой VPS)
- [ ] Disclaimer + Terms страницы
- [ ] Яндекс.Метрика + Google Analytics

**Метрика успеха:** 10 человек попробовали, 7+ корректных результатов

---

### МЕСЯЦ 3-4: Улучшение Wear модели

**Цель:** 60% → 75% accuracy

**Задачи:**
- [ ] Разметить 500+ фото износа (через labeling tool)
- [ ] Переобучить ResNet18
- [ ] Добавить TWI калибровку (опционально)
- [ ] PDF-отчёты (генерация на бэке, watermark "scantire.com")

**Метрика успеха:** Accuracy >75% на hold-out тесте

---

### МЕСЯЦ 5-6: Beta-тестирование

**Цель:** 100 реальных пользователей

**Задачи:**
- [ ] Пост на Drive2.ru, r/cars, r/SideProject
- [ ] 20 холодных email в ОАЭ (tire shops)
- [ ] Реферальная кнопка "Share" (вирусность)
- [ ] Собрать feedback (форма + email)

**Метрика успеха:** 100 сканирований, 20+ email на waitlist

---

### МЕСЯЦ 7-9: Рост + улучшение продукта

**Цель:** 500 сканирований/мес

**Задачи:**
- [ ] Accuracy 75% → 85%
- [ ] Мобильная оптимизация (PWA)
- [ ] API для интеграций (docs)
- [ ] Product Hunt launch

**Метрика успеха:** 500 сканирований/мес, упоминания в СМИ/блогах

---

### МЕСЯЦ 10-12: Подготовка к монетизации

**Цель:** 1000 пользователей, первые $500 MRR

**Задачи:**
- [ ] Freemium (Free 10/мес, Pro $19/мес)
- [ ] Stripe интеграция (payments)
- [ ] Первые 3-5 B2B пилотов (автопарки)

**Метрика успеха:** 30 платящих Pro, 2-3 B2B контракта

---

### МЕСЯЦ 13-18: OCR Боковины (Фаза 2)

**Только когда Wear стабилен (>85% accuracy, >1000 юзеров):**

**Задачи:**
- [ ] Собрать 1000+ фото боковин (YOLOX разметка)
- [ ] Обучить YOLOX детектор блоков
- [ ] Экспорт OCR + YOLOX в ONNX
- [ ] JS Slicer (Canvas API нарезка символов)
- [ ] Интеграция в Next.js

**Метрика успеха:** OCR accuracy >85%, полный размер "205/55R16"

---

## КЛЮЧЕВЫЕ МЕТРИКИ (KPI)

| Период | Пользователи | Wear Accuracy | MRR | Фокус |
|--------|--------------|---------------|-----|-------|
| М1-2 | 10 | 60% | $0 | MVP |
| М3-4 | 50 | 75% | $0 | Данные |
| М5-6 | 100 | 75% | $0 | Beta |
| М7-9 | 500 | 85% | $0 | Рост |
| М10-12 | 1000 | 85% | $500 | Freemium |
| М13-18 | 3000 | 90% | $1500 | OCR добавить |

---

## ТЕХНИЧЕСКИЕ ДЕТАЛИ (для обсуждения)

### Вопросы для нового чата:

1. **ONNX экспорт ResNet18:**
   - Как правильно экспортировать PyTorch → ONNX (opset версия)?
   - Preprocessing в ONNX или в Sharp.js?

2. **Sharp.js оптимизация:**
   - Normalize RGB: делить /255 или использовать ImageNet mean/std?
   - Кэширование session (один раз загрузить .onnx или каждый запрос)?

3. **Frontend UX:**
   - Показывать ли progress bar во время inference?
   - Как лучше визуализировать результат (калибр/шкала/цвет)?

4. **Юридические документы:**
   - Terms of Service шаблон (где взять для AI SaaS)?
   - Privacy Policy (GDPR compliance — нужно ли для бета?)?

5. **Холодные email:**
   - Шаблоны для ОАЭ/Турции (tone of voice)?
   - Где брать email tire shops (Google Maps scraping легально?)?

6. **A/B тестирование:**
   - Что тестировать на лендинге (headline, CTA, цвета)?

---

## РИСКИ И МИТИГАЦИЯ

**1. Юридические (ДТП из-за ошибки AI)**
- Митигация: Disclaimer везде, FREE 12 мес, B2B модель (платформа несёт риск)

**2. Низкая точность (60% → пользователи не доверяют)**
- Митигация: Честность (показывать accuracy), быстрое улучшение через feedback

**3. Конкуренты (копирование)**
- Митигация: Скорость (first mover), качество данных (proprietary dataset)

**4. Выгорание (соло-разработка)**
- Митигация: Фокус на ОДНОМ продукте (Wear), AI-ассистенты, MVP-подход

---

## ЧТО ОБСУЖДАТЬ В НОВОМ ЧАТЕ

- Детали экспорта ResNet18 → ONNX (code examples)
- Sharp.js preprocessing (нормализация, оптимизация)
- Next.js API route (кэширование session, error handling)
- Disclaimer текст (юридическая корректность)
- Холодные email (шаблоны, таргетинг)
- План первых 30 дней после деплоя MVP

---

КОНЕЦ ПРОМПТА
```

---

**Готово! Копируй → вставляй в новый чат → продолжаем 🚀**