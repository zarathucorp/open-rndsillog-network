# Open Rndsillog Project

## What is Open Rndsillog?

Open Rndsillog(오픈 연구실록)은 오픈소스 기반의 무료 전자연구노트 프로젝트입니다. 전자연구노트 성립을 위한 필수조건 및 간편한 기능을 누구나 이용할 수 있게 하기 위한 목적을 가지고 제작하였습니다. 다음은 오픈 연구실록을 활용하면 얻을 수 있는 기능입니다.

- 연구자 개인별 작업 공간 기능
- Azure 클라우드와 PDF 인증서를 활용한 기록 날짜, 기록자, 위변조 확인 기능
- 연구노트맞춤 자동 PDF 파일 변환 및 저장
- GitHub 개발 내역 자동 저장

> **info** 자세한 내용은 [연구실록 사용자가이드](https://zarathu.gitbook.io/rndsillog-docs)에서 확인할 수 있습니다.

## 오픈 연구실록 구성

- [open-rndsillog-network](https://github.com/zarathucorp/open-rndsillog-network): 연구실록 기능 오케스트레이션
- [indulgentia-front](https://github.com/zarathucorp/indulgentia-front): 연구실록 웹
- [indulgentia-back](https://github.com/zarathucorp/indulgentia-back): 연구실록 서버
- [open-rndsillog-githubapp](https://github.com/zarathucorp/open-rndsillog-githubapp): 연구실록 GitHub App

## 오픈 연구실록 기술스택

- open-rndsillog-network
  - [Docker](https://www.docker.com/)
  - [Nginx](https://www.nginx.com/)
- indulgentia-front
  - [NextJS](https://nextjs.org/)
  - [ShadCN](https://ui.shadcn.com/)
- indulgentia-back
  - [FastAPI](https://fastapi.tiangolo.com/)
  - [Supabase](https://supabase.com/)
  - [Azure](https://azure.microsoft.com/)
  - [Libreoffice](https://www.libreoffice.org/) ([H2Orestart](https://github.com/ebandal/H2Orestart))
- open-rndsillog-githubapp
  - [Probot](https://probot.github.io/)

## 사전작업

### Supabase

Supabase는 PostgreSQL 기반의 오픈소스 백엔드 서비스 제공 플랫폼(BaaS)입니다. 데이터베이스와 연구자 계정을 간편하고 안전하게 이용할 수 있습니다.

#### Table

아래 SQL 스크립트를 Supabase SQL Editor를 통해 적용하면 필요한 Table, Trigger 와 Function을 생성할 수 있습니다.

기본 테이블 생성 SQL 스크립트

```SQL
-- bucket 테이블 생성
CREATE TABLE public.bucket (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE,
    project_id UUID NOT NULL REFERENCES public.project(id) ON UPDATE CASCADE ON DELETE CASCADE,
    manager_id UUID NOT NULL REFERENCES auth.users(id) ON UPDATE CASCADE ON DELETE CASCADE,
    title VARCHAR NOT NULL,
    is_deleted BOOLEAN DEFAULT false NOT NULL
);
ALTER POLICY "Policy for service_role" ON public.bucket TO service_role USING (true);

-- gitrepo 테이블 생성
CREATE TABLE public.gitrepo (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE,
    bucket_id UUID NOT NULL REFERENCES public.bucket(id) ON UPDATE CASCADE ON DELETE CASCADE,
    repo_url VARCHAR NOT NULL,
    user_id UUID NOT NULL REFERENCES auth.users(id) ON UPDATE CASCADE ON DELETE CASCADE,
    git_username VARCHAR NOT NULL,
    git_repository VARCHAR NOT NULL,
    is_deleted BOOLEAN DEFAULT false
);
ALTER POLICY "Policy for service_role" ON public.gitrepo TO service_role USING (true);

-- note 테이블 생성
CREATE TABLE public.note (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE,
    user_id UUID NOT NULL REFERENCES auth.users(id) ON UPDATE CASCADE ON DELETE CASCADE,
    bucket_id UUID NOT NULL REFERENCES public.bucket(id) ON UPDATE CASCADE ON DELETE CASCADE,
    title VARCHAR NOT NULL,
    timestamp_transaction_id VARCHAR,
    file_name VARCHAR NOT NULL,
    is_github BOOLEAN NOT NULL,
    github_type VARCHAR,
    github_hash VARCHAR,
    github_link VARCHAR,
    is_deleted BOOLEAN DEFAULT false NOT NULL,
    pdf_hash VARCHAR
);
ALTER POLICY "Policy for service_role" ON public.note TO service_role USING (true);

-- order 테이블 생성
CREATE TABLE public.order (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE,
    team_id UUID NOT NULL REFERENCES public.team(id) ON UPDATE CASCADE ON DELETE CASCADE,
    order_no VARCHAR NOT NULL,
    status VARCHAR,
    payment_key VARCHAR,
    purchase_datetime TIMESTAMP WITH TIME ZONE DEFAULT now(),
    is_canceled BOOLEAN,
    total_amount NUMERIC NOT NULL,
    refund_amount NUMERIC DEFAULT 0,
    purchase_user_id UUID NOT NULL REFERENCES auth.users(id) ON UPDATE CASCADE ON DELETE CASCADE,
    payment_method VARCHAR,
    currency VARCHAR,
    notes TEXT
);
ALTER POLICY "Policy for service_role" ON public.order TO service_role USING (true);

-- project 테이블 생성
CREATE TABLE public.project (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE,
    team_id UUID NOT NULL REFERENCES public.team(id) ON UPDATE CASCADE ON DELETE CASCADE,
    project_leader VARCHAR,
    title VARCHAR NOT NULL,
    grant_number VARCHAR,
    status VARCHAR,
    start_date DATE,
    end_date DATE,
    is_deleted BOOLEAN DEFAULT false NOT NULL
);
ALTER POLICY "Policy for service_role" ON public.project TO service_role USING (true);

-- subscription 테이블 생성
CREATE TABLE public.subscription (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE,
    subscription_id UUID NOT NULL,
    team_id UUID REFERENCES public.team(id) ON UPDATE CASCADE ON DELETE SET NULL,
    entity REGCLASS NOT NULL,
    started_at TIMESTAMP WITH TIME ZONE NOT NULL,
    filters TEXT[] DEFAULT '{}'::TEXT[],
    expired_at TIMESTAMP WITH TIME ZONE NOT NULL,
    max_members NUMERIC DEFAULT 10 NOT NULL,
    claims JSONB NOT NULL,
    is_active BOOLEAN DEFAULT false NOT NULL,
    claims_role REGROLE NOT NULL,
    order_no VARCHAR NOT NULL
);
ALTER POLICY "Policy for service_role" ON public.subscription TO service_role USING (true);

-- team 테이블 생성
CREATE TABLE public.team (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE,
    name VARCHAR NOT NULL,
    team_leader_id UUID REFERENCES auth.users(id) ON UPDATE CASCADE ON DELETE CASCADE,
    is_deleted BOOLEAN DEFAULT false NOT NULL,
    last_note_created_at TIMESTAMP WITH TIME ZONE
);
ALTER POLICY "Policy for service_role" ON public.team TO service_role USING (true);

-- team_invite 테이블 생성
CREATE TABLE public.team_invite (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
    team_id UUID NOT NULL REFERENCES public.team(id) ON UPDATE CASCADE ON DELETE CASCADE,
    invited_user_id UUID NOT NULL REFERENCES auth.users(id) ON UPDATE CASCADE ON DELETE CASCADE,
    is_accepted BOOLEAN,
    updated_at TIMESTAMP WITH TIME ZONE,
    is_deleted BOOLEAN DEFAULT false NOT NULL
);
ALTER POLICY "Policy for service_role" ON public.team_invite TO service_role USING (true);

-- user_setting 테이블 생성
CREATE TABLE public.user_setting (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    team_id UUID REFERENCES public.team(id) ON UPDATE CASCADE ON DELETE SET NULL,
    has_signature BOOLEAN,
    is_admin BOOLEAN,
    first_name VARCHAR,
    last_name VARCHAR,
    email VARCHAR NOT NULL,
    github_token TEXT,
    is_deleted BOOLEAN DEFAULT false NOT NULL,
    last_note_created_at TIMESTAMP WITH TIME ZONE
);
ALTER POLICY "Policy for service_role" ON public.user_setting TO service_role USING (true);
```

Function & Trigger SQL 스크립트

```SQL
-- update_last_note_created_at 함수 추가
CREATE OR REPLACE FUNCTION public.update_last_note_created_at()
RETURNS trigger
LANGUAGE plpgsql
AS $function$
DECLARE
    team_id_from_user UUID; -- team_id를 저장할 변수를 명확히 선언
BEGIN
    -- user_setting 테이블 업데이트
    UPDATE user_setting
    SET last_note_created_at = NOW()
    WHERE id = NEW.user_id;

    -- team_id 가져오기
    SELECT team_id INTO team_id_from_user
    FROM user_setting
    WHERE id = NEW.user_id;

    -- team 테이블 업데이트
    IF team_id_from_user IS NOT NULL THEN
        UPDATE team
        SET last_note_created_at = NOW()
        WHERE id = team_id_from_user;
    END IF;

    RETURN NEW;
END;
$function$;

-- update_team 함수 추가
CREATE OR REPLACE FUNCTION public.update_team()
RETURNS trigger
LANGUAGE plpgsql
AS $function$
BEGIN
  IF NEW.is_deleted = TRUE THEN
    UPDATE team
    SET is_deleted = TRUE
    WHERE team_invite.team_id = NEW.id;
    UPDATE project
    SET is_deleted = TRUE
    WHERE project.team_id = NEW.id;
  ELSIF NEW.is_deleted = FALSE THEN
    UPDATE team
    SET is_deleted = FALSE
    WHERE team_invite.team_id = NEW.id;
  END IF;
  RETURN NEW;
END;
$function$;

-- update_team_invite 함수 추가
CREATE OR REPLACE FUNCTION public.update_team_invite()
RETURNS trigger
LANGUAGE plpgsql
AS $function$
BEGIN
  IF NEW.is_deleted = TRUE THEN
    UPDATE team_invite
    SET is_deleted = TRUE
    WHERE team_invite.team_id = NEW.id;
  ELSIF NEW.is_deleted = FALSE THEN
    UPDATE team_invite
    SET is_deleted = FALSE
    WHERE team_invite.team_id = NEW.id;
  END IF;
  RETURN NEW;
END;
$function$;

-- update_project 함수 추가
CREATE OR REPLACE FUNCTION public.update_project()
RETURNS trigger
LANGUAGE plpgsql
AS $function$
BEGIN
  IF NEW.is_deleted = TRUE THEN
    UPDATE project
    SET is_deleted = TRUE
    WHERE project.team_id = NEW.id;
  ELSIF NEW.is_deleted = FALSE THEN
    UPDATE project
    SET is_deleted = FALSE
    WHERE project.team_id = NEW.id;
  END IF;
  RETURN NEW;
END;
$function$;

-- get_team_info 함수 추가
CREATE OR REPLACE FUNCTION public.get_team_info(u_team_id uuid)
RETURNS TABLE(id uuid, created_at timestamp with time zone, updated_at timestamp with time zone, is_deleted boolean, team_leader_id uuid, team_leader_first_name character varying, team_leader_last_name character varying, team_name character varying, project_num bigint, bucket_num bigint, note_num bigint, linked_repo_num bigint)
LANGUAGE plpgsql
AS $function$
BEGIN
    RETURN QUERY
    SELECT 
        t.id,
        t.created_at,
        t.updated_at,
        t.is_deleted,
        t.team_leader_id,
        us.first_name AS team_leader_first_name,
        us.last_name AS team_leader_last_name,        
        t.name AS team_name,
        COUNT(DISTINCT p.id) AS project_num,
        COUNT(DISTINCT b.id) AS bucket_num,
        COUNT(DISTINCT n.id) AS note_num,
        COUNT(DISTINCT gr.id) AS linked_repo_num
    FROM public.team t
    LEFT JOIN
        public.user_setting us ON t.team_leader_id = us.id AND us.is_deleted = false
    LEFT JOIN
        public.project p ON t.id = p.team_id AND p.is_deleted = false
    LEFT JOIN 
        public.bucket b ON p.id = b.project_id AND b.is_deleted = false
    LEFT JOIN 
        public.note n ON b.id = n.bucket_id AND n.is_deleted = false
    LEFT JOIN
        public.gitrepo gr ON b.id = gr.bucket_id AND gr.is_deleted = false
    WHERE t.id = u_team_id
    GROUP BY 
        t.id, t.created_at, t.updated_at, t.team_leader_id, us.first_name, us.last_name, t.name;
END;
$function$;

-- get_user_settings_with_auth_info 함수 추가
CREATE OR REPLACE FUNCTION public.get_user_settings_with_auth_info()
RETURNS TABLE(id uuid, team_id uuid, has_signature boolean, is_admin boolean, first_name character varying, last_name character varying, email character varying, github_token text, is_deleted boolean, last_note_created_at timestamp with time zone, last_sign_in_at timestamp with time zone, created_at timestamp with time zone)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public'
AS $function$
BEGIN
    RETURN QUERY
    SELECT 
        us.id,
        us.team_id,
        us.has_signature,
        us.is_admin,
        us.first_name,
        us.last_name,
        us.email,
        us.github_token,
        us.is_deleted,
        us.last_note_created_at,
        au.last_sign_in_at,
        au.created_at
    FROM 
        public.user_setting us
    JOIN 
        auth.users au
    ON 
        us.id = au.id
    WHERE 
        us.is_deleted = False;
END;
$function$;

-- verify_note 함수 추가
CREATE OR REPLACE FUNCTION public.verify_note(p_user_id uuid, p_note_id uuid)
RETURNS boolean
LANGUAGE plpgsql
AS $function$
DECLARE
    data_count INTEGER;
BEGIN
    SELECT COUNT(*)
    INTO data_count
    FROM public.team
    JOIN public.user_setting ON public.team.id = public.user_setting.team_id
    JOIN public.project ON public.team.id = public.project.team_id
    JOIN public.bucket ON public.project.id = public.bucket.project_id
    JOIN public.note ON public.bucket.id = public.note.bucket_id
    WHERE public.user_setting.id = p_user_id AND public.note.id = p_note_id;

    IF data_count > 0 THEN
        RETURN TRUE;
    ELSE
        RETURN FALSE;
    END IF;
END;
$function$;

-- read_note_list_with_user_setting 함수 추가
CREATE OR REPLACE FUNCTION public.read_note_list_with_user_setting(b_id uuid)
RETURNS TABLE(id uuid, created_at timestamp with time zone, updated_at timestamp with time zone, user_id uuid, bucket_id uuid, title character varying, timestamp_transaction_id character varying, file_name character varying, is_github boolean, github_type character varying, github_hash character varying, github_link character varying, is_deleted boolean, pdf_hash character varying, user_setting jsonb)
LANGUAGE plpgsql
AS $function$
BEGIN
    RETURN QUERY
    SELECT
        n.id,
        n.created_at,
        n.updated_at,
        n.user_id,
        n.bucket_id,
        n.title,
        n.timestamp_transaction_id,
        n.file_name,
        n.is_github,
        n.github_type,
        n.github_hash,
        n.github_link,
        n.is_deleted,
        n.pdf_hash,
        jsonb_build_object(
            'team_id', u.team_id,
            'has_signature', u.has_signature,
            'is_admin', u.is_admin,
            'first_name', u.first_name,
            'last_name', u.last_name,
            'email', u.email,
            'github_token', u.github_token,
            'is_deleted', u.is_deleted
        ) AS user_setting
    FROM
        public.note n
    JOIN
        public.user_setting u
    ON
        n.user_id = u.id
    WHERE
        n.bucket_id = b_id
        AND n.is_deleted = false
    ORDER BY
        n.created_at DESC;
END;
$function$;

-- get_note_details 함수 추가
CREATE OR REPLACE FUNCTION public.get_note_details(p_note_id uuid)
RETURNS TABLE(note_id uuid, created_at timestamp with time zone, updated_at timestamp with time zone, user_id uuid, bucket_id uuid, note_title character varying, file_name character varying, is_github boolean, github_type character varying, github_hash character varying, github_link character varying, timestamp_transaction_id character varying, first_name character varying, last_name character varying, bucket_title character varying, project_title character varying)
LANGUAGE plpgsql
AS $function$
BEGIN
    RETURN QUERY
    SELECT
        n.id AS note_id,
        n.created_at,
        n.updated_at,
        n.user_id,
        n.bucket_id,
        n.title AS note_title,
        n.file_name,
        n.is_github,
        n.github_type,
        n.github_hash,
        n.github_link,
        n.timestamp_transaction_id,
        u.first_name,
        u.last_name,
        b.title AS bucket_title,
        p.title AS project_title
    FROM
        public.note n
    JOIN
        public.user_setting u ON n.user_id = u.id
    JOIN
        public.bucket b ON n.bucket_id = b.id
    JOIN
        public.project p ON b.project_id = p.id
    WHERE
        n.id = p_note_id
        AND n.is_deleted = FALSE;
END;
$function$;

-- check_gitrepo_exists 함수 추가
CREATE OR REPLACE FUNCTION public.check_gitrepo_exists(g_bucket_id uuid, g_repo_url character varying)
RETURNS boolean
LANGUAGE plpgsql
AS $function$
DECLARE
    result BOOLEAN;
BEGIN
    SELECT EXISTS (
        SELECT 1 FROM gitrepo 
        WHERE bucket_id = check_gitrepo_exists.g_bucket_id 
        AND repo_url = check_gitrepo_exists.g_repo_url
        AND is_deleted = false
    ) INTO result;

    RETURN result;
END;
$function$;

-- get_bucket_info 함수 추가
CREATE OR REPLACE FUNCTION public.get_bucket_info(b_project_id uuid)
RETURNS TABLE(id uuid, created_at timestamp with time zone, updated_at timestamp with time zone, project_id uuid, manager_id uuid, title character varying, is_deleted boolean, manager_first_name character varying, manager_last_name character varying, gitrepos json, note_num bigint)
LANGUAGE plpgsql
AS $function$
BEGIN
    RETURN QUERY
    SELECT 
        b.id,
        b.created_at,
        b.updated_at,
        b.project_id,
        b.manager_id,
        b.title,
        b.is_deleted,
        us.first_name AS manager_first_name,
        us.last_name AS manager_last_name,
        (
            SELECT json_agg(gr)
            FROM public.gitrepo gr
            WHERE gr.bucket_id = b.id AND gr.is_deleted = false
        ) AS gitrepos,
        (
            SELECT count(*)::BIGINT  -- Change this line
            FROM public.note n
            WHERE n.bucket_id = b.id AND n.is_deleted = false
        ) AS note_num
    FROM public.bucket b
    JOIN public.user_setting us ON b.manager_id = us.id
    WHERE b.project_id = b_project_id AND b.is_deleted = false AND us.is_deleted = false;
END;
$function$;

-- get_bucket_info_list 함수 추가
CREATE OR REPLACE FUNCTION public.get_bucket_info_list(b_project_id uuid)
RETURNS TABLE(id uuid, created_at timestamp with time zone, updated_at timestamp with time zone, project_id uuid, manager_id uuid, title character varying, is_deleted boolean, manager_first_name character varying, manager_last_name character varying, gitrepos json, note_num bigint)
LANGUAGE plpgsql
AS $function$
BEGIN
    RETURN QUERY
    SELECT 
        b.id,
        b.created_at,
        b.updated_at,
        b.project_id,
        b.manager_id,
        b.title,
        b.is_deleted,
        us.first_name AS manager_first_name,
        us.last_name AS manager_last_name,
        (
            SELECT json_agg(gr)
            FROM public.gitrepo gr
            WHERE gr.bucket_id = b.id AND gr.is_deleted = false
        ) AS gitrepos,
        (
            SELECT count(*)::BIGINT  -- Change this line
            FROM public.note n
            WHERE n.bucket_id = b.id AND n.is_deleted = false
        ) AS note_num
    FROM public.bucket b
    JOIN public.user_setting us ON b.manager_id = us.id
    WHERE b.project_id = b_project_id AND b.is_deleted = false AND us.is_deleted = false;
END;
$function$;

-- get_team_invite_and_team_and_user_setting 함수 추가
CREATE OR REPLACE FUNCTION public.get_team_invite_and_team_and_user_setting(user_id uuid)
RETURNS TABLE(id uuid, created_at timestamp with time zone, team_id uuid, invited_user_id uuid, is_accepted boolean, updated_at timestamp with time zone, team_name character varying, team_leader jsonb)
LANGUAGE plpgsql
AS $function$
BEGIN
    RETURN QUERY
    SELECT 
        team_invite.id, 
        team_invite.created_at,
        team_invite.team_id,
        team_invite.invited_user_id,
        team_invite.is_accepted,
        team_invite.updated_at,
        team.name,
        jsonb_build_object('id', team.team_leader_id, 'first_name', user_setting.first_name, 'last_name', user_setting.last_name)
    FROM 
        team_invite
    JOIN 
        team ON team_invite.team_id = team.id
    JOIN 
        user_setting ON team.team_leader_id = user_setting.id
    WHERE 
        team_invite.is_deleted IS false AND
        team_invite.invited_user_id = user_id AND
        team_invite.is_accepted IS NULL
    ORDER BY 
        team_invite.created_at DESC;
END;
$function$;

-- get_team_invite_send_and_team_and_user_setting 함수 추가
CREATE OR REPLACE FUNCTION public.get_team_invite_send_and_team_and_user_setting(sent_team_id uuid)
RETURNS TABLE(id uuid, created_at timestamp with time zone, team_id uuid, invited_user_id uuid, is_accepted boolean, updated_at timestamp with time zone, team_name character varying, invited_user jsonb)
LANGUAGE plpgsql
AS $function$
BEGIN
    RETURN QUERY
    SELECT 
        team_invite.id, 
        team_invite.created_at,
        team_invite.team_id,
        team_invite.invited_user_id,
        team_invite.is_accepted,
        team_invite.updated_at,
        team.name,
        jsonb_build_object('id', user_setting.id, 'first_name', user_setting.first_name, 'last_name', user_setting.last_name, 'email', user_setting.email)
    FROM 
        team_invite
    JOIN 
        team ON team_invite.team_id = team.id
    JOIN 
        user_setting ON team_invite.invited_user_id = user_setting.id
    WHERE 
        team_invite.is_deleted IS false AND
        team_invite.team_id = sent_team_id AND
        team_invite.is_accepted IS NULL
    ORDER BY 
        team_invite.created_at DESC;
END;
$function$;

-- update_user_setting_on_user_delete 함수 추가
CREATE OR REPLACE FUNCTION public.update_user_setting_on_user_delete()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
AS $function$
BEGIN
  -- auth.users 테이블에서 row가 삭제될 때, 해당 row의 id와 일치하는 public.user_setting의 row를 찾아
  -- is_deleted를 true로 설정합니다.
  UPDATE public.user_setting
  SET is_deleted = TRUE
  WHERE id = OLD.id;
  RETURN OLD;
END;
$function$;

-- fn_soft_delete_user_setting 함수 추가
CREATE OR REPLACE FUNCTION public.fn_soft_delete_user_setting()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
AS $function$
BEGIN
  -- auth.users 테이블의 row가 업데이트 되어 deleted_at이 null이 아니게 설정될 때,
  -- 해당 row의 id와 일치하는 public.user_setting의 row를 찾아 is_deleted를 true로 설정합니다.
  IF NEW.deleted_at IS NOT NULL THEN
    UPDATE public.user_setting
    SET is_deleted = TRUE
    WHERE id = NEW.id;
  END IF;
  RETURN NEW;
END;
$function$;

-- get_team_user_count 함수 추가
CREATE OR REPLACE FUNCTION public.get_team_user_count(u_team_id uuid)
RETURNS integer
LANGUAGE plpgsql
AS $function$
DECLARE
    -- 변수 선언
    total_count INTEGER;
BEGIN
    -- user_setting에서 조건에 맞는 행의 수를 찾음
    SELECT COUNT(*) INTO total_count FROM public.user_setting
    WHERE team_id = u_team_id AND is_deleted = FALSE;

    -- team_invite에서 조건에 맞는 행의 수를 찾아서 기존 수에 더함
    SELECT total_count + COUNT(*) INTO total_count FROM public.team_invite
    WHERE team_id = u_team_id AND is_accepted IS NULL AND is_deleted = FALSE;

    -- 최종 계산된 총 수를 반환
    RETURN total_count;
END;
$function$;

-- verify_bucket 함수 추가
CREATE OR REPLACE FUNCTION public.verify_bucket(user_id uuid, bucket_id uuid)
RETURNS boolean
LANGUAGE plpgsql
AS $function$
DECLARE
    data_count INTEGER;
BEGIN
    SELECT COUNT(*)
    INTO data_count
    FROM public.team
    JOIN public.user_setting ON public.team.id = public.user_setting.team_id
    JOIN public.project ON public.team.id = public.project.team_id
    JOIN public.bucket ON public.project.id = public.bucket.project_id
    WHERE public.user_setting.id = user_id AND public.bucket.id = bucket_id;

    IF data_count > 0 THEN
        RETURN TRUE;
    ELSE
        RETURN FALSE;
    END IF;
END;
$function$;

-- verify_project 함수 추가
CREATE OR REPLACE FUNCTION public.verify_project(user_id uuid, project_id uuid)
RETURNS boolean
LANGUAGE plpgsql
AS $function$
DECLARE
    data_count INTEGER;
BEGIN
    SELECT COUNT(*)
    INTO data_count
    FROM public.team
    JOIN public.user_setting ON public.team.id = public.user_setting.team_id
    JOIN public.project ON public.team.id = public.project.team_id
    WHERE public.user_setting.id = user_id AND public.project.id = project_id;

    IF data_count > 0 THEN
        RETURN TRUE;
    ELSE
        RETURN FALSE;
    END IF;
END;
$function$;

-- note_breadcrumb_data 함수 추가
CREATE OR REPLACE FUNCTION public.note_breadcrumb_data(note_id uuid)
RETURNS TABLE(note_title character varying, project_id uuid, project_title character varying, bucket_id uuid, bucket_title character varying)
LANGUAGE plpgsql
AS $function$
BEGIN
    RETURN QUERY
    SELECT 
        n.title AS note_title,
        p.id AS project_id,
        p.title AS project_title,
        b.id AS bucket_id,
        b.title AS bucket_title
    FROM 
        note n
    JOIN 
        bucket b ON n.bucket_id = b.id
    JOIN 
        project p ON b.project_id = p.id
    WHERE 
        n.id = note_id;
END;
$function$;

-- bucket_breadcrumb_data 함수 추가
CREATE OR REPLACE FUNCTION public.bucket_breadcrumb_data(bucket_id uuid)
RETURNS TABLE(project_id uuid, project_title character varying, bucket_title character varying)
LANGUAGE plpgsql
AS $function$
BEGIN
    RETURN QUERY
    SELECT 
        p.id AS project_id,
        p.title AS project_title,
        b.title AS bucket_title
    FROM 
        bucket b
    JOIN 
        project p ON b.project_id = p.id
    WHERE 
        b.id = bucket_id;
END;
$function$;

-- update_bucket 함수 추가
CREATE OR REPLACE FUNCTION public.update_bucket()
RETURNS trigger
LANGUAGE plpgsql
AS $function$
BEGIN
  IF NEW.is_deleted = TRUE THEN
    UPDATE bucket
    SET is_deleted = TRUE
    WHERE bucket.project_id = NEW.id;
  ELSIF NEW.is_deleted = FALSE THEN
    UPDATE bucket
    SET is_deleted = FALSE
    WHERE bucket.project_id = NEW.id;
  END IF;
  RETURN NEW;
END;
$function$;

-- update_note 함수 추가
CREATE OR REPLACE FUNCTION public.update_note()
RETURNS trigger
LANGUAGE plpgsql
AS $function$
BEGIN
  IF NEW.is_deleted = TRUE THEN
    UPDATE note
    SET is_deleted = TRUE
    WHERE note.bucket_id = NEW.id;
  ELSIF NEW.is_deleted = FALSE THEN
    UPDATE note
    SET is_deleted = FALSE
    WHERE note.bucket_id = NEW.id;
  END IF;
  RETURN NEW;
END;
$function$;

-- handle_new_user 함수 추가
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public'
AS $function$
BEGIN
  IF new.raw_app_meta_data->>'provider' = 'email' THEN
    INSERT INTO public.user_setting (id, team_id, has_signature, is_admin, first_name, last_name, email)
    VALUES (new.id, null, false, false, new.raw_user_meta_data->>'first_name', new.raw_user_meta_data->>'last_name', new.email);
  ELSE
    INSERT INTO public.user_setting (id, team_id, has_signature, is_admin, first_name, last_name, email)
    VALUES (new.id, null, false, false, new.raw_user_meta_data->>'name', null, new.email);
  END IF;
  RETURN new;
END;
$function$;
```

RLS policy

```SQL
alter policy "Policy for service_role"
on "public"."bucket"
to service_role
using (
  true
);

alter policy "Policy for service_role"
on "public"."gitrepo"
to service_role
using (
  true
);

alter policy "Policy for service_role"
on "public"."note"
to service_role
using (
  true
);

alter policy "Policy for service_role"
on "public"."order"
to service_role
using (
  true
);

alter policy "Policy for service_role"
on "public"."project"
to service_role
using (
  true
);

alter policy "Policy for service_role"
on "public"."subscription"
to service_role
using (
  true
);

alter policy "Policy for service_role"
on "public"."team"
to service_role
using (
  true
);

alter policy "Policy for service_role"
on "public"."team_invite"
to service_role
using (
  true
);

alter policy "Policy for service_role"
on "public"."user_setting"
to service_role
using (
  true
);
```

#### Authentication

Provider

open-rndsillog은 이메일과 구글을 기본으로 설정되어 있습니다. supabase 설정과 open-rndsillog 스크립트 수정을 통해 더 많은 Auth Providers를 추가할 수 있습니다. Supabase Authentication > CONFIGURATION > Providers 에서 설정할 수 있습니다.

Email

SMTP 설정을 해야 연구자 계정을 정상적으로 이용할 수 있습니다.
Supabase Setting > Authentication > SMTP Provider Settings를 통해 설정할 수 있습니다.

#### Supabase 환경변수
- SUPABASE_URL
- SUPABASE_KEY (ANON)
- SUPABASE_KEY (SERVICE)

### Azure Cloud

#### Azure blob storage

[Azure blob storage](https://azure.microsoft.com/ko-kr/products/storage/blobs) 생성 후 사용할 컨테이너를 만듭니다. 이후 open-rndsillog이 접근하기 위해 [AZURE_STORAGE_CONNECTION_STRING](https://learn.microsoft.com/ko-kr/azure/developer/python/sdk/examples/azure-sdk-example-storage-use?tabs=connection-string%2Ccmd)을 준비합니다.
최종적으로 다음 항목들이 환경변수에 활용됩니다.

- AZURE_RESOURCE_GROUP
- AZURE_STORAGE_CONNECTION_STRING
- DEFAULT_AZURE_CONTAINER_NAME

#### Azure confidential ledger

[Azure confidential ledger](https://azure.microsoft.com/ko-kr/products/azure-confidential-ledger) 생성 후 open-rndsillog이 접근하기 위해 [Azure 애플리케이션 등록](https://learn.microsoft.com/ko-kr/azure/confidential-ledger/register-application)을 진행합니다. 해당 애플리케이션 등록이 완료되면 다음 항목들이 환경변수에 활용됩니다.

- AZURE_CLIENT_ID
- AZURE_TENANT_ID
- RNDSillog-Note.pem

## How to setup

1. 각각의 repo를 같은 경로에 git clone을 통해 불러옵니다. 이때 환경변수(.env)을 각 repo에 있는 설명에 따라 설정해줍니다.
2. docker 및 docker compose 설치를 진행합니다.
3. open-rndsillog-network 디렉토리로 이동 후 docker-compose.yml 스크립트를 이용해 실행합니다.
4. 해당 서버의 DNS 설정을 진행하면 외부에서도 접속할 수 있습니다.

진행 예시

```bash
git clone https://github.com/zarathucorp/open-rndsillog-network.git
git clone https://github.com/zarathucorp/indulgentia-front.git
git clone https://github.com/zarathucorp/indulgentia-back.git
git clone https://github.com/zarathucorp/open-rndsillog-githubapp.git

cd open-rndsillog-network
docker compose up -d

(...)

docker ps
# 실행 결과
# CONTAINER ID   IMAGE                            COMMAND                  CREATED          STATUS          PORTS                                                                      NAMES
# a1b2c3d4e5f6   nginx:1.25.4                     "/docker-entrypoint.…"   10 seconds ago   Up 10 seconds   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   rndsillog_nginx
# b2c3d4e5f6a7   rndsillog-network-githubworker   "docker-entrypoint.s…"   10 seconds ago   Up 10 seconds                                                                              github
# c3d4e5f6a7b8   rndsillog-network-backend        "uvicorn main:app --…"   10 seconds ago   Up 10 seconds   8000/tcp                                                                   back
# d4e5f6a7b8c9   rndsillog-network-frontend       "docker-entrypoint.s…"   10 seconds ago   Up 10 seconds   3000/tcp                                                                   front
```

## License

[MIT](LICENSE) © 2025 Zarathu Co.,Ltd