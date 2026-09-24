# Pauta-- =============================================================================
-- Pauta · 0001 · Esquema principal
-- Execute no SQL Editor do Supabase (ou via `supabase db push`), nesta ordem:
--   0001_schema.sql -> 0002_rls.sql -> 0003_views.sql -> 0004_functions.sql
--
-- Convenções:
--   * Toda tabela pessoal tem user_id (default auth.uid()) e RLS ligada (ver 0002).
--   * Chaves estrangeiras compostas (id, user_id) impedem que um usuário
--     aponte para uma linha que pertence a outro usuário.
--   * Estatísticas (horas, questões, acertos) NÃO são gravadas nas tabelas:
--     são calculadas nas views (0003) a partir das sessões e questões.
-- =============================================================================

-- ---------- Tipos -------------------------------------------------------------
create type public.study_category as enum
  ('teoria','revisao','questoes','videoaula','leitura','leitura_lei','resumo','simulado','atividade','outro');
create type public.block_kind as enum
  ('estudo','revisao','questoes','simulado','videoaula','leitura','resumo','atividade');
create type public.block_status as enum
  ('planejado','em_andamento','concluido','cancelado','atrasado');
create type public.topic_status as enum
  ('nao_iniciado','em_andamento','concluido');
create type public.review_status as enum
  ('programada','concluida','ignorada');
create type public.goal_kind as enum
  ('horas','questoes','revisoes','topicos','disciplinas');
create type public.goal_period as enum
  ('diaria','semanal','mensal');
create type public.event_kind as enum
  ('estudo','revisao','aula','atividade','simulado','meta','outro');
create type public.notification_category as enum
  ('inicio_estudo','revisao','revisao_atrasada','meta','planejamento','sessao_pendente');

-- ---------- Utilidades --------------------------------------------------------
create or replace function public.set_updated_at()
returns trigger language plpgsql as $$
begin
  new.updated_at = now();
  return new;
end $$;

-- ---------- Perfil e configurações -------------------------------------------
create table public.profiles (
  id uuid primary key references auth.users (id) on delete cascade,
  full_name text not null default '',
  avatar_url text,
  objective text,
  institution text,
  course text,
  semester smallint check (semester between 1 and 30),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table public.user_settings (
  user_id uuid primary key references auth.users (id) on delete cascade,
  focus_mode boolean not null default false,
  pomodoro_focus_minutes smallint not null default 50 check (pomodoro_focus_minutes between 1 and 240),
  pomodoro_break_minutes smallint not null default 10 check (pomodoro_break_minutes between 1 and 60),
  default_review_intervals smallint[] not null default '{1,7,30,60,90}',
  count_classes_as_study boolean not null default false,
  week_starts_on smallint not null default 0 check (week_starts_on in (0, 1)),
  notification_prefs jsonb not null default
    '{"inicio_estudo":true,"revisao":true,"revisao_atrasada":true,"meta":true,"planejamento":true,"sessao_pendente":true}',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

-- ---------- Disciplinas, tópicos e subtópicos --------------------------------
create table public.subjects (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  name text not null check (char_length(btrim(name)) between 1 and 120),
  description text check (char_length(description) <= 2000),
  teacher text check (char_length(teacher) <= 120),
  workload_hours numeric(6,1) check (workload_hours is null or workload_hours >= 0),
  color text not null default '#0B5563' check (color ~ '^#[0-9A-Fa-f]{6}$'),
  priority smallint not null default 2 check (priority between 1 and 3),
  position integer not null default 0,
  archived_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (id, user_id)
);
create index subjects_user_position_idx on public.subjects (user_id, position);

create table public.topics (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  subject_id uuid not null,
  name text not null check (char_length(btrim(name)) between 1 and 300),
  description text check (char_length(description) <= 2000),
  status public.topic_status not null default 'nao_iniciado',
  difficulty smallint not null default 3 check (difficulty between 1 and 5),
  mastery smallint not null default 0 check (mastery between 0 and 100),
  notes text check (char_length(notes) <= 4000),
  position integer not null default 0,
  completed_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (id, user_id),
  foreign key (subject_id, user_id) references public.subjects (id, user_id) on delete cascade
);
create index topics_subject_position_idx on public.topics (user_id, subject_id, position);

create table public.subtopics (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  topic_id uuid not null,
  name text not null check (char_length(btrim(name)) between 1 and 300),
  done boolean not null default false,
  position integer not null default 0,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (id, user_id),
  foreign key (topic_id, user_id) references public.topics (id, user_id) on delete cascade
);
create index subtopics_topic_position_idx on public.subtopics (user_id, topic_id, position);

-- ---------- Planejamento ------------------------------------------------------
create table public.study_plans (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  name text not null check (char_length(btrim(name)) between 1 and 120),
  objective text check (char_length(objective) <= 500),
  is_active boolean not null default false,
  weekly_hours numeric(5,1) check (weekly_hours is null or (weekly_hours > 0 and weekly_hours <= 168)),
  study_days smallint[] not null default '{1,2,3,4,5}'
    check (study_days <@ array[0,1,2,3,4,5,6]::smallint[]),
  min_session_minutes smallint not null default 30 check (min_session_minutes between 5 and 600),
  max_session_minutes smallint not null default 90 check (max_session_minutes between 5 and 600),
  cycles_completed integer not null default 0 check (cycles_completed >= 0),
  current_cycle integer not null default 1 check (current_cycle >= 1),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  check (min_session_minutes <= max_session_minutes),
  unique (id, user_id)
);
create unique index study_plans_one_active_per_user on public.study_plans (user_id) where is_active;

-- Peso de cada disciplina no planejamento (importância x conhecimento).
create table public.study_plan_subjects (
  plan_id uuid not null,
  subject_id uuid not null,
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  importance numeric(2,1) not null default 3 check (importance between 1 and 5),
  knowledge numeric(2,1) not null default 3 check (knowledge between 1 and 5),
  share_percent numeric(5,2) check (share_percent is null or share_percent between 0 and 100),
  primary key (plan_id, subject_id),
  foreign key (plan_id, user_id) references public.study_plans (id, user_id) on delete cascade,
  foreign key (subject_id, user_id) references public.subjects (id, user_id) on delete cascade
);

create table public.study_blocks (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  plan_id uuid,
  subject_id uuid not null,
  topic_id uuid,
  kind public.block_kind not null default 'estudo',
  status public.block_status not null default 'planejado',
  cycle_number integer not null default 1 check (cycle_number >= 1),
  position integer not null default 0,
  planned_minutes smallint not null check (planned_minutes between 5 and 600),
  scheduled_date date,
  start_time time,
  notes text check (char_length(notes) <= 2000),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (id, user_id),
  foreign key (plan_id, user_id) references public.study_plans (id, user_id) on delete cascade,
  foreign key (subject_id, user_id) references public.subjects (id, user_id) on delete cascade,
  foreign key (topic_id, user_id) references public.topics (id, user_id) on delete set null (topic_id)
);
create index study_blocks_plan_cycle_idx on public.study_blocks (user_id, plan_id, cycle_number, position);
create index study_blocks_date_idx on public.study_blocks (user_id, scheduled_date);

-- ---------- Registro de estudo ------------------------------------------------
create table public.study_sessions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  subject_id uuid not null,
  topic_id uuid,
  block_id uuid,
  category public.study_category not null default 'teoria',
  studied_on date not null default current_date,
  started_at timestamptz,
  duration_seconds integer not null default 0 check (duration_seconds between 0 and 86400),
  material text check (char_length(material) <= 200),
  theory_completed boolean not null default false,
  pages_read integer not null default 0 check (pages_read >= 0),
  page_ranges jsonb not null default '[]'::jsonb,
  comment text check (char_length(comment) <= 4000),
  source text not null default 'manual' check (source in ('manual', 'timer')),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (id, user_id),
  foreign key (subject_id, user_id) references public.subjects (id, user_id) on delete cascade,
  foreign key (topic_id, user_id) references public.topics (id, user_id) on delete set null (topic_id),
  foreign key (block_id, user_id) references public.study_blocks (id, user_id) on delete set null (block_id)
);
create index study_sessions_date_idx on public.study_sessions (user_id, studied_on desc);
create index study_sessions_subject_idx on public.study_sessions (user_id, subject_id);
create index study_sessions_topic_idx on public.study_sessions (user_id, topic_id);

create table public.questions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  session_id uuid,
  subject_id uuid not null,
  topic_id uuid,
  answered_on date not null default current_date,
  correct integer not null default 0 check (correct >= 0),
  wrong integer not null default 0 check (wrong >= 0),
  total integer generated always as (correct + wrong) stored,
  created_at timestamptz not null default now(),
  check (correct + wrong > 0),
  unique (id, user_id),
  foreign key (session_id, user_id) references public.study_sessions (id, user_id) on delete cascade,
  foreign key (subject_id, user_id) references public.subjects (id, user_id) on delete cascade,
  foreign key (topic_id, user_id) references public.topics (id, user_id) on delete set null (topic_id)
);
create index questions_date_idx on public.questions (user_id, answered_on desc);
create index questions_subject_idx on public.questions (user_id, subject_id);
create index questions_topic_idx on public.questions (user_id, topic_id);

create table public.videos (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  session_id uuid not null,
  topic_id uuid,
  title text not null check (char_length(btrim(title)) between 1 and 200),
  start_seconds integer not null default 0 check (start_seconds >= 0),
  end_seconds integer not null default 0 check (end_seconds >= 0),
  created_at timestamptz not null default now(),
  check (end_seconds >= start_seconds),
  unique (id, user_id),
  foreign key (session_id, user_id) references public.study_sessions (id, user_id) on delete cascade,
  foreign key (topic_id, user_id) references public.topics (id, user_id) on delete set null (topic_id)
);
create index videos_session_idx on public.videos (user_id, session_id);

-- ---------- Revisões ----------------------------------------------------------
create table public.reviews (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  subject_id uuid not null,
  topic_id uuid,
  origin_session_id uuid,
  interval_days smallint not null check (interval_days between 1 and 3650),
  due_on date not null,
  status public.review_status not null default 'programada',
  completed_at timestamptz,
  completed_session_id uuid,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (id, user_id),
  foreign key (subject_id, user_id) references public.subjects (id, user_id) on delete cascade,
  foreign key (topic_id, user_id) references public.topics (id, user_id) on delete cascade,
  foreign key (origin_session_id, user_id) references public.study_sessions (id, user_id) on delete set null (origin_session_id),
  foreign key (completed_session_id, user_id) references public.study_sessions (id, user_id) on delete set null (completed_session_id)
);
create index reviews_due_idx on public.reviews (user_id, status, due_on);

-- ---------- Metas, calendário, aulas, notas, notificações --------------------
create table public.goals (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  kind public.goal_kind not null,
  period public.goal_period not null,
  target numeric(10,2) not null check (target > 0),
  subject_id uuid,
  active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (id, user_id),
  foreign key (subject_id, user_id) references public.subjects (id, user_id) on delete cascade
);
create index goals_user_idx on public.goals (user_id, active);

create table public.calendar_events (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  title text not null check (char_length(btrim(title)) between 1 and 200),
  kind public.event_kind not null default 'estudo',
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  all_day boolean not null default false,
  subject_id uuid,
  topic_id uuid,
  block_id uuid,
  review_id uuid,
  description text check (char_length(description) <= 2000),
  completed_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  check (ends_at >= starts_at),
  unique (id, user_id),
  foreign key (subject_id, user_id) references public.subjects (id, user_id) on delete set null (subject_id),
  foreign key (topic_id, user_id) references public.topics (id, user_id) on delete set null (topic_id),
  foreign key (block_id, user_id) references public.study_blocks (id, user_id) on delete set null (block_id),
  foreign key (review_id, user_id) references public.reviews (id, user_id) on delete set null (review_id)
);
create index calendar_events_range_idx on public.calendar_events (user_id, starts_at);

create table public.classes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  subject_id uuid not null,
  teacher text check (char_length(teacher) <= 120),
  weekday smallint not null check (weekday between 0 and 6),
  start_time time not null,
  end_time time not null,
  room text check (char_length(room) <= 60),
  absences smallint not null default 0 check (absences >= 0),
  max_absences smallint check (max_absences is null or max_absences >= 0),
  counts_as_study boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  check (end_time > start_time),
  unique (id, user_id),
  foreign key (subject_id, user_id) references public.subjects (id, user_id) on delete cascade
);

create table public.notes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  subject_id uuid,
  topic_id uuid,
  session_id uuid,
  title text check (char_length(title) <= 200),
  body text not null check (char_length(body) <= 20000),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (id, user_id),
  foreign key (subject_id, user_id) references public.subjects (id, user_id) on delete cascade,
  foreign key (topic_id, user_id) references public.topics (id, user_id) on delete cascade,
  foreign key (session_id, user_id) references public.study_sessions (id, user_id) on delete cascade
);

create table public.notifications (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  category public.notification_category not null,
  title text not null check (char_length(title) <= 200),
  body text check (char_length(body) <= 1000),
  scheduled_for timestamptz not null default now(),
  read_at timestamptz,
  created_at timestamptz not null default now()
);
create index notifications_user_idx on public.notifications (user_id, scheduled_for desc);

-- ---------- Gatilhos updated_at ----------------------------------------------
do $$
declare t text;
begin
  foreach t in array array[
    'profiles','user_settings','subjects','topics','subtopics','study_plans',
    'study_blocks','study_sessions','reviews','goals','calendar_events','classes','notes'
  ] loop
    execute format(
      'create trigger %I before update on public.%I for each row execute function public.set_updated_at()',
      t || '_set_updated_at', t);
  end loop;
end $$;

-- ---------- Novo usuário: cria perfil e configurações ------------------------
create or replace function public.handle_new_user()
returns trigger
language plpgsql
security definer
set search_path = public
as $$
begin
  insert into public.profiles (id, full_name)
  values (new.id, coalesce(new.raw_user_meta_data ->> 'full_name', ''));
  insert into public.user_settings (user_id) values (new.id);
  return new;
end $$;

revoke execute on function public.handle_new_user() from public, anon, authenticated;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function public.handle_new_user();
-- =============================================================================
-- Pauta · 0002 · Row Level Security
-- Regra única: cada usuário só enxerga e altera linhas cujo user_id = auth.uid().
-- (select auth.uid()) é avaliado uma vez por consulta, o que é mais rápido.
-- =============================================================================

-- Tabelas com user_id
do $$
declare t text;
begin
  foreach t in array array[
    'subjects','topics','subtopics','study_plans','study_plan_subjects','study_blocks',
    'study_sessions','questions','videos','reviews','goals','calendar_events',
    'classes','notes','notifications','user_settings'
  ] loop
    execute format('alter table public.%I enable row level security', t);
    execute format(
      'create policy %I on public.%I for all to authenticated
         using (user_id = (select auth.uid()))
         with check (user_id = (select auth.uid()))',
      t || '_own_rows', t);
  end loop;
end $$;

-- Perfil: a chave é o próprio id do usuário; não há exclusão direta
-- (o perfil some junto com a conta, por cascata).
alter table public.profiles enable row level security;

create policy profiles_select_own on public.profiles
  for select to authenticated using (id = (select auth.uid()));
create policy profiles_insert_own on public.profiles
  for insert to authenticated with check (id = (select auth.uid()));
create policy profiles_update_own on public.profiles
  for update to authenticated
  using (id = (select auth.uid())) with check (id = (select auth.uid()));
-- =============================================================================
-- Pauta · 0003 · Views de estatística
-- Tudo é calculado a partir de study_sessions, questions, videos e reviews.
-- security_invoker = true: a view executa com as permissões de quem consulta,
-- então a RLS das tabelas de origem continua valendo (exige PostgreSQL 15+).
-- =============================================================================

create or replace view public.topic_stats with (security_invoker = true) as
select
  t.id                                   as topic_id,
  t.user_id,
  t.subject_id,
  coalesce(s.minutes, 0)                 as minutes_studied,
  coalesce(q.total, 0)::bigint           as questions_total,
  coalesce(q.correct, 0)::bigint         as correct,
  coalesce(q.wrong, 0)::bigint           as wrong,
  case when coalesce(q.total, 0) > 0
       then round(100.0 * q.correct / q.total, 1) end as accuracy,
  coalesce(s.pages, 0)::bigint           as pages_read,
  coalesce(v.videos, 0)::bigint          as videos_count,
  s.last_studied_on,
  r.next_review_on
from public.topics t
left join (
  select topic_id,
         sum(duration_seconds)::numeric / 60 as minutes,
         sum(pages_read)                     as pages,
         max(studied_on)                     as last_studied_on
  from public.study_sessions
  where topic_id is not null
  group by topic_id
) s on s.topic_id = t.id
left join (
  select topic_id, sum(total) as total, sum(correct) as correct, sum(wrong) as wrong
  from public.questions
  where topic_id is not null
  group by topic_id
) q on q.topic_id = t.id
left join (
  select topic_id, count(*) as videos
  from public.videos
  where topic_id is not null
  group by topic_id
) v on v.topic_id = t.id
left join (
  select topic_id, min(due_on) as next_review_on
  from public.reviews
  where status = 'programada' and topic_id is not null
  group by topic_id
) r on r.topic_id = t.id;

create or replace view public.subject_stats with (security_invoker = true) as
select
  sb.id                                  as subject_id,
  sb.user_id,
  coalesce(s.minutes, 0)                 as minutes_studied,
  coalesce(q.total, 0)::bigint           as questions_total,
  coalesce(q.correct, 0)::bigint         as correct,
  coalesce(q.wrong, 0)::bigint           as wrong,
  case when coalesce(q.total, 0) > 0
       then round(100.0 * q.correct / q.total, 1) end as accuracy,
  coalesce(tp.total, 0)::bigint          as topics_total,
  coalesce(tp.done, 0)::bigint           as topics_done,
  s.last_studied_on
from public.subjects sb
left join (
  select subject_id,
         sum(duration_seconds)::numeric / 60 as minutes,
         max(studied_on)                     as last_studied_on
  from public.study_sessions
  group by subject_id
) s on s.subject_id = sb.id
left join (
  select subject_id, sum(total) as total, sum(correct) as correct, sum(wrong) as wrong
  from public.questions
  group by subject_id
) q on q.subject_id = sb.id
left join (
  select subject_id,
         count(*)                                    as total,
         count(*) filter (where status = 'concluido') as done
  from public.topics
  group by subject_id
) tp on tp.subject_id = sb.id;
-- =============================================================================
-- Pauta · 0004 · Funções chamáveis pelo app (RPC)
--   seed_demo_data()       -> "Usar dados de demonstração"
--   clear_my_study_data()  -> apaga os dados de estudo do próprio usuário
-- Ambas são SECURITY DEFINER, mas só operam sobre auth.uid(): nunca sobre
-- dados de outra pessoa.
-- =============================================================================

create or replace function public.seed_demo_data()
returns void
language plpgsql
security definer
set search_path = public
as $$
declare
  uid uuid := auth.uid();
  demo jsonb := $json$[
    {"name":"Direito Processual Civil II","teacher":"Profa. Helena Duarte","color":"#0B5563","hours":80,"priority":3,
     "topics":["Processo de conhecimento","Competência","Petição inicial","Resposta do réu","Provas","Sentença","Recursos"]},
    {"name":"Direito Processual Penal I","teacher":"Prof. Marcos Vieira","color":"#C2503B","hours":60,"priority":3,
     "topics":["Inquérito policial","Ação penal","Competência no processo penal","Prisões cautelares","Provas no processo penal","Procedimentos"]},
    {"name":"Direito Constitucional","teacher":"Profa. Lúcia Andrade","color":"#3E7CB1","hours":80,"priority":2,
     "topics":["Princípios fundamentais","Direitos e garantias fundamentais","Organização do Estado","Poderes da República","Controle de constitucionalidade"]},
    {"name":"Direito Civil","teacher":"Prof. Renato Sampaio","color":"#D9902F","hours":80,"priority":2,
     "topics":["Pessoa natural","Pessoa jurídica","Bens","Fato jurídico","Obrigações","Contratos"]}
  ]$json$;
  cats public.study_category[] :=
    array['teoria','questoes','revisao','videoaula','leitura']::public.study_category[];
  subj jsonb;
  tname text;
  s_idx int := 0;
  t_idx int;
  sid uuid;
  tid uuid;
  sess uuid;
  s_ids uuid[] := '{}';
  cat public.study_category;
  dur int;
  ok int;
  ko int;
  pg int;
  pg_start int;
  iv int;
  due date;
  is_done boolean;
begin
  if uid is null then
    raise exception 'Usuário não autenticado' using errcode = '28000';
  end if;
  if exists (select 1 from public.subjects where user_id = uid) then
    raise exception 'Você já tem disciplinas cadastradas. Limpe seus dados antes de carregar a demonstração.'
      using errcode = 'P0001';
  end if;

  -- Disciplinas e tópicos
  for subj in select jsonb_array_elements(demo) loop
    insert into public.subjects (user_id, name, description, teacher, workload_hours, color, priority, position)
    values (uid, subj ->> 'name', 'Disciplina de demonstração.', subj ->> 'teacher',
            (subj ->> 'hours')::numeric, subj ->> 'color', (subj ->> 'priority')::smallint, s_idx)
    returning id into sid;
    s_ids := s_ids || sid;

    t_idx := 0;
    for tname in select jsonb_array_elements_text(subj -> 'topics') loop
      insert into public.topics (user_id, subject_id, name, difficulty, position)
      values (uid, sid, tname, 2 + (t_idx % 3), t_idx);
      t_idx := t_idx + 1;
    end loop;
    s_idx := s_idx + 1;
  end loop;

  -- Duas semanas de sessões (com lacunas, como na vida real)
  for d in 0..13 loop
    for k in 0..1 loop
      continue when (d + k * 3) % 4 = 3;

      sid := s_ids[1 + ((d * 2 + k) % array_length(s_ids, 1))];
      select id into tid
        from public.topics
        where subject_id = sid and user_id = uid
        order by position
        offset ((d + k) % 5) limit 1;

      cat := cats[1 + ((d + k * 2) % 5)];
      dur := (30 + ((d * 11 + k * 17) % 5) * 15) * 60;
      pg := case when cat = 'leitura' then 12 + ((d * 3) % 10) else 0 end;
      pg_start := 10 + d * 4;

      insert into public.study_sessions
        (user_id, subject_id, topic_id, category, studied_on, started_at, duration_seconds,
         material, theory_completed, pages_read, page_ranges, source)
      values
        (uid, sid, tid, cat, current_date - d,
         (current_date - d)::timestamp + time '08:30' + (k * interval '6 hours'),
         dur,
         case cat when 'videoaula' then 'Aula ' || lpad((d + 1)::text, 2, '0')
                  when 'leitura'   then 'Manual, cap. ' || (d % 9 + 1)
                  else null end,
         cat = 'teoria',
         pg,
         case when pg > 0
              then jsonb_build_array(jsonb_build_object('start', pg_start, 'end', pg_start + pg))
              else '[]'::jsonb end,
         'manual')
      returning id into sess;

      if cat in ('questoes', 'revisao') or (cat = 'teoria' and (d + k) % 2 = 0) then
        ok := 6 + ((d * 7 + k * 3) % 10);
        ko := 1 + ((d * 5 + k * 2) % 6);
        insert into public.questions (user_id, session_id, subject_id, topic_id, answered_on, correct, wrong)
        values (uid, sess, sid, tid, current_date - d, ok, ko);
      end if;

      if cat = 'videoaula' then
        insert into public.videos (user_id, session_id, topic_id, title, start_seconds, end_seconds)
        values (uid, sess, tid, 'Aula ' || lpad((d + 1)::text, 2, '0'), 0, dur);
      end if;

      if cat = 'teoria' then
        foreach iv in array array[1, 7, 30] loop
          due := current_date - d + iv;
          is_done := due < current_date - 1 and d % 2 = 0;
          insert into public.reviews
            (user_id, subject_id, topic_id, origin_session_id, interval_days, due_on, status, completed_at)
          values
            (uid, sid, tid, sess, iv, due,
             case when is_done then 'concluida' else 'programada' end::public.review_status,
             case when is_done then now() - interval '1 day' end);
        end loop;
      end if;
    end loop;
  end loop;

  -- Situação dos tópicos conforme o que foi estudado
  update public.topics set status = 'em_andamento'
    where user_id = uid
      and id in (select topic_id from public.study_sessions where user_id = uid and topic_id is not null);
  update public.topics set status = 'concluido', completed_at = now()
    where user_id = uid and position % 2 = 0
      and id in (select topic_id from public.study_sessions
                 where user_id = uid and theory_completed and topic_id is not null);

  insert into public.goals (user_id, kind, period, target)
  values (uid, 'horas', 'semanal', 15), (uid, 'questoes', 'semanal', 100);
end $$;

create or replace function public.clear_my_study_data()
returns void
language plpgsql
security definer
set search_path = public
as $$
declare
  uid uuid := auth.uid();
begin
  if uid is null then
    raise exception 'Usuário não autenticado' using errcode = '28000';
  end if;
  delete from public.notifications    where user_id = uid;
  delete from public.calendar_events  where user_id = uid;
  delete from public.goals            where user_id = uid;
  delete from public.study_plans      where user_id = uid;
  delete from public.notes            where user_id = uid;
  delete from public.subjects         where user_id = uid; -- cascata: tópicos, sessões, questões, revisões...
end $$;

revoke execute on function public.seed_demo_data()      from public, anon;
revoke execute on function public.clear_my_study_data() from public, anon;
grant  execute on function public.seed_demo_data()      to authenticated;
grant  execute on function public.clear_my_study_data() to authenticated;

-- Visitantes sem login nunca tocam nas tabelas.
revoke all on all tables in schema public from anon;
-- =============================================================================
-- Pauta · 0005 · Registro de estudo, planejamento em ciclo e agregações diárias
-- Funções SECURITY INVOKER: rodam com as permissões (e a RLS) de quem chama.
-- Cada função é uma transação única: ou grava tudo, ou nada.
-- =============================================================================

-- ---------- Rotina semanal (aulas, estudo, revisão, tempo livre...) ----------
alter table public.classes alter column subject_id drop not null;
alter table public.classes add column kind text not null default 'aula'
  check (kind in ('aula', 'estudo', 'revisao', 'questoes', 'atividade', 'livre'));
alter table public.classes add column title text check (char_length(title) <= 120);

-- ---------- Agregações diárias (base do dashboard, histórico e estatísticas) --
create or replace view public.session_daily with (security_invoker = true) as
select
  user_id,
  studied_on                              as day,
  subject_id,
  category,
  sum(duration_seconds)::numeric / 60     as minutes,
  count(*)::bigint                        as sessions,
  coalesce(sum(pages_read), 0)::bigint    as pages
from public.study_sessions
group by user_id, studied_on, subject_id, category;

create or replace view public.question_daily with (security_invoker = true) as
select
  user_id,
  answered_on                  as day,
  subject_id,
  topic_id,
  sum(total)::bigint           as total,
  sum(correct)::bigint         as correct,
  sum(wrong)::bigint           as wrong
from public.questions
group by user_id, answered_on, subject_id, topic_id;

-- ---------- Situação de um bloco do ciclo -------------------------------------
create or replace function public.refresh_block_status(b uuid)
returns void
language plpgsql
set search_path = public
as $$
declare
  secs bigint;
  planned int;
  cur public.block_status;
begin
  if b is null then return; end if;
  select planned_minutes, status into planned, cur from public.study_blocks where id = b;
  if not found or cur = 'cancelado' then return; end if;
  select coalesce(sum(duration_seconds), 0) into secs from public.study_sessions where block_id = b;
  update public.study_blocks
     set status = (case when secs >= planned * 60 then 'concluido'
                        when secs > 0 then 'em_andamento'
                        else 'planejado' end)::public.block_status
   where id = b;
end $$;

-- ---------- Registrar sessão de estudo ----------------------------------------
-- Payload (jsonb): subject_id, topic_id?, block_id?, review_id?, category, studied_on,
--   duration_seconds, material?, theory_completed?, comment?, source?, correct?, wrong?,
--   page_ranges? [{start,end}], videos? [{title,start_seconds,end_seconds}],
--   review_intervals? [dias]
create or replace function public.register_study_session(p jsonb)
returns uuid
language plpgsql
set search_path = public
as $$
declare
  sid uuid;
  v_subject uuid := (p ->> 'subject_id')::uuid;
  v_topic uuid := nullif(p ->> 'topic_id', '')::uuid;
  v_block uuid := nullif(p ->> 'block_id', '')::uuid;
  v_review uuid := nullif(p ->> 'review_id', '')::uuid;
  v_date date := coalesce(nullif(p ->> 'studied_on', '')::date, current_date);
  v_cat public.study_category := coalesce(nullif(p ->> 'category', ''), 'teoria')::public.study_category;
  v_secs int := coalesce((p ->> 'duration_seconds')::int, 0);
  v_theory boolean := coalesce((p ->> 'theory_completed')::boolean, false);
  v_correct int := coalesce((p ->> 'correct')::int, 0);
  v_wrong int := coalesce((p ->> 'wrong')::int, 0);
  v_ranges jsonb := coalesce(p -> 'page_ranges', '[]'::jsonb);
  v_pages int;
  v_video jsonb;
  iv int;
begin
  if v_subject is null then
    raise exception 'Escolha a disciplina.' using errcode = 'P0001';
  end if;

  select coalesce(sum(greatest((r ->> 'end')::int - (r ->> 'start')::int + 1, 0)), 0)
    into v_pages from jsonb_array_elements(v_ranges) r;

  insert into public.study_sessions
    (subject_id, topic_id, block_id, category, studied_on, started_at, duration_seconds,
     material, theory_completed, pages_read, page_ranges, comment, source)
  values
    (v_subject, v_topic, v_block, v_cat, v_date,
     nullif(p ->> 'started_at', '')::timestamptz, v_secs,
     nullif(p ->> 'material', ''), v_theory, v_pages, v_ranges,
     nullif(p ->> 'comment', ''), coalesce(nullif(p ->> 'source', ''), 'manual'))
  returning id into sid;

  if v_correct + v_wrong > 0 then
    insert into public.questions (session_id, subject_id, topic_id, answered_on, correct, wrong)
    values (sid, v_subject, v_topic, v_date, v_correct, v_wrong);
  end if;

  for v_video in select * from jsonb_array_elements(coalesce(p -> 'videos', '[]'::jsonb)) loop
    insert into public.videos (session_id, topic_id, title, start_seconds, end_seconds)
    values (sid, v_topic, v_video ->> 'title',
            coalesce((v_video ->> 'start_seconds')::int, 0),
            coalesce((v_video ->> 'end_seconds')::int, 0));
  end loop;

  for iv in
    select distinct value::int
      from jsonb_array_elements_text(coalesce(p -> 'review_intervals', '[]'::jsonb)) as t(value)
     order by 1
  loop
    continue when iv < 1 or iv > 3650;
    insert into public.reviews (subject_id, topic_id, origin_session_id, interval_days, due_on)
    select v_subject, v_topic, sid, iv, v_date + iv
     where not exists (
       select 1 from public.reviews r
        where r.status = 'programada' and r.subject_id = v_subject
          and r.topic_id is not distinct from v_topic and r.due_on = v_date + iv);
  end loop;

  if v_review is not null then
    update public.reviews
       set status = 'concluida', completed_at = now(), completed_session_id = sid
     where id = v_review and status = 'programada';
  end if;

  if v_topic is not null then
    if v_theory then
      update public.topics set status = 'concluido', completed_at = coalesce(completed_at, now()) where id = v_topic;
    else
      update public.topics set status = 'em_andamento' where id = v_topic and status = 'nao_iniciado';
    end if;
  end if;

  perform public.refresh_block_status(v_block);
  return sid;
end $$;

-- ---------- Editar sessão -----------------------------------------------------
create or replace function public.update_study_session(p_id uuid, p jsonb)
returns void
language plpgsql
set search_path = public
as $$
declare
  old_block uuid;
  v_subject uuid := (p ->> 'subject_id')::uuid;
  v_topic uuid := nullif(p ->> 'topic_id', '')::uuid;
  v_block uuid := nullif(p ->> 'block_id', '')::uuid;
  v_date date := coalesce(nullif(p ->> 'studied_on', '')::date, current_date);
  v_theory boolean := coalesce((p ->> 'theory_completed')::boolean, false);
  v_correct int := coalesce((p ->> 'correct')::int, 0);
  v_wrong int := coalesce((p ->> 'wrong')::int, 0);
  v_ranges jsonb := coalesce(p -> 'page_ranges', '[]'::jsonb);
  v_pages int;
  v_video jsonb;
begin
  select block_id into old_block from public.study_sessions where id = p_id;
  if not found then
    raise exception 'Sessão não encontrada.' using errcode = 'P0002';
  end if;

  select coalesce(sum(greatest((r ->> 'end')::int - (r ->> 'start')::int + 1, 0)), 0)
    into v_pages from jsonb_array_elements(v_ranges) r;

  update public.study_sessions
     set subject_id = v_subject,
         topic_id = v_topic,
         block_id = v_block,
         category = coalesce(nullif(p ->> 'category', ''), 'teoria')::public.study_category,
         studied_on = v_date,
         duration_seconds = coalesce((p ->> 'duration_seconds')::int, 0),
         material = nullif(p ->> 'material', ''),
         theory_completed = v_theory,
         pages_read = v_pages,
         page_ranges = v_ranges,
         comment = nullif(p ->> 'comment', '')
   where id = p_id;

  delete from public.questions where session_id = p_id;
  if v_correct + v_wrong > 0 then
    insert into public.questions (session_id, subject_id, topic_id, answered_on, correct, wrong)
    values (p_id, v_subject, v_topic, v_date, v_correct, v_wrong);
  end if;

  delete from public.videos where session_id = p_id;
  for v_video in select * from jsonb_array_elements(coalesce(p -> 'videos', '[]'::jsonb)) loop
    insert into public.videos (session_id, topic_id, title, start_seconds, end_seconds)
    values (p_id, v_topic, v_video ->> 'title',
            coalesce((v_video ->> 'start_seconds')::int, 0),
            coalesce((v_video ->> 'end_seconds')::int, 0));
  end loop;

  if v_topic is not null then
    if v_theory then
      update public.topics set status = 'concluido', completed_at = coalesce(completed_at, now()) where id = v_topic;
    else
      update public.topics set status = 'em_andamento' where id = v_topic and status = 'nao_iniciado';
    end if;
  end if;

  perform public.refresh_block_status(old_block);
  perform public.refresh_block_status(v_block);
end $$;

-- ---------- Excluir sessão ----------------------------------------------------
-- Revisões ainda pendentes geradas por ela saem junto; as já concluídas ficam.
create or replace function public.delete_study_session(p_id uuid)
returns void
language plpgsql
set search_path = public
as $$
declare
  old_block uuid;
begin
  select block_id into old_block from public.study_sessions where id = p_id;
  if not found then
    raise exception 'Sessão não encontrada.' using errcode = 'P0002';
  end if;
  delete from public.reviews where origin_session_id = p_id and status = 'programada';
  delete from public.study_sessions where id = p_id;
  perform public.refresh_block_status(old_block);
end $$;

-- ---------- Planejamento em ciclo --------------------------------------------
-- Payload: name, objective?, weekly_hours, study_days [0-6], min_session_minutes,
--   max_session_minutes, subjects [{subject_id, importance, knowledge, share_percent}],
--   blocks [{subject_id, minutes}] (já na ordem do ciclo)
create or replace function public.create_study_plan(p jsonb)
returns uuid
language plpgsql
set search_path = public
as $$
declare
  pid uuid;
begin
  if jsonb_array_length(coalesce(p -> 'blocks', '[]'::jsonb)) = 0 then
    raise exception 'O ciclo precisa ter ao menos um bloco.' using errcode = 'P0001';
  end if;

  update public.study_plans set is_active = false where is_active;

  insert into public.study_plans
    (name, objective, is_active, weekly_hours, study_days, min_session_minutes, max_session_minutes)
  values
    (p ->> 'name', nullif(p ->> 'objective', ''), true,
     (p ->> 'weekly_hours')::numeric,
     array(select jsonb_array_elements_text(coalesce(p -> 'study_days', '[1,2,3,4,5]'::jsonb))::smallint),
     coalesce((p ->> 'min_session_minutes')::smallint, 30),
     coalesce((p ->> 'max_session_minutes')::smallint, 90))
  returning id into pid;

  insert into public.study_plan_subjects (plan_id, subject_id, importance, knowledge, share_percent)
  select pid, (e ->> 'subject_id')::uuid, (e ->> 'importance')::numeric,
         (e ->> 'knowledge')::numeric, nullif(e ->> 'share_percent', '')::numeric
    from jsonb_array_elements(coalesce(p -> 'subjects', '[]'::jsonb)) e;

  insert into public.study_blocks (plan_id, subject_id, kind, cycle_number, position, planned_minutes)
  select pid, (t.elem ->> 'subject_id')::uuid, 'estudo', 1, (t.ord - 1)::int, (t.elem ->> 'minutes')::smallint
    from jsonb_array_elements(p -> 'blocks') with ordinality as t(elem, ord);

  return pid;
end $$;

-- Recomeça o ciclo: copia os blocos (menos os cancelados) e conta o ciclo como
-- completo só se todos os blocos foram concluídos.
create or replace function public.restart_study_cycle(p_plan uuid)
returns void
language plpgsql
set search_path = public
as $$
declare
  cur int;
  all_done boolean;
begin
  select current_cycle into cur from public.study_plans where id = p_plan;
  if not found then
    raise exception 'Planejamento não encontrado.' using errcode = 'P0002';
  end if;

  select coalesce(bool_and(status in ('concluido', 'cancelado')), false) into all_done
    from public.study_blocks where plan_id = p_plan and cycle_number = cur;

  insert into public.study_blocks (plan_id, subject_id, topic_id, kind, cycle_number, position, planned_minutes)
  select plan_id, subject_id, topic_id, kind, cur + 1, row_number() over (order by position) - 1, planned_minutes
    from public.study_blocks
   where plan_id = p_plan and cycle_number = cur and status <> 'cancelado';

  update public.study_plans
     set current_cycle = cur + 1,
         cycles_completed = cycles_completed + case when all_done then 1 else 0 end
   where id = p_plan;
end $$;

-- ---------- Permissões --------------------------------------------------------
do $$
declare f text;
begin
  foreach f in array array[
    'refresh_block_status(uuid)', 'register_study_session(jsonb)', 'update_study_session(uuid, jsonb)',
    'delete_study_session(uuid)', 'create_study_plan(jsonb)', 'restart_study_cycle(uuid)'
  ] loop
    execute format('revoke execute on function public.%s from public, anon', f);
    execute format('grant execute on function public.%s to authenticated', f);
  end loop;
end $$;

revoke all on all tables in schema public from anon;

