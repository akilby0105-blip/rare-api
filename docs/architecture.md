# Rare System Architecture

```mermaid
graph LR

  subgraph Client ["Browser — React 18 (localhost:3000)"]
    direction TB
    LS[("localStorage\nauth_token\ncurrent_user_id\nis_staff")]
    Guards["Route Guards\nAuthorized · AdminOnly\n(redirect if no token / not staff)"]
    Components["Page Components\nPostCreate · PostDetail\nUnapprovedPostList · etc."]
    Managers["API Managers\nPostManager · UserManager\nCategoryManager · etc.\n──────────────\nOne file per resource.\nAll calls go through\napi.js for base URL\n+ auth header."]

    LS -->|"reads token\non every request"| Managers
    Guards -->|"renders"| Components
    Components -->|"calls named\nasync functions"| Managers
  end

  subgraph API ["Django 4.2 + DRF (localhost:8088)"]
    direction TB
    CORS["CorsMiddleware\nAllows localhost:3000 only"]
    TokenAuth["TokenAuthentication\nLooks up token in DB,\npopulates request.user"]
    Router["URL Router\nurls.py\n73 routes"]

    subgraph Views ["View Functions  (@api_view)"]
      AuthV["auth_views\nlogin · register · me"]
      PostV["post_views\nCRUD · approve/unapprove\nsearch · image upload"]
      CommentV["comment_views\nlist by post · detail"]
      UserV["user_views\nprofiles · deactivate\nchange role · demotion queue"]
      OtherV["tag_views\ncategory_views\nreaction_views\nsubscription_views"]
    end

    subgraph Serializers ["Serializers (DRF ModelSerializer)"]
      SDetail["Detail serializers\nFull fields + nested objects\nUsed on create/update/detail"]
      SList["List serializers\nSlim fields only\nUsed on list endpoints"]
    end

    ORM["Django ORM\nModel.objects.create/get/filter/save"]

    CORS --> TokenAuth --> Router
    Router --> AuthV & PostV & CommentV & UserV & OtherV
    PostV & CommentV & UserV & OtherV --> SDetail & SList
    AuthV & PostV & CommentV & UserV & OtherV --> ORM
  end

  subgraph DB ["PostgreSQL (localhost:5432, db: rare)"]
    direction TB
    Users["rareapi_rareuser\nextends auth_user\n+ bio, profile_image_url,\ncreated_on"]
    Posts["rareapi_post\ntitle · content · approved\npublication_date · image_url\n▶ FK: user, category"]
    Comments["rareapi_comment\nsubject · content · created_on\n▶ FK: post, author"]
    JoinTables["Join tables\nrareapi_posttag  (post ↔ tag)\nrareapi_postreaction  (user ↔ post ↔ reaction)"]
    Lookup["Lookup tables\nrareapi_category\nrareapi_tag\nrareapi_reaction"]
    Social["Social tables\nrareapi_subscription  (follower → author)\nrareapi_demotionqueue  (admin voting)"]
    TokenTable["authtoken_token\nDRF-managed\ntoken ↔ user"]
  end

  FS[("Filesystem\nmedia/post_images/\nServed at /media/")]

  %% Client ↔ API
  Managers -->|"HTTP fetch\nPOST/GET/PUT/DELETE\nAuthorization: Token …\nContent-Type: application/json"| CORS
  SDetail & SList -->|"JSON response"| Managers

  %% API ↔ DB
  ORM -->|"SQL via psycopg2"| Users & Posts & Comments & JoinTables & Lookup & Social
  TokenAuth -->|"SELECT"| TokenTable

  %% API ↔ Filesystem
  PostV -->|"PUT /posts/:id/image\nwrites file"| FS
  FS -->|"absolute URL\nstored in post.image_url"| Posts
```

## Key facts for new developers

| Topic | Detail |
|---|---|
| Auth flow | Login returns a token; client stores it in `localStorage`; every subsequent request sends `Authorization: Token <value>` |
| Admin check | `is_staff` is returned at login and stored client-side. Server re-checks on every admin endpoint. Client-side value goes stale if a role changes mid-session. |
| Post approval | Regular users' posts are created with `approved=False`. Admins' posts are auto-approved. Admins approve via `PUT /posts/:id/approve`. |
| Unapproved post visibility | List endpoints filter to `approved=True`. The single-post detail endpoint (`GET /posts/:id`) does **not** — any authenticated user can read an unapproved post by ID. |
| Image upload | Two separate API calls: `POST /posts` creates the record, then `PUT /posts/:id/image` uploads the file. No rollback if the second call fails. |
| Subscriptions | Soft-deleted: `ended_on` is set instead of deleting the row. Active subscriptions filter on `ended_on__isnull=True`. No other model uses this pattern. |
| Serializers | Each resource has a `DetailSerializer` (full, nested) and a `ListSerializer` (slim). Detail is used for create/update/single responses; List is used for collection endpoints. |
| Error handling | No `.catch()` on any client-side `fetch` call. Failed requests are silent. |
