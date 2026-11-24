# Stripon-app
My first application
-- STRIPON DATABASE FOUNDATION

create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  username text unique not null,
  bio text default '',
  avatar_url text,
  created_at timestamptz default now()
);

create table public.follows (
  follower_id uuid references auth.users(id) on delete cascade,
  following_id uuid references auth.users(id) on delete cascade,
  created_at timestamptz default now(),
  primary key (follower_id, following_id),
  check (follower_id <> following_id)
);

create table public.videos (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  title text not null,
  description text default '',
  video_url text,
  category text default 'all',
  published boolean default false,
  created_at timestamptz default now()
);

-- Enable security
alter table public.profiles enable row level security;
alter table public.follows enable row level security;
alter table public.videos enable row level security;

-- Profile permissions
grant select on public.profiles to anon, authenticated;
grant insert, update on public.profiles to authenticated;

create policy "Public can view profiles"
on public.profiles for select
to anon, authenticated
using (true);

create policy "Users create own profile"
on public.profiles for insert
to authenticated
with check ((select auth.uid()) = id);

create policy "Users update own profile"
on public.profiles for update
to authenticated
using ((select auth.uid()) = id)
with check ((select auth.uid()) = id);

-- Follow permissions
grant select on public.follows to anon, authenticated;
grant insert, delete on public.follows to authenticated;

create policy "Public can view follows"
on public.follows for select
to anon, authenticated
using (true);

create policy "Users follow as themselves"
on public.follows for insert
to authenticated
with check ((select auth.uid()) = follower_id);

create policy "Users unfollow themselves"
on public.follows for delete
to authenticated
using ((select auth.uid()) = follower_id);

-- Video permissions
grant select on public.videos to anon, authenticated;
grant insert, update, delete on public.videos to authenticated;

create policy "Public can view published videos"
on public.videos for select
to anon, authenticated
using (published = true or (select auth.uid()) = user_id);

create policy "Users upload their own video records"
on public.videos for insert
to authenticated
with check ((select auth.uid()) = user_id);

create policy "Users update their own videos"
on public.videos for update
to authenticated
using ((select auth.uid()) = user_id)
with check ((select auth.uid()) = user_id);

create policy "Users delete their own videos"
on public.videos for delete
to authenticated
using ((select auth.uid()) = user_id);
