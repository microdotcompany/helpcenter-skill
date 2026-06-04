---
name: helpcenter
description: >
  When the user wants to create, update, read, or manage help center articles
  or customer conversations via the Help.Center API. Use when the user says
  "write a help article", "update the docs", "publish an article", "add to the
  help center", "create a knowledge base article", "edit the getting started
  guide", or mentions Help.Center, help articles, or knowledge base content
  management. Also use when the user says "reply to this customer", "draft a
  reply", "list open conversations", "assign this conversation", "add a note
  to the conversation", or mentions the Help.Center inbox, customer support
  conversations, customer replies, or internal discussion threads.
metadata:
  version: 1.1.0
  author: Microdot Company
---

# Help.Center API

Manage help center articles and customer conversations through the Help.Center API.

The API has two halves:

- **Article Management** — create, read, update, publish, and translate help center articles. Organize by category and upload images.
- **Conversation Management** — read inbox conversations, draft and send replies to customers, post internal discussion notes, and update conversation status or assignee.

The two halves compose well: when answering a customer, search articles first, draft a reply that references the relevant article, and let the user review before sending.

# Help.Center Article Management

Manage help center articles through the Help.Center API. Supports creating new articles, reading and updating existing ones, publishing/unpublishing, organizing by category, managing translations, and uploading images.

## Prerequisites

Before making any API calls, you need two pieces of information from the user:

1. **API Key** - Created in Help.Center dashboard under Settings > General > API
   - The key must have appropriate scopes:
     - `content.read` - Required for searching/reading articles, drafts, categories, and center info
     - `content.write` - Required for creating/updating articles, drafts, categories, and uploading images
     - `content.publish` - Required for publishing/unpublishing articles and translations
     - `content.delete` - Required for deleting articles, categories, or translations
     - `conversation.read` - Required for listing/reading conversations, messages, discussion, and drafts
     - `conversation.write` - Required for updating conversation status/assignee, editing drafts, and posting discussion entries
     - `conversation.reply` - Required for sending replies to customers (POST `/reply` and POST `/draft/send`)
2. **Center ID** - Found on the same page

If the user hasn't provided these, ask for them before proceeding. Store them as environment variables for the session:

```bash
export HC_API_KEY="the_api_key"
export HC_CENTER_ID="the_center_id"
```

## Base URL

`https://api.help.center`

## Authentication

All requests require the API key in the Authorization header:

```
Authorization: Bearer $HC_API_KEY
```

## Core Concepts

### Draft Model

Every article has two representations: a **published version** (Page) and a **draft version** (PageDraft). The draft tracks in-progress edits before they go live.

Drafts have three statuses:

| Status                   | Meaning                                                                           |
| ------------------------ | --------------------------------------------------------------------------------- |
| `unpublished`            | Never been published. Only a draft exists.                                        |
| `published`              | Published and the draft is in sync with the published version.                    |
| `published_with_changes` | Published, but the draft has unpublished edits that differ from the live version. |

Use the `status` query parameter on draft endpoints to filter by these states.

### Expand Parameter

Most GET endpoints return lightweight responses by default (no content body). To include the article HTML and text content, add the `expand` query parameter:

```
?expand[]=content
```

This adds a `content` object to the response:

```json
{
  "content": {
    "html": "<h1>Article Title</h1><p>Body...</p>",
    "text": "Article Title\nBody..."
  }
}
```

Always use `expand[]=content` when you need to read or modify article content.

### Pagination

List endpoints use **cursor-based pagination**:

| Parameter        | Description                                               |
| ---------------- | --------------------------------------------------------- |
| `limit`          | Number of items to return. Min: 1, Max: 100, Default: 50  |
| `starting_after` | Return items after this article ID (forward pagination)   |
| `ending_before`  | Return items before this article ID (backward pagination) |

List responses include a `has_more` boolean. When `true`, use the last item's `id` as `starting_after` to fetch the next page:

```
?limit=50&starting_after=LAST_ARTICLE_ID
```

## Response Shapes

### Article Response

Returned by published article endpoints:

```json
{
  "id": "article_id",
  "object": "article",
  "center_id": "center_id",
  "category_id": "cat_id",
  "slug_id": 42,
  "slug": "article-slug",
  "title": "Article Title",
  "status": "published",
  "created_at": "2024-01-01T00:00:00.000Z",
  "updated_at": "2024-01-02T00:00:00.000Z",
  "published_at": "2024-01-01T12:00:00.000Z",
  "seo": { "metaTitle": "...", "metaDesc": "..." },
  "url": "https://domain.help.center/article/42-article-slug",
  "translations": ["de", "fr"],
  "content": { "html": "...", "text": "..." }
}
```

Note: `content` is only included when `expand[]=content` is used. `translations` is an array of language codes that have published translations.

### Draft Response

Returned by draft endpoints:

```json
{
  "id": "article_id",
  "object": "draft",
  "center_id": "center_id",
  "slug_id": 42,
  "slug": "article-slug",
  "title": "Draft Title",
  "category_id": "cat_id",
  "status": "unpublished",
  "has_unpublished_changes": true,
  "created_at": "2024-01-01T00:00:00.000Z",
  "updated_at": "2024-01-02T00:00:00.000Z",
  "published_at": null,
  "preview_url": "https://domain.help.center/preview/PREVIEW_ID",
  "url": "https://domain.help.center/article/42-article-slug",
  "translations": ["de", "fr"],
  "content": { "html": "...", "text": "..." }
}
```

Note: `url` is only present if the article has been published. `status` is one of `unpublished`, `published`, or `published_with_changes`.

### List Response

Returned by list endpoints:

```json
{
  "object": "list",
  "data": [
    /* array of article or draft objects */
  ],
  "has_more": false,
  "count": 10,
  "url": "/api/v0/centers/CENTER_ID/articles"
}
```

### Count Response

Returned by count endpoints:

```json
{
  "object": "count",
  "total": 42,
  "filters": {
    "category": null,
    "search": null,
    "status": null
  },
  "url": "/api/v0/centers/CENTER_ID/articles/count"
}
```

### Error Response

All errors follow this format:

```json
{
  "error": {
    "type": "invalid_request_error",
    "message": "Human-readable description of the error",
    "code": "specific_error_code"
  }
}
```

Common error codes:
| HTTP Status | Code | Meaning |
|-------------|------|---------|
| 400 | `missing_required_field` | A required field (e.g., `title`) was not provided |
| 400 | `root_not_published` | Cannot publish translation when root article is not published |
| 400 | `category_has_articles` | Cannot delete a category that has articles assigned to it |
| 401 | `invalid_api_key` | API key is invalid or missing |
| 403 | `insufficient_scope` | API key lacks the required scope |
| 403 | `center_blocked` | Center is blocked |
| 404 | `not_found` | Resource not found |
| 404 | `endpoint_not_found` | The requested API endpoint does not exist |
| 429 | `rate_limited` | Too many requests, wait and retry |

## Workflow

### When the user wants to UPDATE an existing article

1. **Search for the article first** to find its ID and current content:

   ```bash
   curl -s -X GET \
     -H "Authorization: Bearer $HC_API_KEY" \
     -H "Content-Type: application/json" \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/articles?search=SEARCH_TERM&expand[]=content"
   ```

2. **Read the full article** using the article ID from search results:

   ```bash
   curl -s -X GET \
     -H "Authorization: Bearer $HC_API_KEY" \
     -H "Content-Type: application/json" \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/ARTICLE_ID?expand[]=content"
   ```

3. **Update only the specific part** the user wants changed. Merge the user's changes into the existing HTML content, preserving everything else. Update via the draft endpoint:

   ```bash
   curl -s -X PATCH \
     -H "Authorization: Bearer $HC_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "title": "Updated Title",
       "html": "<h1>Updated full HTML content with changes merged in</h1>"
     }' \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/ARTICLE_ID/draft"
   ```

   The draft endpoint only accepts `title` and `html`. The API auto-generates the plain text version from the HTML. The `json` field (editor state) is internal and not accepted via the API.

4. **Publish the updated article** (ask the user first if they want to publish or keep as draft):
   ```bash
   curl -s -X POST \
     -H "Authorization: Bearer $HC_API_KEY" \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/ARTICLE_ID/publish"
   ```

### When the user wants to CREATE a new article

1. **List categories** so the article can be assigned properly:

   ```bash
   curl -s -X GET \
     -H "Authorization: Bearer $HC_API_KEY" \
     -H "Content-Type: application/json" \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/categories"
   ```

2. **Write the article content** as clean, well-structured HTML. Follow these content guidelines:
   - Use semantic HTML: `<h1>` for main title, `<h2>` for sections, `<h3>` for subsections
   - Use `<p>` tags for paragraphs
   - Use `<ul>`/`<ol>` for lists
   - Use `<code>` for inline code and `<pre><code>` for code blocks
   - Use `<strong>` for emphasis on key terms
   - Use `<a href="...">` for links
   - Keep the tone clear, helpful, and concise
   - Structure content so users can scan and find what they need quickly

3. **Create the article**:

   ```bash
   curl -s -X POST \
     -H "Authorization: Bearer $HC_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "title": "Article Title",
       "content": {
         "html": "<h1>Title</h1><p>Content here...</p>"
       },
       "category_id": "CATEGORY_ID",
       "metadata": {
         "seo": {
           "metaTitle": "SEO Title (50-60 chars)",
           "metaDesc": "SEO description (150-160 chars)"
         }
       }
     }' \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/articles"
   ```

   **Create article fields:**
   | Field | Required | Description |
   |-------|----------|-------------|
   | `title` | Yes | Article title |
   | `content.html` | No | HTML content body. Defaults to empty if omitted. |
   | `category_id` | No | Category ID from the categories list. Defaults to the first category if omitted. |
   | `metadata.seo` | No | SEO metadata object with `metaTitle` and `metaDesc` fields. |

   The response returns the draft object with `expand: ['content']`.

4. **Publish if requested**:
   ```bash
   curl -s -X POST \
     -H "Authorization: Bearer $HC_API_KEY" \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/ARTICLE_ID/publish"
   ```

## Important Rules

1. **Always search before creating.** If the user says "write an article about X", search for existing articles on that topic first. If one exists, confirm with the user whether they want to update it or create a new one.

2. **Preserve existing content when updating.** Never overwrite an entire article when only a section needs changing. Fetch the current content, modify the relevant part, and send back the full updated HTML.

3. **Always ask before publishing.** Default to creating as draft. Only publish when the user explicitly asks for it.

4. **Handle errors gracefully.** Check the error response body for `error.code` to understand the specific issue. See the Error Response section above for common codes.

5. **Use pagination for large result sets.** Default limit is 50, max is 100. Check `has_more` in the response and use `starting_after` with the last item's `id` to fetch more.

6. **Unpublishing cascades to translations.** When the root (default language) article is unpublished, **all translations are also unpublished** automatically. Warn the user about this before unpublishing a root article that has translations.

## API Endpoint Reference

### Center

#### Get Center Info

```
GET /v0/centers/:centerId
```

Scope: `content.read`

Returns center configuration including language settings.

**Response:**

```json
{
  "id": "center_id",
  "object": "center",
  "name": "My Help Center",
  "domain": "mycompany.help.center",
  "default_language": "en",
  "additional_languages": ["de", "fr", "es"],
  "created_at": "2024-01-01T00:00:00.000Z"
}
```

Use `additional_languages` to check which translation languages are configured before creating translations.

### Articles (Published)

#### List Articles

```
GET /v0/centers/:centerId/articles
```

Scope: `content.read`

Returns only **published** articles.

**Query Parameters:**
| Parameter | Description |
|-----------|-------------|
| `search` | Search in title and text fields |
| `category` | Filter by category ID |
| `limit` | Items per page (1-100, default: 50) |
| `starting_after` | Cursor: article ID to start after |
| `ending_before` | Cursor: article ID to end before |
| `expand[]` | Set to `content` to include HTML/text |

**Example:**

```bash
curl -s -X GET \
  -H "Authorization: Bearer $HC_API_KEY" \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/articles?search=getting+started&category=CAT_ID&limit=20&expand[]=content"
```

#### Get Article

```
GET /v0/centers/:centerId/articles/:articleId
```

Scope: `content.read`

Returns a single published article. Returns 404 if the article exists but is not published.

**Query Parameters:**
| Parameter | Description |
|-----------|-------------|
| `expand[]` | Set to `content` to include HTML/text |

#### Count Articles

```
GET /v0/centers/:centerId/articles/count
```

Scope: `content.read`

Returns the count of published articles.

**Query Parameters:**
| Parameter | Description |
|-----------|-------------|
| `search` | Search in title and text fields |
| `category` | Filter by category ID |

**Example:**

```bash
curl -s -X GET \
  -H "Authorization: Bearer $HC_API_KEY" \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/count?category=CAT_ID"
```

#### Create Article

```
POST /v0/centers/:centerId/articles
```

Scope: `content.write`

Creates a new article as a draft.

**Request Body:**

```json
{
  "title": "Article Title",
  "content": {
    "html": "<h1>Title</h1><p>Content</p>"
  },
  "category_id": "CATEGORY_ID",
  "metadata": {
    "seo": {
      "metaTitle": "SEO Title",
      "metaDesc": "SEO Description"
    }
  }
}
```

| Field          | Required | Description                                  |
| -------------- | -------- | -------------------------------------------- |
| `title`        | Yes      | Article title                                |
| `content.html` | No       | HTML content body                            |
| `category_id`  | No       | Category ID. Defaults to first category.     |
| `metadata.seo` | No       | SEO metadata with `metaTitle` and `metaDesc` |

#### Delete Article

```
DELETE /v0/centers/:centerId/articles/:articleId
```

Scope: `content.delete`

Permanently deletes the article, its draft, and all translations. Returns `204 No Content`.

#### Publish Article

```
POST /v0/centers/:centerId/articles/:articleId/publish
```

Scope: `content.publish`

Publishes the current draft content as the live article. HTML is sanitized before publishing. A slug is auto-generated from the title if one doesn't exist.

#### Unpublish Article

```
POST /v0/centers/:centerId/articles/:articleId/unpublish
```

Scope: `content.publish`

Unpublishes the article, removing it from the live help center. **Warning: This also unpublishes ALL translations of the article.** The draft content is preserved.

#### Duplicate Article

```
POST /v0/centers/:centerId/articles/:articleId/duplicate
```

Scope: `content.write`

Creates a copy of the article with:

- Title prefixed with "Copy of " (including translated titles)
- All metadata copied (category, tags, SEO, subtitle)
- All translation drafts copied (content only, not published state)
- The duplicate is always created as an unpublished draft

Returns the new draft response.

### Drafts

#### List Drafts

```
GET /v0/centers/:centerId/articles/drafts
```

Scope: `content.read`

Returns all article drafts (published, unpublished, or with changes).

**Query Parameters:**
| Parameter | Description |
|-----------|-------------|
| `status` | Filter by draft status: `unpublished`, `published`, or `published_with_changes` |
| `search` | Search in title and text fields |
| `category` | Filter by category ID |
| `limit` | Items per page (1-100, default: 50) |
| `starting_after` | Cursor: article ID to start after |
| `ending_before` | Cursor: article ID to end before |
| `expand[]` | Set to `content` to include HTML/text |

**Example - list articles with unpublished changes:**

```bash
curl -s -X GET \
  -H "Authorization: Bearer $HC_API_KEY" \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/drafts?status=published_with_changes&expand[]=content"
```

#### Count Drafts

```
GET /v0/centers/:centerId/articles/drafts/count
```

Scope: `content.read`

Returns the count of drafts matching the given filters.

**Query Parameters:**
| Parameter | Description |
|-----------|-------------|
| `status` | Filter: `unpublished`, `published`, or `published_with_changes` |
| `search` | Search in title and text fields |
| `category` | Filter by category ID |

**Example:**

```bash
curl -s -X GET \
  -H "Authorization: Bearer $HC_API_KEY" \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/drafts/count?status=unpublished"
```

#### Get Draft

```
GET /v0/centers/:centerId/articles/:articleId/draft
```

Scope: `content.read`

Returns the draft version of a specific article including unpublished changes.

**Query Parameters:**
| Parameter | Description |
|-----------|-------------|
| `expand[]` | Set to `content` to include HTML/text |

**Example:**

```bash
curl -s -X GET \
  -H "Authorization: Bearer $HC_API_KEY" \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/ARTICLE_ID/draft?expand[]=content"
```

#### Update Draft

```
PATCH /v0/centers/:centerId/articles/:articleId/draft
```

Scope: `content.write`

Updates the draft content. Only accepts `title` and `html`. The plain text version is auto-generated from HTML. After updating, the draft status becomes `published_with_changes` if the article was previously published.

**Request Body:**

```json
{
  "title": "Updated Title",
  "html": "<h1>Updated Title</h1><p>New content</p>"
}
```

| Field   | Description        |
| ------- | ------------------ |
| `title` | Draft title        |
| `html`  | Draft HTML content |

#### Update Article Metadata

```
PATCH /v0/centers/:centerId/articles/:articleId/metadata
```

Scope: `content.write`

Updates article metadata without affecting draft content. These changes apply to the published article immediately (no publish step needed).

**Request Body:**

```json
{
  "category": "NEW_CATEGORY_ID",
  "subtitle": "Article subtitle",
  "slug": "custom-url-slug",
  "seo": {
    "metaTitle": "SEO Title (50-60 chars)",
    "metaDesc": "SEO Description (150-160 chars)"
  }
}
```

| Field      | Description                                                                                          |
| ---------- | ---------------------------------------------------------------------------------------------------- |
| `category` | Reassign to a different category ID. Automatically updates category membership.                      |
| `subtitle` | Article subtitle                                                                                     |
| `slug`     | Custom URL slug                                                                                      |
| `seo`      | Full SEO object. Individual fields (`seo.metaTitle`, `seo.metaDesc`) can also be updated separately. |

All fields are optional, but at least one must be provided.

#### Discard Draft Changes

```
POST /v0/centers/:centerId/articles/:articleId/draft/discard
```

Scope: `content.write`

Restores the draft to match the currently published version, discarding all unpublished changes. Returns 404 if the article has never been published (there is no published version to restore from).

**Example:**

```bash
curl -s -X POST \
  -H "Authorization: Bearer $HC_API_KEY" \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/ARTICLE_ID/draft/discard"
```

### Categories

Categories help organize articles. Supports one level of subcategories (parent > child).

#### List Categories

```
GET /v0/centers/:centerId/articles/categories
```

Scope: `content.read`

Returns all categories with their subcategories.

**Category Response Shape:**

```json
{
  "object": "list",
  "data": [
    {
      "id": "cat_abc123",
      "name": "Getting Started",
      "description": "Articles for new users",
      "slug": "getting-started",
      "slug_id": 1,
      "translations": {
        "de": {
          "name": "Erste Schritte",
          "description": "Artikel fuer neue Benutzer"
        }
      },
      "children": [
        {
          "id": "cat_def456",
          "name": "Installation",
          "description": "Setup guides",
          "slug": "installation",
          "slug_id": 2
        }
      ]
    }
  ],
  "url": "/api/v0/centers/CENTER_ID/articles/categories"
}
```

#### Create Category

```
POST /v0/centers/:centerId/articles/categories
```

Scope: `content.write`

**Request Body:**

```json
{
  "name": "Getting Started",
  "description": "Articles for new users",
  "icon": "<svg>...</svg>",
  "parent_id": "PARENT_CATEGORY_ID",
  "translations": {
    "de": {
      "name": "Erste Schritte",
      "description": "Artikel fuer neue Benutzer"
    },
    "fr": {
      "name": "Pour commencer",
      "description": "Articles pour les nouveaux"
    }
  }
}
```

| Field          | Required | Description                                                                                   |
| -------------- | -------- | --------------------------------------------------------------------------------------------- |
| `name`         | Yes      | Category name                                                                                 |
| `description`  | No       | Category description                                                                          |
| `icon`         | No       | Custom SVG icon                                                                               |
| `parent_id`    | No       | Parent category ID to create a subcategory                                                    |
| `translations` | No       | Translated name and description by language code. Languages must be configured on the center. |

Returns `201` with the full categories list.

#### Update Category

```
PATCH /v0/centers/:centerId/articles/categories/:categoryId
```

Scope: `content.write`

**Request Body:**

```json
{
  "name": "Updated Name",
  "description": "Updated description",
  "icon": "<svg>...</svg>",
  "translations": {
    "de": {
      "name": "Aktualisierter Name",
      "description": "Aktualisierte Beschreibung"
    }
  }
}
```

| Field          | Description                                   |
| -------------- | --------------------------------------------- |
| `name`         | Category name                                 |
| `description`  | Category description                          |
| `icon`         | Custom SVG icon                               |
| `translations` | Translated name/description per language code |

At least one field must be provided. Returns the full categories list.

#### Delete Category

```
DELETE /v0/centers/:centerId/articles/categories/:categoryId
```

Scope: `content.delete`

Deletes a category. **Cannot delete a category that has articles assigned to it**, including articles assigned to its subcategories. The error response includes the IDs of articles blocking deletion so you can reassign them first.

Returns the full categories list on success.

### Image Upload

#### Upload Image

```
POST /v0/centers/:centerId/articles/images
POST /v0/centers/:centerId/articles/:articleId/images
```

Scope: `content.write`

Upload an image for use in articles. Both endpoints work the same; the article-specific variant is optional.

**Request:** Multipart form data with field name `image`.

```bash
curl -s -X POST \
  -H "Authorization: Bearer $HC_API_KEY" \
  -F "image=@/path/to/image.jpg" \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/images"
```

**Constraints:**

- Maximum size: 10MB
- Supported formats: JPEG, PNG, GIF, WebP, SVG
- Use `multipart/form-data` with field name `image`

**Response:**

```json
{
  "success": true,
  "data": {
    "url": "https://cdn.help.center/images/...",
    "filename": "image.jpg",
    "size": 1024576
  }
}
```

Use the returned `url` in your article HTML: `<img src="URL" alt="description">`.

### Translations

Centers can have additional languages configured. Use the Get Center Info endpoint to check `additional_languages` for available language codes (e.g., `de`, `fr`, `es`).

#### Important Rules for Translations

1. **The default language article must be published first.** You cannot publish a translation until the root article is published. The API returns error code `root_not_published` if you try.
2. **Translations are per-article.** Each article can have independent translations for each configured language.
3. **Translation URLs use a path prefix.** A German translation URL looks like `https://domain.help.center/de/article/42-slug`, not `?language=de`.
4. **Unpublishing the root cascades.** Unpublishing the default language article also unpublishes all translations.

#### Get Published Translation

```
GET /v0/centers/:centerId/articles/:articleId/translations/:language
```

Scope: `content.read`

Returns the published translation. The root article must be published and the translation must be published. Returns 404 otherwise.

**Query Parameters:**
| Parameter | Description |
|-----------|-------------|
| `expand[]` | Set to `content` to include HTML/text |

#### Get Draft Translation

```
GET /v0/centers/:centerId/articles/:articleId/translations/:language/draft
```

Scope: `content.read`

Returns the draft translation including unpublished changes.

**Query Parameters:**
| Parameter | Description |
|-----------|-------------|
| `expand[]` | Set to `content` to include HTML/text |

#### Update Translation Draft

```
PATCH /v0/centers/:centerId/articles/:articleId/translations/:language/draft
```

Scope: `content.write`

**Request Body:**

```json
{
  "title": "Translated Title",
  "html": "<p>Translated content</p>"
}
```

At least one of `title` or `html` must be provided. Marks the translation as having unpublished changes.

#### Update Translation Metadata

```
PATCH /v0/centers/:centerId/articles/:articleId/translations/:language/metadata
```

Scope: `content.write`

**Request Body:**

```json
{
  "slug": "translated-slug",
  "seo": {
    "metaTitle": "Translated SEO Title",
    "metaDesc": "Translated SEO Description"
  }
}
```

Individual SEO fields can be updated separately (`seo.metaTitle` or `seo.metaDesc`).

#### Publish Translation

```
POST /v0/centers/:centerId/articles/:articleId/translations/:language/publish
```

Scope: `content.publish`

Publishes the translation draft. **Requires the root (default language) article to be published first.** The translation must have a title or content. A slug is auto-generated from the translated title if one doesn't exist.

#### Unpublish Translation

```
POST /v0/centers/:centerId/articles/:articleId/translations/:language/unpublish
```

Scope: `content.publish`

Unpublishes a single translation. The draft is preserved.

#### Delete Translation

```
DELETE /v0/centers/:centerId/articles/:articleId/translations/:language
```

Scope: `content.delete`

Permanently removes both draft and published content for this language. Returns `204 No Content`.

### Translation Workflow

1. **Check available languages** on the center:

   ```bash
   curl -s -X GET \
     -H "Authorization: Bearer $HC_API_KEY" \
     "https://api.help.center/v0/centers/$HC_CENTER_ID"
   ```

   Check the `additional_languages` array in the response.

2. **Set the translation draft content**:

   ```bash
   curl -s -X PATCH \
     -H "Authorization: Bearer $HC_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "title": "Translated Title",
       "html": "<p>Translated content</p>"
     }' \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/ARTICLE_ID/translations/LANGUAGE/draft"
   ```

3. **Verify the root article is published.** The API rejects translation publishing with a `root_not_published` error if the default language article isn't published. If it's not published, **warn the user** and ask if they'd like to publish the root article first.

4. **Publish the translation**:
   ```bash
   curl -s -X POST \
     -H "Authorization: Bearer $HC_API_KEY" \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/ARTICLE_ID/translations/LANGUAGE/publish"
   ```

## Content Writing Guidelines

When writing help center articles:

- **Lead with the outcome.** Start by telling the user what they'll be able to do after reading.
- **Use short paragraphs.** 2-3 sentences max per paragraph.
- **Add step-by-step instructions** with numbered lists for procedures.
- **Include examples** wherever possible to make abstract concepts concrete.
- **Use screenshots or visuals** references where helpful (use the image upload endpoint to host images, then reference the returned URL in your HTML).
- **End with next steps** or related articles when relevant.
- **Write for scanning.** Use descriptive headings so users can jump to what they need.

## API Quick Reference

| Action                      | Method | Endpoint                                                                 | Scope   |
| --------------------------- | ------ | ------------------------------------------------------------------------ | ------- |
| Get center info             | GET    | `/v0/centers/:centerId`                                                  | read    |
| List articles               | GET    | `/v0/centers/:centerId/articles`                                         | read    |
| Get article                 | GET    | `/v0/centers/:centerId/articles/:articleId`                              | read    |
| Count articles              | GET    | `/v0/centers/:centerId/articles/count`                                   | read    |
| Create article              | POST   | `/v0/centers/:centerId/articles`                                         | write   |
| Delete article              | DELETE | `/v0/centers/:centerId/articles/:articleId`                              | delete  |
| Publish article             | POST   | `/v0/centers/:centerId/articles/:articleId/publish`                      | publish |
| Unpublish article           | POST   | `/v0/centers/:centerId/articles/:articleId/unpublish`                    | publish |
| Duplicate article           | POST   | `/v0/centers/:centerId/articles/:articleId/duplicate`                    | write   |
| List drafts                 | GET    | `/v0/centers/:centerId/articles/drafts`                                  | read    |
| Count drafts                | GET    | `/v0/centers/:centerId/articles/drafts/count`                            | read    |
| Get draft                   | GET    | `/v0/centers/:centerId/articles/:articleId/draft`                        | read    |
| Update draft                | PATCH  | `/v0/centers/:centerId/articles/:articleId/draft`                        | write   |
| Update metadata             | PATCH  | `/v0/centers/:centerId/articles/:articleId/metadata`                     | write   |
| Discard draft               | POST   | `/v0/centers/:centerId/articles/:articleId/draft/discard`                | write   |
| List categories             | GET    | `/v0/centers/:centerId/articles/categories`                              | read    |
| Create category             | POST   | `/v0/centers/:centerId/articles/categories`                              | write   |
| Update category             | PATCH  | `/v0/centers/:centerId/articles/categories/:categoryId`                  | write   |
| Delete category             | DELETE | `/v0/centers/:centerId/articles/categories/:categoryId`                  | delete  |
| Upload image                | POST   | `/v0/centers/:centerId/articles/images`                                  | write   |
| Upload image (article)      | POST   | `/v0/centers/:centerId/articles/:articleId/images`                       | write   |
| Get translation             | GET    | `/v0/centers/:centerId/articles/:articleId/translations/:lang`           | read    |
| Get translation draft       | GET    | `/v0/centers/:centerId/articles/:articleId/translations/:lang/draft`     | read    |
| Update translation draft    | PATCH  | `/v0/centers/:centerId/articles/:articleId/translations/:lang/draft`     | write   |
| Update translation metadata | PATCH  | `/v0/centers/:centerId/articles/:articleId/translations/:lang/metadata`  | write   |
| Publish translation         | POST   | `/v0/centers/:centerId/articles/:articleId/translations/:lang/publish`   | publish |
| Unpublish translation       | POST   | `/v0/centers/:centerId/articles/:articleId/translations/:lang/unpublish` | publish |
| Delete translation          | DELETE | `/v0/centers/:centerId/articles/:articleId/translations/:lang`           | delete  |

# Help.Center Conversation Management

Manage customer support conversations through the Help.Center API. Supports listing and reading conversations, drafting replies, sending replies to customers, posting internal discussion entries, and updating conversation status or assignee.

## Conversation Concepts

### What is a Conversation?

A conversation is a thread of messages between a customer (the **contact**) and one or more agents at the help center. Conversations are identified by both a Mongo `id` and a per-center `number`. The API uses `id`.

### Conversation Channels

A conversation's `from` field indicates the channel it originated on:

| `from`     | Channel                                              |
| ---------- | ---------------------------------------------------- |
| `email`    | Customer sent an email to the support address.       |
| `widget`   | Customer chatted via the embedded help.center widget. |
| `website`  | Customer interacted via the website (e.g. AI search). |

**Read endpoints work for all channels.** You can list, filter, and read messages and discussion on widget and website conversations exactly as you would on email conversations.

**Reply is delivered by email, regardless of channel.** The reply API (`POST /reply` and `POST /draft/send`) sends an email to the customer. For email-originating conversations this is the natural thread continuation. For widget and website conversations, the API looks for an email address attached to the conversation (on the linked `contact` or in conversation `metaData`) and replies there.

If no email address can be resolved from any source — envelope history, draft, contact, or metaData — the reply returns `400 missing_recipient`. Some widget/website conversations don't have an associated email (anonymous chat) and so cannot be replied to via this API.

Each conversation has:

- A list of **messages** — the visible thread, including inbound (`role: user`) and outbound (`role: agent`) emails, plus AI replies (`role: ai`) and system entries.
- A **discussion** — a separate, internal-only thread used by the team to coordinate on the conversation. Customers never see the discussion.
- An optional **draft** — a single in-progress reply (content + envelope) stored on the conversation. Editing the draft never sends anything to the customer.
- **Status** (`open`, `pending`, `closed`), **assignee** (a user on the center), and a **contact** (the customer).

### Status Values

| Status    | Meaning                                                                |
| --------- | ---------------------------------------------------------------------- |
| `open`    | New or active conversation that needs attention.                       |
| `pending` | In progress — being worked on but not closed.                          |
| `closed`  | Resolved.                                                              |

Only these three status values are accepted on PATCH. Any other value returns `400 invalid_status`.

### Reply vs Discussion

- A **reply** (POST `/conversations/:id/reply`) sends an email to the customer. This is an external side effect — always confirm with the user before calling it.
- A **discussion entry** (POST `/conversations/:id/discussion`) is internal-only. Nothing leaves the system. Use it for notes, context, and team coordination.

These are intentionally separate scopes: `conversation.reply` for sending; `conversation.write` for everything internal (drafts, discussion, status, assignee).

### Mentions

Discussion entries can mention specific team members. Mentioned users receive an email and Slack notification.

There are two ways to mention users when posting a discussion entry:

1. **Shorthand placeholders** (recommended): write `{{@USER_ID}}` in the `content` HTML. The server resolves each placeholder to the proper mention markup and uses the user's display name.

   ```json
   {
     "content": "<p>Pinging {{@65c2d3e4f5060708091011bb}} — can you take a look at this checkout bug?</p>"
   }
   ```

2. **Explicit list**: pass a `mentions` array of user ids alongside the content. The server attaches notifications to all of them, even if the user ids don't appear inline in the HTML.

   ```json
   {
     "content": "<p>FYI to the team on this thread.</p>",
     "mentions": ["65c2d3e4f5060708091011bb", "65c2d3e4f5060708091011cc"]
   }
   ```

Both approaches can be combined — the server dedupes by user id. To get valid user ids for mentions, call `GET /v0/centers/:centerId/users`.

A discussion entry's response includes a `mentions` array of `{id, name, email}` for every user that was notified, so consumers don't have to parse HTML.

**Filtering conversations by mention.** `GET /conversations?mentioned=USER_ID` returns conversations where that user has been mentioned in discussion. Pass `mentioned=me` to filter by the API key's owner. Repeat the param for OR semantics across multiple users.

### Draft Model

Drafts on conversations work differently from article drafts:

- Each conversation has **at most one** draft (a single in-progress reply), stored directly on the conversation.
- The draft holds `content` (HTML reply body) and `envelope` (`to`, `cc`, `bcc`).
- Drafts do not have a publish step — to send, call POST `/conversations/:id/draft/send`. This sends the draft as a reply and clears it atomically.
- Editing the draft is safe and has no external effect.

### Messages, Discussion, and Truncation

Conversation message bodies can be long (legacy email threads, attachments, quoted history). Discussion entries are typically shorter — internal notes from teammates. The truncation strategy reflects this:

- **`GET /conversations/:id` returns the full conversation**: metadata, draft, **all messages**, and **all discussion entries**, all sorted descending by `created_at` (newest first). Each message and discussion entry has its `html` and `text` truncated at **2000 characters**. When truncated, the entry includes `truncated: true` and a `content_length` object showing the real lengths.
- **`GET /conversations` (list)** inlines the **5 most recent messages** per conversation (newest first, truncated) and the **full discussion thread** (truncated). `messages_has_more: true` indicates older messages exist — fetch the full conversation via `GET /conversations/:id` for those. Discussion is small enough that it is included in full on list responses too.
- **To read the full body of a truncated message or discussion entry, fetch it by id**: `GET /conversations/:id/messages/:messageId`. The single-entry response is never truncated. The same endpoint returns both messages and discussion entries (they live in the same model).

**When to fetch by id:** anytime an entry has `truncated: true` and you need to read the parts that were cut off. Always do this before composing a reply — replies written off a truncated quote tend to miss what the customer actually said.

### Message IDs

Every message has two identifiers:

- `id` — the Mongo `_id` of the message. Use this when you got the id from a list response.
- `message_id` — the RFC 5322 email Message-ID (a string like `<abc@email.amazonses.com>`). Present on email messages only.

The path parameter `:messageId` accepts **either**. The server tries to parse it as a Mongo ObjectId first; if that fails it falls back to looking up by `message_id`. URL-encode the email Message-ID (it contains `<`, `>`, and `@`).

### Pagination

Conversation list endpoints use cursor pagination identical to articles:

| Parameter        | Description                                                            |
| ---------------- | ---------------------------------------------------------------------- |
| `limit`          | Items per page. Conversations: 1–100, default 50. Messages and discussion: 1–50, default 20. |
| `starting_after` | Cursor: id to start after (forward pagination)                         |
| `ending_before`  | Cursor: id to end before (backward pagination)                         |

Use `has_more` and the last item's `id` to page forward.

## Conversation Response Shapes

### Conversation Object

```json
{
  "object": "conversation",
  "id": "65f1a2b3c4d5e6f708091011",
  "number": 1247,
  "center_id": "65a2b3c4d5e6f70809101112",
  "status": "open",
  "from": "email",
  "contact": {
    "object": "contact",
    "id": "65b1c2d3e4f50607080910aa",
    "name": "Jane Doe",
    "email": "jane@example.com",
    "avatar": "https://www.gravatar.com/avatar/..."
  },
  "assignee": {
    "object": "user",
    "id": "65c2d3e4f5060708091011bb",
    "name": "Agent Smith",
    "email": "smith@company.com"
  },
  "unread": true,
  "message_count": 12,
  "discussion_count": 3,
  "last_message_at": "2026-05-13T14:22:00.000Z",
  "draft": {
    "content": "<p>Thanks for reaching out...</p>",
    "envelope": {
      "to": { "address": "jane@example.com" },
      "cc": [],
      "bcc": []
    }
  },
  "messages": [
    /* On GET /conversations/:id: all messages, newest first, each truncated at 2000 chars.
       On GET /conversations: only the 5 most recent (newest first, truncated). */
  ],
  "messages_has_more": false,
  "discussion": [
    /* Full discussion thread, newest first, each entry truncated at 2000 chars.
       Present on both GET /conversations/:id and GET /conversations. */
  ],
  "created_at": "2026-05-10T09:14:00.000Z",
  "updated_at": "2026-05-13T14:22:00.000Z"
}
```

Notes:

- `assignee` is `null` when the conversation is unassigned.
- `draft` is `null` when no draft exists.
- `from` indicates the originating channel and is one of `email`, `widget`, or `website`. Read endpoints work for all three; reply is email-only delivery (see Conversation Channels above).
- **On `GET /conversations/:id` (single conversation):** `messages` contains the full thread, `discussion` contains the full discussion. Both are sorted newest-first and each entry is truncated at 2000 chars. `messages_has_more` is always `false` on this endpoint.
- **On `GET /conversations` (list):** each conversation includes the latest 5 `messages` (newest first, truncated) and the **full** `discussion` thread (truncated). `messages_has_more` is `true` when older messages exist beyond the 5 inlined — fetch the full conversation via `GET /conversations/:id` to read them.
- `message_count` and `discussion_count` are the full thread counts, regardless of how many entries are inlined.

### Message Object

Returned by list endpoints (truncated when content exceeds 2000 characters):

```json
{
  "object": "message",
  "id": "65f1a2b3c4d5e6f708091020",
  "message_id": "<abc123@email.amazonses.com>",
  "role": "user",
  "html": "<p>Hi, I'm having trouble with...</p>",
  "text": "Hi, I'm having trouble with...",
  "truncated": false,
  "envelope": {
    "from": { "name": "Jane Doe", "address": "jane@example.com" },
    "to": [{ "address": "support@company.help.center" }],
    "cc": [],
    "bcc": []
  },
  "subject": "Issue with checkout",
  "attachments": [
    { "filename": "screenshot.png", "url": "https://cdn.help.center/..." }
  ],
  "bounced": false,
  "created_at": "2026-05-10T09:14:00.000Z"
}
```

When truncated:

```json
{
  "object": "message",
  "id": "...",
  "html": "<p>...first 2000 chars...</p>",
  "text": "...first 2000 chars...",
  "truncated": true,
  "content_length": { "html": 18432, "text": 4221 },
  "...": "..."
}
```

`role` is one of:

| Role     | Meaning                                                                  |
| -------- | ------------------------------------------------------------------------ |
| `user`   | Inbound message from the customer (email received).                      |
| `agent`  | Outbound message from a human agent (email sent via reply).              |
| `ai`     | AI-generated reply (common on `widget` and `website` conversations).     |
| `system` | System-generated message.                                                |

**Discussion entries** use the same shape with `role: "note"` (for human-posted entries) or `role: "system"`. They also include a `mentions` array listing every user that was notified:

```json
{
  "object": "message",
  "id": "...",
  "role": "note",
  "html": "<p>Pinging <span class=\"mention\" data-id=\"USER_ID\">@Alice</span> on this</p>",
  "text": "Pinging @Alice on this",
  "mentions": [
    { "id": "65c2d3e4f5060708091011bb", "name": "Alice Wong", "email": "alice@company.com" }
  ],
  "envelope": { "from": {...}, "to": [...], "cc": [], "bcc": [] },
  "created_at": "..."
}
```

The `mentions` array is only present on discussion entries, not on customer-facing messages.

### Draft Response

```json
{
  "object": "draft",
  "conversation_id": "65f1a2b3c4d5e6f708091011",
  "content": "<p>Thanks for reaching out...</p>",
  "envelope": {
    "to": { "address": "jane@example.com" },
    "cc": [{ "address": "manager@example.com" }],
    "bcc": []
  },
  "updated_at": "2026-05-13T14:25:00.000Z"
}
```

When no draft exists, GET `/draft` returns `null` for `content` and the resolved envelope defaults (see "Recipient and Subject Resolution" below).

### List Response

Same shape as articles:

```json
{
  "object": "list",
  "data": [
    /* array of conversation, message, or discussion objects */
  ],
  "has_more": false,
  "count": 10,
  "url": "/api/v0/centers/CENTER_ID/conversations"
}
```

### Count Response

```json
{
  "object": "count",
  "open": 14,
  "pending": 3,
  "closed": 421,
  "total": 438,
  "url": "/api/v0/centers/CENTER_ID/conversations/count"
}
```

### Error Response

Errors use the same shape as the article API. Conversation-specific error codes:

| HTTP Status | Code                       | Meaning                                                                       |
| ----------- | -------------------------- | ----------------------------------------------------------------------------- |
| 400         | `invalid_status`           | Status must be one of `open`, `pending`, `closed`.                            |
| 400         | `invalid_assignee`         | Assignee user id is not a member of this center.                              |
| 400         | `invalid_mention`          | One or more mention user ids do not belong to this center.                    |
| 400         | `missing_recipient`        | A reply was attempted but no `to` address could be resolved.                  |
| 400         | `missing_content`          | Reply or discussion entry had no `content`.                                   |
| 400         | `empty_draft`              | POST `/draft/send` was called but the stored draft is empty.                  |
| 500         | `channel_not_configured`   | The center has no support channel configured to send from.                    |
| 502         | `email_send_failed`        | The upstream email provider rejected the send.                                |

## Recipient and Subject Resolution

When sending a reply (POST `/reply` or POST `/draft/send`), the server resolves `to`, `cc`, `bcc`, and `subject` automatically if the request omits them. The resolution mirrors the dashboard reply editor's behavior exactly and works for all channel types — for widget and website conversations, the fallback to `contact.email` and `metaData.email` is what makes email reply possible.

**Precedence (highest wins):**

1. Field explicitly supplied in the request body.
2. Field present on the conversation's stored draft.
3. Derived from the conversation history (see below).
4. Fallback default.

### `to` Resolution

The first match wins:

1. Override from request body.
2. `draft.envelope.to`.
3. The `envelope.from` of the most recent **inbound** (`role: user`) message.
4. The first recipient (`envelope.to[0]`) of the most recent **outbound** (`role: agent`) message (used when there are no inbound messages — e.g., conversations started outbound). Recent is used so the reply follows where the thread currently goes rather than where it started.
5. `{ name: contact.name, address: contact.email }`.
6. `{ address: metadata.email }` if present.

If none of these yield an address, the request fails with `400 missing_recipient`.

### `cc` Resolution

1. Override from request body.
2. `draft.envelope.cc`.
3. Derived from the most recent inbound message: take its `envelope.to` plus `envelope.cc`, then **exclude** the resolved primary `to` address and the center's own send addresses (`defaultAddress`, `customAddress`). Deduplicate.
4. Empty array.

### `bcc` Resolution

1. Override from request body.
2. `draft.envelope.bcc`.
3. Empty array. **There is no default BCC.** The server never auto-populates BCC.

### Subject Resolution

Computed at send time. The conversation's stored draft does **not** carry a subject; supply one explicitly if you want to override.

1. Override from request body.
2. If there is at least one inbound message: `` `Re : ${lastInboundSubject}` ``.
3. Else if there is at least one outbound message: `lastOutboundSubject` (no `Re :` prefix).
4. Else: `` `Question from ${to.address}` ``.

**Quirk to be aware of:** the `Re :` prefix has a space before the colon, and the API does **not** deduplicate existing `Re:` prefixes. A long thread can produce `Re : Re: Re: ...` subjects. This matches the dashboard exactly. Pass an explicit `subject` if you need different behavior.

### `from` (Sender)

The sender address is always resolved server-side from the center's configured support channel. It cannot be overridden via the API. Centers must have at least one verified channel configured, or the API returns `500 channel_not_configured`.

### Previewing a Reply (`dry_run`)

Both `POST /reply` and `POST /draft/send` accept a `dry_run` flag. When set, the server runs the full resolver (using any overrides you pass and any stored draft envelope), returns the resolved envelope, subject, and references — and does **not** send an email or modify the conversation. Use it to verify recipients before committing to a send.

`dry_run` may be passed either as a query string (`?dry_run=true` or `?dry_run=1`) or in the JSON body (`{"dry_run": true}`).

In dry-run mode:
- `content` is **not required** on `/reply`. If omitted, the preview's `content` field is `null`.
- An empty stored draft does **not** raise `400 empty_draft` on `/draft/send` — the preview returns regardless.
- The stored draft is **not** cleared by `/draft/send` dry runs.
- Resolver errors (e.g. `400 missing_recipient`, `500 channel_not_configured`) still surface the same way.

**Response shape (HTTP 200):**

```json
{
  "object": "reply_preview",
  "envelope": {
    "from":  { "name": "Acme Support", "address": "support@acme.com" },
    "to":    [{ "name": "Jane",        "address": "jane@example.com" }],
    "cc":    [{ "address": "manager@example.com" }],
    "bcc":   []
  },
  "subject": "Re : Issue with checkout",
  "references": ["<msg1@email.amazonses.com>", "<msg2@email.amazonses.com>"],
  "content": "<p>Hi Jane...</p>"
}
```

The `object` field distinguishes a preview from a real send (`"reply"`).

**When to use it:**
- Before sending to confirm the resolved recipients match what the user expects, especially on widget/website conversations where the address comes from `contact` or `metaData`.
- To debug `missing_recipient` errors — the preview will return the same error without side effects.
- As a CI safety check in integrations that compose replies programmatically.

## Workflow

### When the user wants to LIST or TRIAGE conversations

1. **List open conversations**:

   ```bash
   curl -s -X GET \
     -H "Authorization: Bearer $HC_API_KEY" \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations?status=open&limit=20"
   ```

2. **Filter further** by assignee, contact, search term, or date range. See "List Conversations" below for the full parameter list.

3. **Read a specific conversation.** The single-conversation GET returns metadata, the draft, the full messages thread, and the full discussion thread (all newest-first, each entry truncated at 2000 chars):

   ```bash
   curl -s -X GET \
     -H "Authorization: Bearer $HC_API_KEY" \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations/CONVERSATION_ID"
   ```

   This is a single round trip — no further calls are needed for most triage and reply flows.

4. **Fetch the full body of a truncated message** before composing a reply. If the latest inbound message has `truncated: true`, fetch it by id:

   ```bash
   curl -s -X GET \
     -H "Authorization: Bearer $HC_API_KEY" \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations/CONVERSATION_ID/messages/MESSAGE_ID"
   ```

### When the user wants to REPLY to a customer

1. **Read the conversation thread first.** Call `GET /conversations/:id` to get the conversation, the draft, the full messages thread, and the full discussion. If the latest inbound message has `truncated: true`, fetch it by id for the full body before composing. Don't write a reply off a truncated quote.

2. **Search articles** if the question may be answerable from an existing help article. Reference the article inline in the reply.

3. **Save a draft first.** Write the reply as a draft so the user can review before sending:

   ```bash
   curl -s -X PATCH \
     -H "Authorization: Bearer $HC_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "content": "<p>Hi Jane,</p><p>Thanks for reaching out...</p>"
     }' \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations/CONVERSATION_ID/draft"
   ```

   The draft is stored on the conversation. Editing the draft has no external effect.

4. **Preview the resolved recipients with `dry_run`.** Confirm the server will send to the address the user expects — this is especially important for widget/website conversations where the recipient comes from `contact` or `metaData` rather than email envelope history:

   ```bash
   curl -s -X POST \
     -H "Authorization: Bearer $HC_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"dry_run": true}' \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations/CONVERSATION_ID/draft/send"
   ```

   The response includes the resolved `envelope.to`, `cc`, `bcc`, and `subject`. Nothing is sent and the draft is not cleared.

5. **Confirm with the user before sending.** Sending a reply emails the customer — this is an external side effect that cannot be undone.

6. **Send the draft** (server resolves `to`, `cc`, `bcc`, and `subject` from conversation history and the draft envelope):

   ```bash
   curl -s -X POST \
     -H "Authorization: Bearer $HC_API_KEY" \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations/CONVERSATION_ID/draft/send"
   ```

   Or, send a reply directly without staging a draft (only when the user has already approved exactly what will be sent):

   ```bash
   curl -s -X POST \
     -H "Authorization: Bearer $HC_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "content": "<p>Hi Jane,</p><p>Thanks for reaching out...</p>"
     }' \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations/CONVERSATION_ID/reply"
   ```

### When the user wants to add an INTERNAL NOTE

Use the discussion endpoint. Nothing is sent to the customer.

```bash
curl -s -X POST \
  -H "Authorization: Bearer $HC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "<p>This looks like the same bug from last week.</p>"
  }' \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations/CONVERSATION_ID/discussion"
```

**To mention a specific teammate** and notify them via email and Slack, first list the center's users to get their ids:

```bash
curl -s -X GET \
  -H "Authorization: Bearer $HC_API_KEY" \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/users"
```

Then post the note using `{{@USER_ID}}` placeholders in the content (the server resolves them and triggers notifications):

```bash
curl -s -X POST \
  -H "Authorization: Bearer $HC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "<p>Hey {{@65c2d3e4f5060708091011bb}}, this looks like the issue from last week. Can you take a look?</p>"
  }' \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations/CONVERSATION_ID/discussion"
```

### When the user wants to FIND conversations they were mentioned in

```bash
curl -s -X GET \
  -H "Authorization: Bearer $HC_API_KEY" \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations?mentioned=me&status=open"
```

Pass an explicit user id (`?mentioned=USER_ID`) for someone other than the key owner.

### When the user wants to ASSIGN or CHANGE STATUS

```bash
curl -s -X PATCH \
  -H "Authorization: Bearer $HC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "pending",
    "assignee": "USER_ID"
  }' \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations/CONVERSATION_ID"
```

Send `"assignee": null` to unassign.

## Important Rules for Conversations

1. **Always confirm before sending a reply.** POST `/reply` and POST `/draft/send` email the customer immediately. This is unlike publishing an article — there is no draft state on the customer's side and no undo. Default to staging the reply as a draft and asking the user to confirm.

2. **Read before replying.** Fetch the conversation before composing — `GET /conversations/:id` returns the full messages and discussion threads in one call. If the most recent inbound message has `truncated: true`, fetch it by id for the full body. Replies written off a truncated quote tend to miss what the customer actually said.

   For widget/website conversations, also check that an email address is available on the `contact` or in `metaData` before attempting to reply — anonymous chats can't be replied to via email and will return `400 missing_recipient`.

3. **Discussion is internal-only.** Use POST `/discussion` for notes, never POST `/reply`. Be explicit with the user about which one you are about to call.

4. **Drafts are safe to overwrite.** Each conversation has at most one draft. PATCH replaces fields you supply and leaves others alone. To clear the draft entirely, use POST `/draft/discard`.

5. **Don't change status without being asked.** Changing a conversation to `closed` or `pending` is a deliberate action — only do it when the user explicitly requests it.

6. **Subject quirks.** The auto-resolved subject uses `Re : ` (with a space) and does not strip existing `Re:` prefixes. If the user cares about clean subjects, pass `subject` explicitly in the request body.

7. **Use pagination.** Conversation lists default to 50 items per page; messages and discussion default to 20. Check `has_more` and use `starting_after` with the last item's `id` for the next page.

8. **Compose with articles.** When answering a customer, search the article API first. If a relevant article exists, reference it in the reply (link to its public URL) instead of restating its content.

## API Endpoint Reference (Conversations)

### List Conversations

```
GET /v0/centers/:centerId/conversations
```

Scope: `conversation.read`

**Query Parameters:**

| Parameter        | Description                                                                                          |
| ---------------- | ---------------------------------------------------------------------------------------------------- |
| `status`         | Filter by status: `open`, `pending`, or `closed`. Repeat for multiple.                               |
| `assignee`       | Filter by assignee user id. Pass `unassigned` to find unassigned.                                    |
| `contact`        | Filter by contact id.                                                                                |
| `mentioned`      | Filter to conversations where this user was mentioned in discussion. Pass `me` for the key's owner, or a user id. Repeat for OR semantics. |
| `search`         | Full-text search across conversation messages and metadata.                                          |
| `unread`         | `true` to return only unread conversations.                                                          |
| `created_after`  | ISO 8601 timestamp — return conversations created on or after this time.                             |
| `created_before` | ISO 8601 timestamp — return conversations created on or before this time.                            |
| `limit`          | Items per page (1–100, default: 50).                                                                 |
| `starting_after` | Cursor: conversation id to start after.                                                              |
| `ending_before`  | Cursor: conversation id to end before.                                                               |

**Example:**

```bash
curl -s -X GET \
  -H "Authorization: Bearer $HC_API_KEY" \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations?status=open&unread=true&limit=20"
```

### Count Conversations

```
GET /v0/centers/:centerId/conversations/count
```

Scope: `conversation.read`

Returns counts per status, honoring the same filters as the list endpoint (except cursor params).

**Query Parameters:** Same as List Conversations, minus pagination.

**Response:**

```json
{
  "object": "count",
  "open": 14,
  "pending": 3,
  "closed": 421,
  "total": 438,
  "url": "/api/v0/centers/CENTER_ID/conversations/count"
}
```

### Get Conversation

```
GET /v0/centers/:centerId/conversations/:conversationId
```

Scope: `conversation.read`

Returns the full conversation object: metadata, `contact`, `assignee`, `draft`, the full `messages` thread, and the full `discussion` thread. Both threads are sorted newest-first and each entry is truncated at 2000 characters.

Reading a conversation via the API does **not** mark it as read.

### Update Conversation

```
PATCH /v0/centers/:centerId/conversations/:conversationId
```

Scope: `conversation.write`

**Request Body** (at least one field required):

```json
{
  "status": "pending",
  "assignee": "USER_ID"
}
```

| Field      | Description                                                              |
| ---------- | ------------------------------------------------------------------------ |
| `status`   | New status. One of `open`, `pending`, `closed`.                          |
| `assignee` | User id to assign. Pass `null` to unassign. User must belong to center.  |

Returns the updated conversation.

### Get Message

```
GET /v0/centers/:centerId/conversations/:conversationId/messages/:messageId
```

Scope: `conversation.read`

Returns a single message or discussion entry with its full untruncated `html` and `text`. Both messages and discussion entries live in the same model and are looked up by the same endpoint. The `:messageId` parameter accepts either the Mongo `id` or the email `message_id` (URL-encoded).

**Example — fetch by Mongo id:**

```bash
curl -s -X GET \
  -H "Authorization: Bearer $HC_API_KEY" \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations/CONV_ID/messages/65f1a2b3c4d5e6f708091020"
```

**Example — fetch by email Message-ID:**

```bash
curl -s -X GET \
  -H "Authorization: Bearer $HC_API_KEY" \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/conversations/CONV_ID/messages/%3Cabc123%40email.amazonses.com%3E"
```

### Send Reply

```
POST /v0/centers/:centerId/conversations/:conversationId/reply
```

Scope: `conversation.reply`

Sends an email reply to the customer and appends the sent message to the conversation. This has external side effects — always confirm with the user first.

**Request Body:**

```json
{
  "content": "<p>Hi Jane,</p><p>Thanks for reaching out...</p>",
  "to": [{ "address": "jane@example.com" }],
  "cc": [{ "address": "manager@example.com" }],
  "bcc": [],
  "subject": "Re : Issue with checkout"
}
```

| Field     | Required | Description                                                                       |
| --------- | -------- | --------------------------------------------------------------------------------- |
| `content` | Yes (unless `dry_run`) | HTML reply body.                                                       |
| `to`      | No       | Recipient(s). If omitted, resolved from conversation history (see resolution rules above). |
| `cc`      | No       | CC recipient(s). If omitted, resolved from conversation history.                  |
| `bcc`     | No       | BCC recipient(s). If omitted, empty.                                              |
| `subject` | No       | Email subject. If omitted, derived from the last message (`Re : <subject>`).      |
| `dry_run` | No       | If `true` (also accepted as `?dry_run=true` query param), the server returns the resolved envelope and subject as `{ "object": "reply_preview", ... }` and does **not** send. See "Previewing a Reply" above. |

Returns the appended message object and the updated conversation. When `dry_run` is set, returns a `reply_preview` instead.

### Get Draft

```
GET /v0/centers/:centerId/conversations/:conversationId/draft
```

Scope: `conversation.read`

Returns the current draft, or `null` content with resolved envelope defaults if no draft exists.

### Update Draft

```
PATCH /v0/centers/:centerId/conversations/:conversationId/draft
```

Scope: `conversation.write`

Upserts the draft. Only supplied fields are updated; omitted fields are left alone.

**Request Body:**

```json
{
  "content": "<p>Hi Jane...</p>",
  "envelope": {
    "to": { "address": "jane@example.com" },
    "cc": [{ "address": "manager@example.com" }],
    "bcc": []
  }
}
```

| Field             | Description                                                                  |
| ----------------- | ---------------------------------------------------------------------------- |
| `content`         | HTML draft body.                                                             |
| `envelope.to`     | Primary recipient.                                                           |
| `envelope.cc`     | CC recipients (pass `[]` to clear).                                          |
| `envelope.bcc`    | BCC recipients (pass `[]` to clear).                                         |

### Discard Draft

```
POST /v0/centers/:centerId/conversations/:conversationId/draft/discard
```

Scope: `conversation.write`

Clears the conversation's stored draft entirely.

### Send Draft

```
POST /v0/centers/:centerId/conversations/:conversationId/draft/send
```

Scope: `conversation.reply`

Sends the stored draft as a reply, then clears the draft. The server resolves any missing envelope fields and subject using the rules in "Recipient and Subject Resolution". Returns the appended message object and the updated conversation.

Returns `400 empty_draft` if no draft content is set.

**Request Body (optional):**

| Field     | Description                                                                                  |
| --------- | -------------------------------------------------------------------------------------------- |
| `dry_run` | If `true` (also accepted as `?dry_run=true` query param), returns a `reply_preview` with the resolved envelope and subject, does **not** send, and does **not** clear the draft. An empty draft does not raise `empty_draft` in dry-run mode. See "Previewing a Reply" above. |

### Post Discussion Entry

```
POST /v0/centers/:centerId/conversations/:conversationId/discussion
```

Scope: `conversation.write`

Appends an internal entry to the discussion. Not visible to the customer. Mentioned users are notified via email and Slack.

**Request Body:**

```json
{
  "content": "<p>Hey {{@65c2d3e4f5060708091011bb}}, can you take a look?</p>",
  "mentions": ["65c2d3e4f5060708091011cc"]
}
```

| Field      | Required | Description                                                                                                                          |
| ---------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `content`  | Yes      | HTML discussion body. May include `{{@USER_ID}}` placeholders — the server resolves each to the proper mention markup.               |
| `mentions` | No       | Array of additional user ids to mention. Combined with users matched from `{{@USER_ID}}` placeholders; deduplicated.                 |

Returns the appended discussion entry, including a resolved `mentions` array of `{id, name, email}` for every user notified.

Returns `400 invalid_mention` if any supplied user id (in `{{@USER_ID}}` or `mentions`) does not belong to this center.

### List Center Users

```
GET /v0/centers/:centerId/users
```

Scope: `conversation.read`

Returns the team members on this center. Use this to discover user ids for mentions, assignees, and the `mentioned` filter.

**Response:**

```json
{
  "object": "list",
  "data": [
    {
      "object": "user",
      "id": "65c2d3e4f5060708091011bb",
      "name": "Alice Wong",
      "email": "alice@company.com",
      "avatar": "https://www.gravatar.com/avatar/..."
    }
  ],
  "url": "/api/v0/centers/CENTER_ID/users"
}
```

## Conversation Quick Reference

| Action                  | Method | Endpoint                                                                       | Scope |
| ----------------------- | ------ | ------------------------------------------------------------------------------ | ----- |
| List conversations      | GET    | `/v0/centers/:centerId/conversations`                                          | read  |
| Count conversations     | GET    | `/v0/centers/:centerId/conversations/count`                                    | read  |
| Get conversation        | GET    | `/v0/centers/:centerId/conversations/:conversationId`                          | read  |
| Update conversation     | PATCH  | `/v0/centers/:centerId/conversations/:conversationId`                          | write |
| Get message             | GET    | `/v0/centers/:centerId/conversations/:conversationId/messages/:messageId`      | read  |
| Send reply              | POST   | `/v0/centers/:centerId/conversations/:conversationId/reply`                    | reply |
| Get draft               | GET    | `/v0/centers/:centerId/conversations/:conversationId/draft`                    | read  |
| Update draft            | PATCH  | `/v0/centers/:centerId/conversations/:conversationId/draft`                    | write |
| Discard draft           | POST   | `/v0/centers/:centerId/conversations/:conversationId/draft/discard`            | write |
| Send draft              | POST   | `/v0/centers/:centerId/conversations/:conversationId/draft/send`               | reply |
| Post discussion entry   | POST   | `/v0/centers/:centerId/conversations/:conversationId/discussion`               | write |
| List center users       | GET    | `/v0/centers/:centerId/users`                                                  | read  |
