# Rare Database Schema

```mermaid
erDiagram

    RAREUSER {
        int         id              PK
        string      username
        string      email
        string      first_name
        string      last_name
        bool        is_staff
        bool        is_active
        string      bio
        string      profile_image_url
        date        created_on
    }

    CATEGORY {
        int     id      PK
        string  label
    }

    TAG {
        int     id      PK
        string  label
    }

    REACTION {
        int     id          PK
        string  label
        string  image_url
    }

    POST {
        int     id                  PK
        int     user_id             FK
        int     category_id         FK
        string  title
        text    content
        date    publication_date
        string  image_url
        bool    approved
    }

    COMMENT {
        int         id          PK
        int         post_id     FK
        int         author_id   FK
        string      subject
        text        content
        datetime    created_on
    }

    POSTTAG {
        int     id       PK
        int     post_id  FK
        int     tag_id   FK
    }

    POSTREACTION {
        int     id           PK
        int     user_id      FK
        int     post_id      FK
        int     reaction_id  FK
    }

    SUBSCRIPTION {
        int         id           PK
        int         follower_id  FK
        int         author_id    FK
        date        created_on
        datetime    ended_on
    }

    DEMOTIONQUEUE {
        int     id               PK
        int     admin_id         FK
        int     approver_one_id  FK
        string  action
    }

    RAREUSER    ||--o{ POST          : "writes"
    CATEGORY    ||--o{ POST          : "categorizes"
    POST        ||--o{ COMMENT       : "has"
    RAREUSER    ||--o{ COMMENT       : "authors"
    POST        ||--o{ POSTTAG       : "tagged via"
    TAG         ||--o{ POSTTAG       : "applied via"
    POST        ||--o{ POSTREACTION  : "receives"
    RAREUSER    ||--o{ POSTREACTION  : "reacts via"
    REACTION    ||--o{ POSTREACTION  : "used in"
    RAREUSER    ||--o{ SUBSCRIPTION  : "follows (follower_id)"
    RAREUSER    ||--o{ SUBSCRIPTION  : "followed by (author_id)"
    RAREUSER    ||--o{ DEMOTIONQUEUE : "is targeted (admin_id)"
    RAREUSER    ||--o{ DEMOTIONQUEUE : "votes (approver_one_id)"
```

## Notes

**RAREUSER** extends Django's `AbstractUser`. The fields shown above are the additions and the most relevant inherited fields. `password` (a bcrypt hash) is omitted for clarity.

**POSTTAG** and **POSTREACTION** are explicit through-models rather than Django `ManyToManyField` relations. The ORM does not expose `.tags` or `.reactions` shorthand — views query `PostTag.objects` and `PostReaction.objects` directly.

**SUBSCRIPTION.ended_on** is nullable. A null value means the subscription is active; a set value means it was cancelled. This is the only soft-delete pattern in the schema — all other models use hard deletes.

**DEMOTIONQUEUE** has a `unique_together` constraint on `(action, admin_id, approver_one_id)` preventing duplicate votes. The `action` field is a free-form string with no enforced values. Only one approver field exists (`approver_one_id`), suggesting a multi-approver voting system was planned but not completed.

**POST.approved** controls visibility. List endpoints filter to `approved = true`. The single-record detail endpoint does not filter on this field — any authenticated user can fetch an unapproved post by ID.

**POST.publication_date** is set to today's date by the server on creation. List endpoints also filter `publication_date <= today`, which means the field could support scheduled publishing, but no UI or API surface for that exists yet.
