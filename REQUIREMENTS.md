# REQUIREMENTS.md

This document outlines the use cases, functional requirements, and technical specifications for newsletter2paper.

## Project Overview

newsletter2paper is a web application that aggregates content from multiple Substack newsletters and generates custom PDF documents. Users can curate their own personalized newspaper by selecting publications, configuring layout preferences, and generating formatted PDFs that compile recent articles from their chosen sources.

## User Types

### Guest Users
- Can access full functionality without authentication
- Configuration stored in browser localStorage
- Cannot save issues across devices or sessions
- Limited to single-session workflows

### Authenticated Users
- Sign in via magic link (email) or OAuth providers (Google)
- Configuration persisted to database (Supabase)
- Can manage multiple issues/newspapers
- Access issues across devices and sessions
- Can revisit and regenerate previous issues

## Use Cases

### UC-1: Publication Discovery and Selection

**Primary Actor:** User (Guest or Authenticated)

**Preconditions:** User has opened the application

**Main Flow:**
1. User clicks "SEARCH" to open publication search modal
2. User enters search query (publication name, author, or topic)
3. System searches Substack directory and displays results
4. User reviews search results showing publication name, handle, and subscriber count
5. User selects/deselects publications via checkboxes
6. Selected publications appear in "Publications to Print" section
7. System auto-saves configuration (authenticated users) or stores in localStorage (guests)

**Alternative Flows:**
- **1a. Add via URL:** User clicks "ADD VIA URL" and enters publication URL directly
- **1b. Search History:** User selects from previously searched publications
- **4a. User/Publication Disambiguation:** If search returns user results, system converts to publication URL automatically

**Postconditions:** Selected publications are added to user's issue configuration

---

### UC-2: Issue Configuration

**Primary Actor:** User (Guest or Authenticated)

**Preconditions:** User has opened the application

**Main Flow:**
1. User enters newspaper title in "Configure Your Newspaper" section
2. User selects printing schedule (Last 24 Hours, Last Week, Last Month) - currently disabled, defaults to weekly
3. User selects format (Newspaper or Essay) - currently disabled, defaults to newspaper
4. System auto-saves configuration after 1 second of inactivity

**Postconditions:** Issue configuration is saved and ready for PDF generation

---

### UC-3: Article Preview

**Primary Actor:** User (Guest or Authenticated)

**Preconditions:**
- User has selected at least one publication
- Issue has been saved (authenticated users)

**Main Flow:**
1. System automatically fetches recent articles for selected publications
2. For each publication, system displays:
   - Publication name and details
   - List of available articles (title, author, date)
   - Article count
3. User reviews which articles will be included in PDF
4. Preview updates when publications are added/removed

**Alternative Flows:**
- **2a. Preview Fetch Fails:** System displays warning but allows PDF generation to proceed
- **2b. No Recent Articles:** System shows "No recent articles" message for that publication

**Postconditions:** User has visibility into which articles will be included

---

### UC-4: Image Management (Essay Format Only)

**Primary Actor:** User (Guest or Authenticated)

**Preconditions:**
- Format is set to "Essay"
- User has selected publications

**Main Flow:**
1. System displays image toggle controls for each publication
2. User clicks image icon to toggle image removal for specific publication
3. OR user uses master checkbox to toggle all publications at once
4. System updates publication settings with `remove_images` flag
5. Visual indicator shows current state (image icon vs. hide image icon)

**Postconditions:** Image removal preferences are saved per publication

---

### UC-5: PDF Generation

**Primary Actor:** User (Guest or Authenticated)

**Preconditions:**
- User has configured an issue
- At least one publication is selected

**Main Flow:**
1. User clicks "Generate PDF" button
2. System displays loading state
3. Backend fetches recent articles from RSS feeds for all selected publications
4. Backend prepares article data with metadata (title, author, date, content)
5. Backend calls Go PDF service with article data and layout preferences
6. Go service:
   - Fetches article content if not provided
   - Downloads and processes images (respecting remove_images settings)
   - Generates HTML from appropriate template (newspaper or essay)
   - Converts HTML to PDF using WeasyPrint
7. Backend uploads PDF to Supabase storage
8. System displays PDF download link to user
9. User clicks link to download or view PDF

**Alternative Flows:**
- **3a. No Recent Articles:** System returns error "No articles found"
- **6a. Article Fetch Fails:** System continues with successfully fetched articles, logs errors
- **6b. PDF Generation Fails:** System displays error message to user
- **7a. Storage Upload Fails:** System returns error with details

**Postconditions:** PDF is generated and accessible via public URL

---

### UC-6: Authentication

**Primary Actor:** Guest User

**Preconditions:** User is not authenticated

**Main Flow:**
1. User clicks "Sign In" button
2. System displays authentication modal
3. User chooses authentication method:
   - **Magic Link:** Enters email address
   - **OAuth:** Selects Google provider
4. System processes authentication
5. User receives confirmation (magic link email or OAuth redirect)
6. User completes authentication
7. System migrates localStorage data to database
8. User is redirected to application with authenticated session

**Postconditions:** User is authenticated and data is persisted

---

### UC-7: Issue Management (Authenticated Only)

**Primary Actor:** Authenticated User

**Preconditions:** User is signed in

**Main Flow:**
1. User clicks dropdown menu next to issue title
2. System displays list of user's saved issues
3. User selects an issue from the list
4. System loads issue configuration:
   - Title
   - Format
   - Selected publications with settings
5. User can modify configuration or generate new PDF

**Alternative Flows:**
- **2a. No Saved Issues:** System shows empty state
- **5a. Create New Issue:** User can start fresh configuration

**Postconditions:** Selected issue is loaded and active

---

## Functional Requirements

### Publication Management

**FR-1.1:** System shall support searching Substack publications by name, author, or topic
**FR-1.2:** System shall allow adding publications via direct URL
**FR-1.3:** System shall automatically discover RSS feed URLs from publication URLs
**FR-1.4:** System shall support both publication and user (author) search results
**FR-1.5:** System shall maintain search history for quick re-selection
**FR-1.6:** System shall validate publication URLs before adding
**FR-1.7:** System shall prevent duplicate publications in same issue

### Issue Configuration

**FR-2.1:** System shall allow custom issue titles (text input)
**FR-2.2:** System shall support two layout formats: Newspaper and Essay
**FR-2.3:** System shall support three time ranges: Last 24 Hours, Last Week, Last Month
**FR-2.4:** System shall auto-save configuration changes after 1 second delay
**FR-2.5:** System shall persist authenticated user configurations to database
**FR-2.6:** System shall store guest user configurations in browser localStorage
**FR-2.7:** System shall allow selecting unlimited publications per issue

### Article Fetching

**FR-3.1:** System shall fetch articles from RSS feeds based on time range
**FR-3.2:** System shall respect max articles per publication limit (default: 5)
**FR-3.3:** System shall parse RSS feed XML to extract article metadata
**FR-3.4:** System shall extract: title, author, publication date, content URL, description
**FR-3.5:** System shall handle both full content and summary-only RSS feeds
**FR-3.6:** System shall provide article previews before PDF generation
**FR-3.7:** System shall gracefully handle feed fetch failures
**FR-3.8:** System shall support common RSS/Atom feed formats

### PDF Generation

**FR-4.1:** System shall generate PDFs in two layout types:
- **Newspaper:** Multi-column layout with masonry-style article arrangement
- **Essay:** Single-column reading format

**FR-4.2:** System shall include in generated PDFs:
- Issue title as header
- Article title, author, publication name, date
- Article content with formatting preserved
- Images (unless removed via settings)
- Table of contents (for newspaper format)

**FR-4.3:** System shall support per-publication image removal
**FR-4.4:** System shall support global image removal override
**FR-4.5:** System shall upload generated PDFs to cloud storage
**FR-4.6:** System shall provide public download URLs for generated PDFs
**FR-4.7:** System shall generate PDFs within 120 seconds timeout
**FR-4.8:** System shall clean up temporary files after generation
**FR-4.9:** System shall preserve HTML for debugging when requested
**FR-4.10:** System shall handle concurrent PDF generation requests

### Authentication & User Management

**FR-5.1:** System shall support magic link authentication via email
**FR-5.2:** System shall support OAuth authentication (Google)
**FR-5.3:** System shall not require authentication for core functionality
**FR-5.4:** System shall migrate guest data to authenticated account on sign-in
**FR-5.5:** System shall maintain user sessions across browser restarts
**FR-5.6:** System shall allow users to sign out
**FR-5.7:** System shall display user profile information when authenticated

### Issue Management

**FR-6.1:** System shall allow authenticated users to save multiple issues
**FR-6.2:** System shall allow switching between saved issues
**FR-6.3:** System shall track issue creation and update timestamps
**FR-6.4:** System shall allow loading previous issue configurations
**FR-6.5:** System shall preserve publication selections per issue
**FR-6.6:** System shall preserve layout and image settings per issue

### User Interface

**FR-7.1:** System shall provide responsive design for desktop and mobile
**FR-7.2:** System shall display loading states during async operations
**FR-7.3:** System shall show error messages for failed operations
**FR-7.4:** System shall provide visual feedback for user actions
**FR-7.5:** System shall display article counts per publication
**FR-7.6:** System shall show image removal status indicators
**FR-7.7:** System shall provide search result previews with metadata
**FR-7.8:** System shall display publication subscriber counts when available

## Non-Functional Requirements

### Performance

**NFR-1.1:** RSS feed discovery shall complete within 10 seconds
**NFR-1.2:** Article preview fetch shall complete within 5 seconds
**NFR-1.3:** PDF generation shall complete within 120 seconds
**NFR-1.4:** UI auto-save debounce shall be 1 second
**NFR-1.5:** Search results shall display within 2 seconds
**NFR-1.6:** System shall support at least 10 concurrent PDF generations

### Reliability

**NFR-2.1:** System shall handle RSS feed fetch failures gracefully
**NFR-2.2:** System shall retry failed network requests up to 3 times
**NFR-2.3:** System shall continue PDF generation with partial article set on failures
**NFR-2.4:** System shall maintain data consistency across services
**NFR-2.5:** System shall clean up resources after PDF generation

### Scalability

**NFR-3.1:** System shall support unlimited publications per issue
**NFR-3.2:** System shall handle PDFs with up to 100 articles
**NFR-3.3:** System shall scale horizontally for API and PDF services
**NFR-3.4:** Database shall handle at least 10,000 users
**NFR-3.5:** Storage shall accommodate at least 100GB of PDFs

### Security

**NFR-4.1:** System shall validate all user inputs
**NFR-4.2:** System shall sanitize HTML content from RSS feeds
**NFR-4.3:** System shall use HTTPS for all external requests
**NFR-4.4:** System shall protect against XSS and injection attacks
**NFR-4.5:** System shall use secure authentication tokens (Supabase)
**NFR-4.6:** System shall not expose sensitive credentials in responses
**NFR-4.7:** System shall implement CORS policies for API access

### Usability

**NFR-5.1:** System shall be usable without documentation for basic tasks
**NFR-5.2:** Error messages shall be clear and actionable
**NFR-5.3:** System shall provide immediate feedback for user actions
**NFR-5.4:** UI shall follow consistent design patterns
**NFR-5.5:** System shall be accessible via keyboard navigation

### Compatibility

**NFR-6.1:** Frontend shall support modern browsers (Chrome, Firefox, Safari, Edge)
**NFR-6.2:** System shall work on desktop and mobile devices
**NFR-6.3:** PDFs shall be compatible with standard PDF readers
**NFR-6.4:** System shall support various RSS/Atom feed formats
**NFR-6.5:** API shall follow RESTful conventions

## Technical Requirements

### Frontend Stack

- **Framework:** Next.js 15.5.9 (React 19)
- **UI Library:** Material-UI (MUI) 7.3.2
- **Styling:** Tailwind CSS 4
- **State Management:** React Context API
- **Testing:** Vitest 3.2.4
- **Authentication:** Supabase Auth

### Backend Stack

- **API Framework:** FastAPI 0.116.2
- **Database:** PostgreSQL (via Supabase)
- **ORM:** SQLModel 0.0.25
- **PDF Generation:** Go service with WeasyPrint
- **HTML Processing:** BeautifulSoup4 4.13.5
- **RSS Parsing:** Native XML parsing
- **Storage:** Supabase Storage

### PDF Service Stack

- **Language:** Go 1.22.0
- **HTML Parsing:** goquery (github.com/PuerkitoBio/goquery)
- **Concurrency:** golang.org/x/sync
- **PDF Generation:** WeasyPrint (via HTML/CSS)

### Infrastructure

- **Container Orchestration:** Docker Compose
- **API Server:** Uvicorn (production: Gunicorn)
- **Reverse Proxy:** CORS middleware
- **File Sharing:** Docker volumes
- **Deployment:** Container-based (FastAPI + Go sidecar pattern)

### Database Schema

**Tables:**
- `users` - User accounts and profiles
- `issues` - Newsletter configurations
- `publications` - Substack publication metadata
- `articles` - Fetched article content
- `issue_publications` - Many-to-many with settings
- `user_issues` - Many-to-many for issue ownership

### API Endpoints

**Issues:**
- `POST /issues/` - Create issue
- `PUT /issues/{issue_id}` - Update issue
- `GET /issues/{issue_id}` - Get issue
- `POST /issues/{issue_id}/publications` - Add publications to issue
- `GET /issues/{issue_id}/publications` - Get issue publications

**Publications:**
- `GET /publications/` - List/search publications
- `POST /publications/` - Create publication
- `GET /publications/{publication_id}` - Get publication

**Articles:**
- `POST /articles/fetch/{issue_id}` - Fetch articles for issue
- `GET /articles/issue/{issue_id}/summary` - Article summary

**RSS:**
- `GET /rss/feed-url` - Discover RSS feed URL
- `GET /rss/articles` - Fetch articles from feed

**PDF:**
- `POST /pdf/generate/{issue_id}` - Generate PDF
- `GET /pdf/download/{issue_id}` - Download PDF
- `GET /pdf/status/{issue_id}` - Check generation status

## System Capabilities

### Content Aggregation
- Automatic RSS feed discovery from URLs
- Multi-source article fetching
- Parallel article download (up to 4 concurrent)
- HTML content extraction and cleaning
- Image download and processing
- Metadata extraction (title, author, date, publication)

### PDF Customization
- Two layout templates (newspaper, essay)
- Per-publication image control
- Custom issue titles
- Date range filtering
- Article count limits
- Responsive typography and spacing

### Data Management
- Multi-user support with isolation
- Guest mode with localStorage
- Issue versioning (created_at, updated_at)
- Publication deduplication
- Auto-save with debouncing
- Cross-device synchronization (authenticated)

### Integration Points
- Supabase Authentication
- Supabase PostgreSQL Database
- Supabase Object Storage
- Substack RSS feeds
- External publication URLs
- OAuth providers (Google)

## Future Considerations

**Potential Enhancements:**
- Email delivery of generated PDFs
- Scheduled/recurring PDF generation
- Custom CSS theming for PDFs
- Additional layout templates
- Article filtering and sorting options
- Publication categories/tags
- Collaborative issue sharing
- Export to other formats (EPUB, MOBI)
- Reader mode with annotations
- Advanced search with filters
- Publication recommendations
- Analytics and reading statistics
- Mobile native applications
- Offline reading support
- Integration with other newsletter platforms (Ghost, Medium)

**Technical Debt:**
- Enable printing schedule selection (currently disabled)
- Enable format selection (currently disabled)
- Add comprehensive test coverage
- Implement API rate limiting
- Add request caching layer
- Optimize database queries with indexes
- Implement CDN for static assets
- Add comprehensive logging and monitoring
- Create admin dashboard
- Document all API endpoints (OpenAPI/Swagger)
