# Open Rndsillog Project

## What is Open Rndsillog?

Open Rndsillog is an open-source based free electronic research notebook project. It was created with the purpose of making essential conditions for establishing electronic research notebooks and simple functions accessible to everyone. The following are the features you can get by using Open Rndsillog:

- Personal workspace function for each researcher
- Record date, recorder, and tamper verification using Azure cloud and PDF certificates
- Automatic PDF file conversion and storage tailored to research notes
- Automatic saving of GitHub development history

> **info** For more details, please refer to the [Research Notebook User Guide](https://zarathu.gitbook.io/rndsillog-docs).

## Open Rndsillog Components

- [open-rndsillog-network](https://github.com/zarathucorp/open-rndsillog-network): Research Notebook Function Orchestration
- [indulgentia-front](https://github.com/zarathucorp/indulgentia-front): Research Notebook Web
- [indulgentia-back](https://github.com/zarathucorp/indulgentia-back): Research Notebook Server
- [indulgentia-github](https://github.com/zarathucorp/indulgentia-github): Research Notebook GitHub App

## Open Rndsillog Tech Stack

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
- indulgentia-github
  - [Probot](https://probot.github.io/)

## Prerequisites

### Supabase

Supabase is an open-source backend service platform (BaaS) based on PostgreSQL. It allows easy and secure use of databases and researcher accounts.

#### Table

Applying the following SQL script through the Supabase SQL Editor will create the necessary Tables, Triggers, and Functions.

Basic table creation SQL script

```SQL
-- Create bucket table
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

-- Create gitrepo table
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

-- Create note table
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

-- Create order table
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

-- Create project table
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

-- Create subscription table
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

-- Create team table
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

-- Create team_invite table
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

-- Create user_setting table
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

Function & Trigger SQL script

```SQL
-- Add update_last_note_created_at function
CREATE OR REPLACE FUNCTION public.update_last_note_created_at()
RETURNS trigger
LANGUAGE plpgsql
AS $function$
DECLARE
    team_id_from_user UUID; -- Declare a variable to store team_id
BEGIN
    -- Update user_setting table
    UPDATE user_setting
    SET last_note_created_at = NOW()
    WHERE id = NEW.user_id;

    -- Get team_id
    SELECT team_id INTO team_id_from_user
    FROM user_setting
    WHERE id = NEW.user_id;

    -- Update team table
    IF team_id_from_user IS NOT NULL THEN
        UPDATE team
        SET last_note_created_at = NOW()
        WHERE id = team_id_from_user;
    END IF;

    RETURN NEW;
END;
$function$;

-- Add update_team function
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

-- Add update_team_invite function
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

-- Add update_project function
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

-- Add get_team_info function
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

-- Add get_user_settings_with_auth_info function
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

-- Add verify_note function
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

-- Add read_note_list_with_user_setting function
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

-- Add get_note_details function
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

-- Add check_gitrepo_exists function
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

-- Add get_bucket_info function
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
            WHERE n.bucket_id = b.id
            AND n.is_deleted = false
        ) AS note_num
    FROM public.bucket b
    JOIN public.user_setting us ON b.manager_id = us.id
    WHERE b.project_id = b_project_id AND b.is_deleted = false AND us.is_deleted = false;
END;
$function$;

-- Add get_bucket_info_list function
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
            WHERE n.bucket_id = b.id
            AND n.is_deleted = false
        ) AS note_num
    FROM public.bucket b
    JOIN public.user_setting us ON b.manager_id = us.id
    WHERE b.project_id = b_project_id AND b.is_deleted = false AND us.is_deleted = false;
END;
$function$;

-- Add get_team_invite_and_team_and_user_setting function
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

-- Add get_team_invite_send_and_team_and_user_setting function
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

-- Add update_user_setting_on_user_delete function
CREATE OR REPLACE FUNCTION public.update_user_setting_on_user_delete()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
AS $function$
BEGIN
  -- When a row is deleted from auth.users table, find the corresponding row in public.user_setting and set is_deleted to true.
  UPDATE public.user_setting
  SET is_deleted = TRUE
  WHERE id = OLD.id;
  RETURN OLD;
END;
$function$;

-- Add fn_soft_delete_user_setting function
CREATE OR REPLACE FUNCTION public.fn_soft_delete_user_setting()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
AS $function$
BEGIN
  -- When a row in auth.users table is updated and deleted_at is set to a non-null value, find the corresponding row in public.user_setting and set is_deleted to true.
  IF NEW.deleted_at IS NOT NULL THEN
    UPDATE public.user_setting
    SET is_deleted = TRUE
    WHERE id = NEW.id;
  END IF;
  RETURN NEW;
END;
$function$;

-- Add get_team_user_count function
CREATE OR REPLACE FUNCTION public.get_team_user_count(u_team_id uuid)
RETURNS integer
LANGUAGE plpgsql
AS $function$
DECLARE
    -- Declare a variable
    total_count INTEGER;
BEGIN
    -- Find the number of rows in user_setting that match the condition
    SELECT COUNT(*) INTO total_count FROM public.user_setting
    WHERE team_id = u_team_id AND is_deleted = FALSE;

    -- Find the number of rows in team_invite that match the condition and add to the existing count
    SELECT total_count + COUNT(*) INTO total_count FROM public.team_invite
    WHERE team_id = u_team_id AND is_accepted IS NULL AND is_deleted = FALSE;

    -- Return the final calculated total count
    RETURN total_count;
END;
$function$;

-- Add verify_bucket function
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

-- Add verify_project function
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

-- Add note_breadcrumb_data function
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

-- Add bucket_breadcrumb_data function
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

-- Add update_bucket function
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

-- Add update_note function
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

-- Add handle_new_user function
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

open-rndsillog is set up with email and Google by default. You can add more Auth Providers through Supabase settings and open-rndsillog script modifications. This can be configured in Supabase Authentication > CONFIGURATION > Providers.

Email

SMTP settings must be configured to use researcher accounts properly. This can be configured in Supabase Setting > Authentication > SMTP Provider Settings.

#### Supabase Environment Variables

- SUPABASE_URL
- SUPABASE_KEY (ANON)
- SUPABASE_KEY (SERVICE)

### Azure Cloud

#### Azure blob storage

Create [Azure blob storage](https://azure.microsoft.com/ko-kr/products/storage/blobs) and then create a container to use. Then prepare the [AZURE_STORAGE_CONNECTION_STRING](https://learn.microsoft.com/ko-kr/azure/developer/python/sdk/examples/azure-sdk-example-storage-use?tabs=connection-string%2Ccmd) for open-rndsillog to access.
Finally, the following items will be used as environment variables.

- AZURE_RESOURCE_GROUP
- AZURE_STORAGE_CONNECTION_STRING
- DEFAULT_AZURE_CONTAINER_NAME

#### Azure confidential ledger

Create [Azure confidential ledger](https://azure.microsoft.com/ko-kr/products/azure-confidential-ledger) and then register the application for open-rndsillog to access by following [Azure application registration](https://learn.microsoft.com/ko-kr/azure/confidential-ledger/register-application). Once the application registration is complete, the following items will be used as environment variables.

- AZURE_CLIENT_ID
- AZURE_TENANT_ID
- RNDSillog-Note.pem

## How to setup

1. Clone each repo to the same path using git clone. At this time, set the environment variables (.env) according to the description in each repo.
2. Install docker and docker compose.
3. Move to the open-rndsillog-network directory and run it using the docker-compose.yml script.
4. Configure the DNS settings of the server to allow external access.

Example

```bash
git clone https://github.com/zarathucorp/open-rndsillog-network.git
git clone https://github.com/zarathucorp/indulgentia-front.git
git clone https://github.com/zarathucorp/indulgentia-back.git
git clone https://github.com/zarathucorp/indulgentia-github.git

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