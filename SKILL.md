---
name: helpcenter
description: >
  When the user wants to create, update, read, or manage help center articles
  via the Help.Center API. Use when the user says "write a help article",
  "update the docs", "publish an article", "add to the help center", "create a
  knowledge base article", "edit the getting started guide", or mentions
  Help.Center, help articles, or knowledge base content management. Also use
  when the user wants to search existing articles, manage drafts, change
  categories, or publish/unpublish content.
metadata:
  version: 1.0.0
  author: Microdot Company
---

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
