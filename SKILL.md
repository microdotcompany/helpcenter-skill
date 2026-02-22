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

Manage help center articles through the Help.Center API. Supports creating new articles, reading and updating existing ones, publishing/unpublishing, and organizing by category.

## Prerequisites

Before making any API calls, you need two pieces of information from the user:

1. **API Key** - Found in Help.Center dashboard under Settings > General > API
2. **Center ID** - Found in the same location

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
       "category_id": "category-slug",
       "status": "draft"
     }' \
     "https://api.help.center/v0/centers/$HC_CENTER_ID/articles"
   ```

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

4. **Handle errors gracefully.** Check HTTP status codes. Common issues:
   - 401: API key is invalid or missing
   - 404: Article or center not found
   - 400: Missing required fields (title is always required)
   - 429: Rate limited (wait and retry)

5. **Use pagination for large result sets.** The API returns max 100 articles per request. Use `starting_after` with the last article's ID to fetch more.

## API Quick Reference

| Action | Method | Endpoint |
|--------|--------|----------|
| List articles | GET | `/v0/centers/:centerId/articles` |
| Search articles | GET | `/v0/centers/:centerId/articles?search=query` |
| Get article | GET | `/v0/centers/:centerId/articles/:articleId` |
| Create article | POST | `/v0/centers/:centerId/articles` |
| Update draft | PATCH | `/v0/centers/:centerId/articles/:articleId/draft` |
| Update metadata | PATCH | `/v0/centers/:centerId/articles/:articleId/metadata` |
| Publish | POST | `/v0/centers/:centerId/articles/:articleId/publish` |
| Unpublish | POST | `/v0/centers/:centerId/articles/:articleId/unpublish` |
| Delete | DELETE | `/v0/centers/:centerId/articles/:articleId` |
| Duplicate | POST | `/v0/centers/:centerId/articles/:articleId/duplicate` |
| List drafts | GET | `/v0/centers/:centerId/articles/drafts` |
| Get draft | GET | `/v0/centers/:centerId/articles/:articleId/draft` |
| Discard draft | POST | `/v0/centers/:centerId/articles/:articleId/draft/discard` |
| List categories | GET | `/v0/centers/:centerId/articles/categories` |
| Get center info | GET | `/v0/centers/:centerId` |
| Count articles | GET | `/v0/centers/:centerId/articles/count` |

## Content Writing Guidelines

When writing help center articles:

- **Lead with the outcome.** Start by telling the user what they'll be able to do after reading.
- **Use short paragraphs.** 2-3 sentences max per paragraph.
- **Add step-by-step instructions** with numbered lists for procedures.
- **Include examples** wherever possible to make abstract concepts concrete.
- **Use screenshots or visuals** references where helpful (note: images need to be hosted externally and referenced via URL in the HTML).
- **End with next steps** or related articles when relevant.
- **Write for scanning.** Use descriptive headings so users can jump to what they need.

## SEO Metadata

When creating articles, optionally include SEO metadata:

```json
{
  "metadata": {
    "seo": {
      "title": "Concise, keyword-rich title (50-60 chars)",
      "description": "Clear summary of the article (150-160 chars)"
    }
  }
}
```

You can also update SEO metadata on existing articles via the metadata endpoint:

```bash
curl -s -X PATCH \
  -H "Authorization: Bearer $HC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "seo": {
      "title": "SEO Title",
      "description": "SEO Description"
    }
  }' \
  "https://api.help.center/v0/centers/$HC_CENTER_ID/articles/ARTICLE_ID/metadata"
```
