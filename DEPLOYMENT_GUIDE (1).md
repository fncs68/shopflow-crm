# ShopFlow CRM — Complete Deployment Guide

## Tech Stack
- **Frontend**: Next.js 15 App Router + Tailwind CSS
- **Database + Auth + Realtime**: Supabase
- **Payments**: Razorpay (UPI, cards, wallets)
- **WhatsApp**: Meta WhatsApp Business Cloud API
- **Deployment**: Vercel (free tier)

---

## Step 1: Create Next.js App

```bash
npx create-next-app@latest shopflow-crm --typescript --tailwind --app
cd shopflow-crm
npm install @supabase/supabase-js @supabase/auth-helpers-nextjs razorpay
npm install date-fns react-hot-toast lucide-react
```

---

## Step 2: Supabase Database Schema

Go to Supabase Dashboard → SQL Editor and run:

```sql
CREATE TABLE businesses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  owner_id UUID REFERENCES auth.users(id),
  whatsapp_number TEXT,
  whatsapp_phone_number_id TEXT,
  whatsapp_access_token TEXT,
  plan TEXT DEFAULT 'free' CHECK (plan IN ('free','pro','lifetime')),
  conversations_used INT DEFAULT 0,
  razorpay_subscription_id TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE team_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
  user_id UUID REFERENCES auth.users(id),
  name TEXT NOT NULL,
  email TEXT NOT NULL,
  role TEXT DEFAULT 'agent' CHECK (role IN ('admin','agent')),
  is_online BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
  name TEXT,
  phone TEXT NOT NULL,
  whatsapp_id TEXT UNIQUE,
  tag TEXT CHECK (tag IN ('vip','paid','followup','hot','not_interested')),
  notes TEXT,
  last_seen TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
  customer_id UUID REFERENCES customers(id),
  assigned_to UUID REFERENCES team_members(id),
  status TEXT DEFAULT 'open' CHECK (status IN ('open','resolved','waiting')),
  last_message_at TIMESTAMPTZ DEFAULT NOW(),
  unread_count INT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID REFERENCES conversations(id) ON DELETE CASCADE,
  business_id UUID REFERENCES businesses(id),
  direction TEXT NOT NULL CHECK (direction IN ('incoming','outgoing')),
  message_type TEXT DEFAULT 'text' CHECK (message_type IN ('text','image','document','audio','video','template')),
  content TEXT,
  media_url TEXT,
  wa_message_id TEXT,
  sent_by UUID REFERENCES team_members(id),
  status TEXT DEFAULT 'sent' CHECK (status IN ('sent','delivered','read','failed')),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE quick_replies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
  trigger_text TEXT NOT NULL,
  reply_text TEXT NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  description TEXT,
  price DECIMAL(10,2),
  image_url TEXT,
  stock_count INT DEFAULT 0,
  is_available BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
  customer_id UUID REFERENCES customers(id),
  items JSONB DEFAULT '[]',
  total_amount DECIMAL(10,2),
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending','confirmed','paid','delivered','cancelled')),
  payment_id TEXT,
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE broadcasts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  message TEXT NOT NULL,
  segment TEXT DEFAULT 'all',
  status TEXT DEFAULT 'draft' CHECK (status IN ('draft','sending','sent','failed')),
  sent_count INT DEFAULT 0,
  delivered_count INT DEFAULT 0,
  read_count INT DEFAULT 0,
  scheduled_at TIMESTAMPTZ,
  sent_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID REFERENCES businesses(id),
  razorpay_order_id TEXT,
  razorpay_payment_id TEXT,
  amount INT NOT NULL,
  currency TEXT DEFAULT 'INR',
  plan TEXT,
  status TEXT DEFAULT 'created' CHECK (status IN ('created','paid','failed')),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Enable Realtime
ALTER PUBLICATION supabase_realtime ADD TABLE messages;
ALTER PUBLICATION supabase_realtime ADD TABLE conversations;

-- Row Level Security
ALTER TABLE businesses ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE broadcasts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Own business data" ON businesses FOR ALL USING (owner_id = auth.uid());
CREATE POLICY "Own customers" ON customers FOR ALL USING (
  business_id IN (SELECT id FROM businesses WHERE owner_id = auth.uid())
);
CREATE POLICY "Own messages" ON messages FOR ALL USING (
  business_id IN (SELECT id FROM businesses WHERE owner_id = auth.uid())
);
CREATE POLICY "Own conversations" ON conversations FOR ALL USING (
  business_id IN (SELECT id FROM businesses WHERE owner_id = auth.uid())
);

-- Free plan conversation counter
CREATE OR REPLACE FUNCTION increment_conversations(business_id UUID)
RETURNS void AS $$
BEGIN
  UPDATE businesses SET conversations_used = conversations_used + 1
  WHERE id = business_id AND plan = 'free';
  IF (SELECT conversations_used FROM businesses WHERE id = business_id) > 500 THEN
    RAISE EXCEPTION 'Free plan limit reached. Please upgrade to Pro.';
  END IF;
END;
$$ LANGUAGE plpgsql;
```

---

## Step 3: Environment Variables (.env.local)

```env
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

RAZORPAY_KEY_ID=rzp_live_xxxxxxxxxxxx
RAZORPAY_KEY_SECRET=your-razorpay-secret
NEXT_PUBLIC_RAZORPAY_KEY_ID=rzp_live_xxxxxxxxxxxx

WHATSAPP_ACCESS_TOKEN=your-meta-access-token
WHATSAPP_PHONE_NUMBER_ID=your-phone-number-id
WHATSAPP_WEBHOOK_VERIFY_TOKEN=your-random-secret-token
WHATSAPP_BUSINESS_ACCOUNT_ID=your-business-account-id

NEXT_PUBLIC_APP_URL=https://your-app.vercel.app
```

---

## Step 4: WhatsApp Business Cloud API

### 4a. Create Meta App
1. Go to https://developers.facebook.com → Create App → Business type
2. Add "WhatsApp" product
3. Go to WhatsApp → API Setup
4. Copy Phone Number ID and Access Token

### 4b. Webhook Handler (app/api/webhook/whatsapp/route.ts)

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
)

export async function GET(req: NextRequest) {
  const { searchParams } = new URL(req.url)
  const mode = searchParams.get('hub.mode')
  const token = searchParams.get('hub.verify_token')
  const challenge = searchParams.get('hub.challenge')
  if (mode === 'subscribe' && token === process.env.WHATSAPP_WEBHOOK_VERIFY_TOKEN) {
    return new NextResponse(challenge, { status: 200 })
  }
  return new NextResponse('Forbidden', { status: 403 })
}

export async function POST(req: NextRequest) {
  const body = await req.json()
  try {
    const value = body.entry?.[0]?.changes?.[0]?.value
    if (value?.messages) {
      const msg = value.messages[0]
      const contact = value.contacts?.[0]
      const phoneNumber = msg.from
      const messageText = msg.text?.body || ''
      const phoneNumberId = value.metadata?.phone_number_id

      const { data: business } = await supabase
        .from('businesses').select('id')
        .eq('whatsapp_phone_number_id', phoneNumberId).single()
      if (!business) return NextResponse.json({ status: 'ok' })

      const { data: customer } = await supabase.from('customers').upsert({
        business_id: business.id, phone: phoneNumber, whatsapp_id: phoneNumber,
        name: contact?.profile?.name || phoneNumber, last_seen: new Date().toISOString()
      }, { onConflict: 'whatsapp_id' }).select().single()

      let { data: conversation } = await supabase.from('conversations')
        .select('id, unread_count').eq('business_id', business.id)
        .eq('customer_id', customer.id).eq('status', 'open').single()

      if (!conversation) {
        const { data: newConv } = await supabase.from('conversations')
          .insert({ business_id: business.id, customer_id: customer.id })
          .select().single()
        conversation = newConv
        await supabase.rpc('increment_conversations', { business_id: business.id })
      }

      await supabase.from('messages').insert({
        conversation_id: conversation.id, business_id: business.id,
        direction: 'incoming', content: messageText, wa_message_id: msg.id
      })

      await supabase.from('conversations').update({
        last_message_at: new Date().toISOString(),
        unread_count: (conversation.unread_count || 0) + 1
      }).eq('id', conversation.id)

      // Auto-reply check
      const { data: quickReplies } = await supabase.from('quick_replies')
        .select('*').eq('business_id', business.id).eq('is_active', true)
      for (const qr of quickReplies || []) {
        if (messageText.toLowerCase().includes(qr.trigger_text.toLowerCase())) {
          await fetch(`https://graph.facebook.com/v19.0/${phoneNumberId}/messages`, {
            method: 'POST',
            headers: { 'Authorization': `Bearer ${process.env.WHATSAPP_ACCESS_TOKEN}`, 'Content-Type': 'application/json' },
            body: JSON.stringify({ messaging_product: 'whatsapp', to: phoneNumber, type: 'text', text: { body: qr.reply_text } })
          })
          break
        }
      }
    }
  } catch (err) { console.error(err) }
  return NextResponse.json({ status: 'ok' })
}
```

### 4c. Register Webhook
- Webhook URL: https://your-app.vercel.app/api/webhook/whatsapp
- Verify Token: same as WHATSAPP_WEBHOOK_VERIFY_TOKEN
- Subscribe to: messages, message_deliveries, message_reads

---

## Step 5: Razorpay Integration

### Create Order (app/api/payment/create-order/route.ts)
```typescript
import { NextRequest, NextResponse } from 'next/server'
import Razorpay from 'razorpay'

const razorpay = new Razorpay({
  key_id: process.env.RAZORPAY_KEY_ID!,
  key_secret: process.env.RAZORPAY_KEY_SECRET!
})

export async function POST(req: NextRequest) {
  const { plan } = await req.json()
  const amount = plan === 'monthly' ? 79900 : 499900
  const order = await razorpay.orders.create({ amount, currency: 'INR', receipt: `receipt_${Date.now()}`, notes: { plan } })
  return NextResponse.json({ orderId: order.id, amount, currency: 'INR' })
}
```

### Verify Payment (app/api/payment/verify/route.ts)
```typescript
import { NextRequest, NextResponse } from 'next/server'
import crypto from 'crypto'
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(process.env.NEXT_PUBLIC_SUPABASE_URL!, process.env.SUPABASE_SERVICE_ROLE_KEY!)

export async function POST(req: NextRequest) {
  const { razorpay_order_id, razorpay_payment_id, razorpay_signature, business_id, plan } = await req.json()
  const sign = razorpay_order_id + '|' + razorpay_payment_id
  const expectedSig = crypto.createHmac('sha256', process.env.RAZORPAY_KEY_SECRET!).update(sign).digest('hex')
  if (expectedSig !== razorpay_signature) return NextResponse.json({ error: 'Invalid' }, { status: 400 })
  await supabase.from('businesses').update({ plan: plan === 'monthly' ? 'pro' : 'lifetime' }).eq('id', business_id)
  await supabase.from('payments').insert({ business_id, razorpay_order_id, razorpay_payment_id, amount: plan === 'monthly' ? 79900 : 499900, plan, status: 'paid' })
  return NextResponse.json({ success: true })
}
```

### Frontend (in your component)
```typescript
async function handleUpgrade(plan: 'monthly' | 'lifetime') {
  const res = await fetch('/api/payment/create-order', { method: 'POST', body: JSON.stringify({ plan }) })
  const { orderId, amount } = await res.json()
  const options = {
    key: process.env.NEXT_PUBLIC_RAZORPAY_KEY_ID,
    amount, currency: 'INR',
    name: 'ShopFlow CRM',
    description: plan === 'monthly' ? 'Pro Plan - Rs.799/month' : 'Lifetime Deal - Rs.4,999',
    order_id: orderId,
    theme: { color: '#25D366' },
    handler: async (response: any) => {
      await fetch('/api/payment/verify', { method: 'POST', body: JSON.stringify({ ...response, business_id, plan }) })
    }
  }
  const rzp = new (window as any).Razorpay(options)
  rzp.open()
}
```

---

## Step 6: Deploy to Vercel

```bash
npm install -g vercel
vercel --prod
```

Then add all environment variables in Vercel Dashboard → Settings → Environment Variables.

---

## Step 7: Realtime Inbox

```typescript
useEffect(() => {
  const channel = supabase.channel('messages')
    .on('postgres_changes', {
      event: 'INSERT', schema: 'public', table: 'messages',
      filter: `business_id=eq.${businessId}`
    }, (payload) => {
      setMessages(prev => [...prev, payload.new])
    }).subscribe()
  return () => supabase.removeChannel(channel)
}, [businessId])
```

---

## Pricing Summary

| Plan | Price | Features |
|------|-------|---------|
| Free | Rs.0/month | 500 conversations, 1 number, 1 user |
| Pro | Rs.799/month | Unlimited conversations, 5 numbers, 10 team members, broadcast, analytics |
| Lifetime | Rs.4,999 one-time | Everything Pro, forever, no monthly fees |

---

## Go-To-Market India Strategy
1. Share in local business WhatsApp groups
2. Google My Business targeting kirana/salon/clinic owners
3. YouTube Shorts demo in Hindi
4. Partner with CA/GST consultants who serve small businesses
5. Offer free setup service for first 100 customers
6. List on IndiaMART, Justdial as software service
