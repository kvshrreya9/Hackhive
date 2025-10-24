# Glucose Monitor Setup Instructions

## Database Setup

After connecting to Supabase, you need to run this SQL script in the Supabase SQL Editor to set up your database tables:

```sql
-- Enable UUID extension
create extension if not exists "uuid-ossp";

-- Create profiles table (extends auth.users)
create table public.profiles (
  id uuid references auth.users on delete cascade primary key,
  email text unique not null,
  full_name text,
  created_at timestamp with time zone default timezone('utc'::text, now()) not null,
  updated_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Create glucose_readings table
create table public.glucose_readings (
  id uuid default uuid_generate_v4() primary key,
  user_id uuid references auth.users on delete cascade not null,
  glucose_level integer not null check (glucose_level > 0 and glucose_level < 1000),
  reading_time timestamp with time zone default timezone('utc'::text, now()) not null,
  notes text,
  created_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Enable Row Level Security
alter table public.profiles enable row level security;
alter table public.glucose_readings enable row level security;

-- Policies for profiles table
create policy "Users can view own profile" on public.profiles
  for select using (auth.uid() = id);

create policy "Users can update own profile" on public.profiles
  for update using (auth.uid() = id);

create policy "Users can insert own profile" on public.profiles
  for insert with check (auth.uid() = id);

-- Policies for glucose_readings table  
create policy "Users can view own glucose readings" on public.glucose_readings
  for select using (auth.uid() = user_id);

create policy "Users can insert own glucose readings" on public.glucose_readings
  for insert with check (auth.uid() = user_id);

create policy "Users can update own glucose readings" on public.glucose_readings
  for update using (auth.uid() = user_id);

create policy "Users can delete own glucose readings" on public.glucose_readings
  for delete using (auth.uid() = user_id);

-- Function to automatically create profile on signup
create or replace function public.handle_new_user()
returns trigger as $$
begin
  insert into public.profiles (id, email, full_name)
  values (new.id, new.email, new.raw_user_meta_data->>'full_name');
  return new;
end;
$$ language plpgsql security definer;

-- Trigger to create profile on signup
create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function public.handle_new_user();

-- Indexes for performance
create index glucose_readings_user_id_idx on public.glucose_readings(user_id);
create index glucose_readings_reading_time_idx on public.glucose_readings(reading_time desc);
create index glucose_readings_user_time_idx on public.glucose_readings(user_id, reading_time desc);
```

## Update Supabase Configuration

1. Go to your Supabase project settings
2. Copy your project URL and anon key
3. Update `src/lib/supabase.ts` with your actual Supabase credentials:

```typescript
const supabaseUrl = 'your-actual-project-url'
const supabaseKey = 'your-actual-anon-key'
```

## Deploy Edge Function

The glucose statistics are calculated using a Supabase Edge Function. Deploy it using the Supabase CLI:

```bash
supabase functions deploy glucose-stats
```

## Features Included

✅ **User Authentication** - Sign up/sign in with email and password
✅ **Real-time Glucose Tracking** - Add and view glucose readings 
✅ **Interactive Charts** - Daily and weekly glucose trends
✅ **Smart Alerts** - Warnings for high glucose levels (200+ mg/dL)
✅ **Statistics Dashboard** - Average, time in range, trends
✅ **Secure Data Storage** - Row Level Security policies protect user data
✅ **Responsive Design** - Works on desktop and mobile devices

## Alert Thresholds

- **Normal**: 70-140 mg/dL (Green)
- **Elevated**: 140-200 mg/dL (Yellow) 
- **High**: 200-250 mg/dL (Orange - Mild Warning)
- **Critical**: 250+ mg/dL (Red - Immediate Attention Required)

## Next Steps

1. Run the SQL setup script in Supabase
2. Update your Supabase credentials
3. Deploy the edge function
4. Start logging glucose readings!

Your Raspberry Pi sensor can send data directly to the `/api/readings` endpoint once deployed.