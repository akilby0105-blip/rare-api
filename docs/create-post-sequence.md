# Sequence: User Creates a New Post

```mermaid
sequenceDiagram
    actor       User
    participant PostCreate  as PostCreate.js
    participant PostManager as PostManager.js
    participant LS          as localStorage
    participant CORS        as CorsMiddleware
    participant TokenAuth   as TokenAuthentication
    participant Router      as urls.py
    participant PostView    as post_views.py
    participant DB          as PostgreSQL
    participant FS          as Filesystem

    User->>PostCreate: clicks Save (form onSubmit)
    activate PostCreate

    Note over PostCreate: reads title, category_id, content<br/>from useRef handles — no onChange tracking

    PostCreate->>PostManager: createPost({ title, category_id, content })
    activate PostManager

    PostManager->>LS: getItem("auth_token")
    LS-->>PostManager: "abc123…"

    PostManager->>CORS: POST http://localhost:8088/posts<br/>Authorization: Token abc123…<br/>Content-Type: application/json<br/>{ "title": "…", "category_id": 3, "content": "…" }
    activate CORS

    CORS->>CORS: check Origin: http://localhost:3000<br/>✓ in CORS_ORIGIN_WHITELIST (settings.py)
    CORS->>TokenAuth: forward request
    deactivate CORS
    activate TokenAuth

    TokenAuth->>DB: SELECT key, user_id FROM authtoken_token<br/>WHERE key = 'abc123…'
    DB-->>TokenAuth: token row → user_id

    TokenAuth->>DB: SELECT * FROM rareapi_rareuser<br/>WHERE id = &lt;user_id&gt;
    DB-->>TokenAuth: RareUser row

    TokenAuth->>TokenAuth: set request.user = RareUser instance
    TokenAuth->>Router: forward request
    deactivate TokenAuth
    activate Router

    Router->>PostView: POST /posts → post_list(request)
    deactivate Router
    activate PostView

    PostView->>DB: SELECT * FROM rareapi_category<br/>WHERE id = &lt;category_id&gt;
    DB-->>PostView: Category row

    Note over PostView,DB: If category does not exist,<br/>view returns HTTP 400 here and stops.

    PostView->>DB: INSERT INTO rareapi_post<br/>(user_id, category_id, title, content,<br/>image_url, publication_date, approved)<br/>VALUES (…)
    Note over PostView: approved  = request.user.is_staff<br/>True  → admin post, immediately public<br/>False → enters moderation queue

    DB-->>PostView: new Post row with assigned id

    PostView->>PostView: PostDetailSerializer(post).data<br/>→ nests UserSummarySerializer(user)<br/>→ nests CategorySerializer(category)<br/>→ runs get_tags() — returns [] for new post

    PostView-->>PostManager: HTTP 201<br/>{ id, title, content, approved,<br/>  publication_date, image_url,<br/>  user: {…}, category: {…}, tags: [] }
    deactivate PostView

    PostManager-->>PostCreate: post object (parsed JSON)
    deactivate PostManager

    alt user attached an image file

        PostCreate->>PostManager: uploadPostImage(post.id, formData)
        activate PostManager

        PostManager->>LS: getItem("auth_token")
        LS-->>PostManager: "abc123…"

        PostManager->>CORS: PUT http://localhost:8088/posts/&lt;id&gt;/image<br/>Authorization: Token abc123…<br/>Content-Type: multipart/form-data<br/>[binary image data]
        activate CORS
        CORS->>TokenAuth: (same middleware chain — origin check + token lookup)
        deactivate CORS
        activate TokenAuth
        TokenAuth->>PostView: PUT /posts/&lt;id&gt;/image → upload_post_image(request, pk)
        deactivate TokenAuth
        activate PostView

        PostView->>DB: SELECT * FROM rareapi_post<br/>WHERE id = &lt;id&gt;
        DB-->>PostView: Post row

        PostView->>FS: write binary to<br/>media/post_images/post_&lt;id&gt;_&lt;filename&gt;
        FS-->>PostView: file saved

        PostView->>DB: UPDATE rareapi_post<br/>SET image_url = 'http://…/media/post_images/…'<br/>WHERE id = &lt;id&gt;
        DB-->>PostView: 1 row updated

        PostView-->>PostManager: HTTP 200 { "image_url": "http://…" }
        deactivate PostView

        PostManager-->>PostCreate: { image_url }
        deactivate PostManager

    else no image selected

        Note over PostCreate: skips the upload call entirely

    end

    PostCreate->>PostCreate: navigate(`/posts/${post.id}`)
    deactivate PostCreate
```

## Files involved

| Step | File |
|---|---|
| Form UI + submit handler | `rare-client/src/components/posts/PostCreate.js` |
| `createPost()` + `uploadPostImage()` | `rare-client/src/managers/PostManager.js` |
| Base URL + auth header helper | `rare-client/src/managers/api.js` |
| Token stored and read | `localStorage` (set at login by `AuthManager.js`) |
| CORS origin check | `django-cors-headers` — configured in `rareproject/settings.py` |
| Token → user lookup | DRF `TokenAuthentication` — `authtoken_token` table |
| Route dispatch | `rareapi/urls.py` — `path('posts', post_list)` |
| Post creation logic | `rareapi/views/post_views.py` — `post_list()` |
| Image upload logic | `rareapi/views/post_views.py` — `upload_post_image()` |
| Response shaping | `rareapi/serializers/post_serializers.py` — `PostDetailSerializer` |
| Nested user in response | `rareapi/serializers/user_serializers.py` — `UserSummarySerializer` |
| Nested category in response | `rareapi/serializers/category_serializers.py` — `CategorySerializer` |
| DB models written | `rareapi/models/post.py`, `rareapi/models/category.py` |
| Image written to disk | `media/post_images/` (path set by `MEDIA_ROOT` in `settings.py`) |

## Things that are not shown

- **No error handling on the client.** If either `POST /posts` or `PUT /posts/:id/image` fails, the `.then()` chain simply does not continue. There is no `.catch()`, no user-facing message, and no rollback. A failed image upload leaves an approved (or queued) post in the database with an empty `image_url`.
- **No loading state.** The Save button is not disabled during the fetch calls. A user can click it multiple times and create duplicate posts.
- **The middleware chain is abbreviated.** Django runs the full `MIDDLEWARE` stack (security, session, CSRF, etc.) on every request. Only CORS and token auth are shown here because they are the two that directly affect this flow.
- **`publication_date` is always today.** The view sets `publication_date=timezone.now().date()` server-side; whatever the client sends in the body for this field is ignored.
