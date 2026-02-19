# GameMarket - Маркетплейс игровых товаров

Полнофункциональный маркетплейс для продажи игровых товаров и услуг, объединяющий лучшие функции FunPay и PlayerOK.

## Функционал

- ✅ Авторизация через Google OAuth и Email/Password
- ✅ Система чатов между пользователями (Realtime)
- ✅ Создание и управление лотами
- ✅ Категории и подкатегории товаров
- ✅ Система заказов с уникальными номерами
- ✅ Финансовый учет (пополнение, вывод, история)
- ✅ Топ продавцов
- ✅ Профили пользователей
- ✅ Темная/светлая тема
- ✅ Адаптивный дизайн

## Технологический стек

- **Frontend**: React + TypeScript + Vite + Tailwind CSS + shadcn/ui
- **Backend**: Supabase (PostgreSQL + Auth + Realtime + Storage)
- **Иконки**: Lucide React

## Настройка проекта

### 1. Создание проекта в Supabase

1. Перейдите на [supabase.com](https://supabase.com) и создайте аккаунт
2. Создайте новый проект
3. Сохраните **Project URL** и **anon/public key** (Настройки -> API)

### 2. Настройка базы данных

В SQL Editor выполните следующие запросы:

```sql
-- Таблица профилей
CREATE TABLE profiles (
  id UUID REFERENCES auth.users ON DELETE CASCADE PRIMARY KEY,
  username TEXT NOT NULL UNIQUE,
  avatar_url TEXT,
  email TEXT NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  last_online TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  nickname_changed_at TIMESTAMP WITH TIME ZONE,
  balance DECIMAL(10, 2) DEFAULT 0
);

-- Таблица категорий
CREATE TABLE categories (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  slug TEXT NOT NULL UNIQUE,
  icon TEXT,
  type TEXT NOT NULL CHECK (type IN ('game', 'app', 'service')),
  parent_id UUID REFERENCES categories(id),
  "order" INTEGER DEFAULT 0,
  image_url TEXT,
  is_new BOOLEAN DEFAULT FALSE
);

-- Таблица подкатегорий
CREATE TABLE subcategories (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  category_id UUID NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  slug TEXT NOT NULL,
  icon TEXT,
  UNIQUE(category_id, slug)
);

-- Таблица лотов
CREATE TABLE lots (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  seller_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  category_id UUID NOT NULL REFERENCES categories(id),
  subcategory_id UUID REFERENCES subcategories(id),
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  price DECIMAL(10, 2) NOT NULL,
  old_price DECIMAL(10, 2),
  quantity INTEGER DEFAULT 1,
  status TEXT DEFAULT 'active' CHECK (status IN ('active', 'sold', 'archived')),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  images TEXT[] DEFAULT '{}'
);

-- Таблица заказов
CREATE TABLE orders (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  order_number TEXT NOT NULL UNIQUE,
  buyer_id UUID NOT NULL REFERENCES profiles(id),
  seller_id UUID NOT NULL REFERENCES profiles(id),
  lot_id UUID NOT NULL REFERENCES lots(id),
  amount DECIMAL(10, 2) NOT NULL,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'paid', 'completed', 'cancelled', 'disputed')),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  completed_at TIMESTAMP WITH TIME ZONE
);

-- Таблица транзакций
CREATE TABLE transactions (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  type TEXT NOT NULL CHECK (type IN ('deposit', 'withdrawal', 'payment', 'refund')),
  amount DECIMAL(10, 2) NOT NULL,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'completed', 'failed', 'cancelled')),
  payment_method TEXT,
  description TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  order_id UUID REFERENCES orders(id)
);

-- Таблица чатов
CREATE TABLE chats (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  participant_1 UUID NOT NULL REFERENCES profiles(id),
  participant_2 UUID NOT NULL REFERENCES profiles(id),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  last_message_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE(participant_1, participant_2)
);

-- Таблица сообщений
CREATE TABLE messages (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  chat_id UUID NOT NULL REFERENCES chats(id) ON DELETE CASCADE,
  sender_id UUID NOT NULL REFERENCES profiles(id),
  content TEXT NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  is_read BOOLEAN DEFAULT FALSE
);

-- Таблица отзывов
CREATE TABLE reviews (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  order_id UUID NOT NULL REFERENCES orders(id),
  reviewer_id UUID NOT NULL REFERENCES profiles(id),
  reviewed_id UUID NOT NULL REFERENCES profiles(id),
  rating INTEGER NOT NULL CHECK (rating >= 1 AND rating <= 5),
  comment TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Функция для генерации номера заказа
CREATE OR REPLACE FUNCTION generate_order_number()
RETURNS TEXT AS $$
BEGIN
  RETURN 'ORD-' || TO_CHAR(NOW(), 'YYYYMMDD') || '-' || UPPER(SUBSTRING(MD5(RANDOM()::TEXT), 1, 6));
END;
$$ LANGUAGE plpgsql;

-- Триггер для автоматического создания профиля
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, username, email, avatar_url)
  VALUES (
    NEW.id,
    COALESCE(NEW.raw_user_meta_data->>'username', 'user_' || SUBSTRING(NEW.id::TEXT, 1, 8)),
    NEW.email,
    NEW.raw_user_meta_data->>'avatar_url'
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();

-- RLS Policies
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE categories ENABLE ROW LEVEL SECURITY;
ALTER TABLE subcategories ENABLE ROW LEVEL SECURITY;
ALTER TABLE lots ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE transactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE chats ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE reviews ENABLE ROW LEVEL SECURITY;

-- Profiles policies
CREATE POLICY "Profiles are viewable by everyone" ON profiles
  FOR SELECT USING (true);

CREATE POLICY "Users can update own profile" ON profiles
  FOR UPDATE USING (auth.uid() = id);

-- Categories policies
CREATE POLICY "Categories are viewable by everyone" ON categories
  FOR SELECT USING (true);

-- Subcategories policies
CREATE POLICY "Subcategories are viewable by everyone" ON subcategories
  FOR SELECT USING (true);

-- Lots policies
CREATE POLICY "Lots are viewable by everyone" ON lots
  FOR SELECT USING (true);

CREATE POLICY "Users can create lots" ON lots
  FOR INSERT WITH CHECK (auth.uid() = seller_id);

CREATE POLICY "Users can update own lots" ON lots
  FOR UPDATE USING (auth.uid() = seller_id);

CREATE POLICY "Users can delete own lots" ON lots
  FOR DELETE USING (auth.uid() = seller_id);

-- Orders policies
CREATE POLICY "Users can view own orders" ON orders
  FOR SELECT USING (auth.uid() = buyer_id OR auth.uid() = seller_id);

CREATE POLICY "Users can create orders" ON orders
  FOR INSERT WITH CHECK (auth.uid() = buyer_id);

-- Transactions policies
CREATE POLICY "Users can view own transactions" ON transactions
  FOR SELECT USING (auth.uid() = user_id);

-- Chats policies
CREATE POLICY "Users can view own chats" ON chats
  FOR SELECT USING (auth.uid() = participant_1 OR auth.uid() = participant_2);

CREATE POLICY "Users can create chats" ON chats
  FOR INSERT WITH CHECK (auth.uid() = participant_1 OR auth.uid() = participant_2);

-- Messages policies
CREATE POLICY "Users can view messages in own chats" ON messages
  FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM chats 
      WHERE chats.id = messages.chat_id 
      AND (chats.participant_1 = auth.uid() OR chats.participant_2 = auth.uid())
    )
  );

CREATE POLICY "Users can send messages" ON messages
  FOR INSERT WITH CHECK (auth.uid() = sender_id);

-- Reviews policies
CREATE POLICY "Reviews are viewable by everyone" ON reviews
  FOR SELECT USING (true);

CREATE POLICY "Users can create reviews" ON reviews
  FOR INSERT WITH CHECK (auth.uid() = reviewer_id);
```

### 3. Настройка Storage

Создайте два бакета в Storage:

1. **avatars** - для аватарок пользователей
   - Public bucket
   - Allowed MIME types: `image/*`
   - Max file size: 10MB

2. **lot-images** - для изображений лотов
   - Public bucket
   - Allowed MIME types: `image/*`
   - Max file size: 5MB

### 4. Настройка Auth

В Authentication -> Providers включите:

1. **Email** - включить Confirm email (для верификации)
2. **Google** - настроить OAuth:
   - Создайте проект в [Google Cloud Console](https://console.cloud.google.com/)
   - Включите Google+ API
   - Создайте OAuth 2.0 credentials
   - Добавьте Authorized redirect URI: `https://your-project.supabase.co/auth/v1/callback`
   - Скопируйте Client ID и Client Secret в настройки Supabase

### 5. Настройка переменных окружения

Создайте файл `.env` в корне проекта:

```env
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key
```

### 6. Установка зависимостей и запуск

```bash
npm install
npm run dev
```

### 7. Сборка для production

```bash
npm run build
```

## Настройка платежной системы

Для интеграции реальных платежей рекомендуется использовать:

### 1. ЮKassa (ЮMoney)

```javascript
// Пример интеграции
const createPayment = async (amount, description, orderId) => {
  const response = await fetch('https://api.yookassa.ru/v3/payments', {
    method: 'POST',
    headers: {
      'Authorization': 'Basic ' + btoa('shopId:secretKey'),
      'Content-Type': 'application/json',
      'Idempotence-Key': orderId,
    },
    body: JSON.stringify({
      amount: {
        value: amount.toFixed(2),
        currency: 'RUB'
      },
      capture: true,
      confirmation: {
        type: 'redirect',
        return_url: 'https://your-site.com/payment/success'
      },
      description: description,
      metadata: {
        order_id: orderId
      }
    })
  });
  return response.json();
};
```

### 2. CloudPayments

```javascript
const pay = async (amount, description) => {
  const widget = new cp.CloudPayments();
  widget.pay('auth', {
    publicId: 'your-public-id',
    description: description,
    amount: amount,
    currency: 'RUB',
    accountId: userId,
    data: {
      order_id: orderId
    }
  }, {
    onSuccess: function(options) {
      // Обработка успешной оплаты
    },
    onFail: function(reason, options) {
      // Обработка ошибки
    }
  });
};
```

### 3. CryptoCloud (криптовалюты)

```javascript
const createCryptoPayment = async (amount, orderId) => {
  const response = await fetch('https://api.cryptocloud.plus/v2/invoice/create', {
    method: 'POST',
    headers: {
      'Authorization': 'Token your-api-key',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      amount: amount,
      shop_id: 'your-shop-id',
      order_id: orderId,
      currency: 'RUB'
    })
  });
  return response.json();
};
```

## Настройка топа продавцов

Для автоматического расчета топа продавцов создайте материализованное представление:

```sql
CREATE MATERIALIZED VIEW top_sellers AS
SELECT 
  p.id,
  p.username,
  p.avatar_url,
  COUNT(DISTINCT o.id) as sales_count,
  COUNT(DISTINCT CASE WHEN r.rating >= 4 THEN r.id END) as positive_reviews,
  COALESCE(AVG(r.rating), 0) as rating
FROM profiles p
LEFT JOIN orders o ON p.id = o.seller_id AND o.status = 'completed'
LEFT JOIN reviews r ON p.id = r.reviewed_id
GROUP BY p.id, p.username, p.avatar_url
HAVING COUNT(DISTINCT o.id) > 0
ORDER BY sales_count DESC, rating DESC
LIMIT 100;

-- Создайте функцию для обновления
CREATE OR REPLACE FUNCTION refresh_top_sellers()
RETURNS void AS $$
BEGIN
  REFRESH MATERIALIZED VIEW top_sellers;
END;
$$ LANGUAGE plpgsql;

-- Настройте cron для обновления каждый час
SELECT cron.schedule('refresh-top-sellers', '0 * * * *', 'SELECT refresh_top_sellers()');
```

## Webhook для платежей

Создайте Edge Function для обработки webhook:

```typescript
// supabase/functions/payment-webhook/index.ts
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

serve(async (req) => {
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  )

  const { type, object } = await req.json()

  if (type === 'payment.succeeded') {
    const { order_id } = object.metadata
    
    // Update order status
    await supabase
      .from('orders')
      .update({ status: 'paid' })
      .eq('order_number', order_id)
    
    // Create transaction
    await supabase
      .from('transactions')
      .insert({
        user_id: object.metadata.user_id,
        type: 'deposit',
        amount: object.amount.value,
        status: 'completed',
        payment_method: 'card',
        description: `Пополнение через ${type}`
      })
  }

  return new Response(JSON.stringify({ success: true }), {
    headers: { 'Content-Type': 'application/json' }
  })
})
```

## Структура проекта

```
src/
├── components/          # React компоненты
│   ├── Header.tsx
│   ├── CategoryCard.tsx
│   ├── LotCard.tsx
│   └── TopSellers.tsx
├── contexts/            # React контексты
│   ├── AuthContext.tsx
│   └── ThemeContext.tsx
├── lib/                 # Утилиты и API
│   ├── supabase.ts
│   └── database.types.ts
├── pages/               # Страницы приложения
│   ├── Home.tsx
│   ├── Category.tsx
│   ├── Lot.tsx
│   ├── Profile.tsx
│   ├── Chats.tsx
│   ├── Finances.tsx
│   ├── Auth.tsx
│   ├── Sell.tsx
│   ├── Games.tsx
│   └── MyLots.tsx
├── types/               # TypeScript типы
│   └── index.ts
├── App.tsx
├── index.css
└── main.tsx
```

## Поддержка

По всем вопросам обращайтесь в Telegram: [@LovelyConfig](https://t.me/LovelyConfig)

Плагины FPC: [@lovely_market1](https://t.me/lovely_market1)

## Лицензия

MIT License
