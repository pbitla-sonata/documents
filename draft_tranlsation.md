🗺️ Implementation Roadmap
Phase 1: Static UI Strings (2-3 weeks)
Goal: Translate filter labels, placeholders, and aria labels

Task	Deliverable	Repository
Add message IDs to messages.js	Filter titles, placeholders, aria labels defined	frontend-enterprise
Update FacetDropdown.jsx	Uses intl.formatMessage for dynamic translation	frontend-enterprise
Update TypeaheadFacetDropdown.jsx	Uses message keys for placeholders	frontend-enterprise
Extract i18n strings	Run npm run i18n_extract	frontend-enterprise
Push to Transifex	Submit to openedx-translations	openedx-translations
✅ Pros:

Quick win with high user impact
No backend changes required
Leverages existing i18n infrastructure
❌ Cons:

Only translates UI chrome, not data values
Requires coordination with translation team
Phase 2: Semi-Dynamic Facets (Subjects) (1-2 weeks)
Goal: Translate ~50-100 subject values via static message files

Task	Deliverable
Enumerate all subjects from DB	List of 50-100 subject names
Add subjects to messages.js	One defineMessages entry per subject
Extract and translate	Push to openedx-translations
✅ Pros:

No database schema changes
Reuses existing FacetItem.jsx lookup pattern
Works immediately for all locales
❌ Cons:

Manual maintenance when new subjects added
Not scalable for skills (5,000+ values)
Phase 3: Backend Translation Infrastructure (4-6 weeks) ⚠️ CRITICAL
Goal: Build TranslatedFacetCache and multi-locale pipeline

3.1 Database Setup
Input Requirements:

Python
# Required fields for TranslatedFacetCache
{
  "english_value": "Machine Learning",      # Source string (from json_metadata)
  "field_type": "skill",                     # skill | subject | ability
  "language_code": "es",                     # Target locale
  "translated_value": "Aprendizaje Automático", # AI-generated translation
  "source": "xpert_ai"                       # xpert_ai | human | google_translate
}
Database Schema:

Python
class TranslatedFacetCache(TimeStampedModel):
    english_value = models.CharField(max_length=500, db_index=True)
    field_type = models.CharField(max_length=50, choices=[...])
    language_code = models.CharField(max_length=10)
    translated_value = models.CharField(max_length=500)
    source = models.CharField(max_length=20, default='xpert_ai')
    
    class Meta:
        unique_together = ('english_value', 'field_type', 'language_code')
        indexes = [
            models.Index(fields=['english_value', 'field_type', 'language_code']),
        ]
Storage Strategy:

One row per unique value per locale (deduplication at database level)
Example: "Python" skill → 20 rows (one per locale), not 200,000 rows (one per course per locale)
Cost Comparison:

Metric	Without Cache	With TranslatedFacetCache
Xpert AI calls (bootstrap)	2,000,000	100,000
Weekly AI calls (100 new skills)	100,000	2,000
DB storage	~500 MB JSON	~5 MB rows
Fix one translation	Update 50-100 rows	Update 1 row
3.2 Translation Pipeline
Management Command: populate_translations

bash
# Weekly incremental run
python manage.py populate_translations --facets-only --incremental

# Bootstrap new locale
python manage.py populate_translations --language-codes fr --full

# Batch size control
python manage.py populate_translations --batch-size 50
Algorithm Flow:

Mermaid
graph TB
    A[Collect unique skills/subjects from DB] --> B{For each locale}
    B --> C[Query TranslatedFacetCache for existing translations]
    C --> D{Cache misses?}
    D -->|Yes| E[Batch into groups of 50]
    D -->|No| F[Next locale]
    E --> G[Call Xpert AI batch_translate]
    G --> H{Parse successful?}
    H -->|Yes| I[bulk_create to TranslatedFacetCache]
    H -->|No| J[Log error, keep English fallback]
    J --> I
    I --> K{More batches?}
    K -->|Yes| E
    K -->|No| F
    F --> L{All locales done?}
    L -->|No| B
    L -->|Yes| M[Trigger reindex_algolia]
Required Inputs for Translation:

Python
# Input to Xpert AI batch_translate
{
  "values": ["Python", "Machine Learning", "Data Analysis"],
  "language_code": "es",
  "field_type": "skill"  # for context
}

# Expected Output
{
  "translations": [
    "Python",  # Technical terms often untranslated
    "Aprendizaje Automático",
    "Análisis de Datos"
  ]
}
3.3 Algolia Indexing
Function: create_localized_algolia_object

Python
def create_localized_algolia_object(algolia_object, content_metadata, language_code):
    # 1. Fetch pre-translated title/description from ContentTranslation
    translation = content_metadata.translations.filter(language_code=language_code).first()
    if not translation:
        return None
    
    # 2. Deep copy English object
    localized = copy.deepcopy(algolia_object)
    
    # 3. Apply text translations
    localized['title'] = translation.title
    localized['description'] = translation.full_description
    
    # 4. Translate skill_names and subjects from cache
    skill_names = localized.get('skill_names', [])
    cache_hits = TranslatedFacetCache.objects.filter(
        english_value__in=skill_names,
        field_type='skill',
        language_code=language_code
    ).values_list('english_value', 'translated_value')
    
    lookup = dict(cache_hits)
    localized['skill_names'] = [lookup.get(s, s) for s in skill_names]  # Fallback to EN
    
    # 5. Mark object with locale
    localized['objectID'] = f"{algolia_object['objectID']}-{language_code}"
    localized['metadata_language'] = language_code
    
    return localized
Algolia Object Structure:

JSON
// English object
{
  "objectID": "course-v1:edX+CS50",
  "metadata_language": "en",
  "title": "Introduction to Computer Science",
  "skill_names": ["Python", "Algorithms"],
  "subjects": ["Computer Science"],
  "level_type": "Introductory"
}

// Spanish object (co-indexed)
{
  "objectID": "course-v1:edX+CS50-es",
  "metadata_language": "es",
  "title": "Introducción a las Ciencias de la Computación",
  "skill_names": ["Python", "Algoritmos"],
  "subjects": ["Ciencias de la Computación"],
  "level_type": "Introductory"
}
✅ Pros:

Massive AI cost reduction (100x fewer API calls)
Fast indexing (cache lookup vs AI call)
Human corrections protected (via source field)
Future-proof (add new field_types without schema changes)
❌ Cons:

New model to maintain (~2 days dev time)
Two DB reads during indexing (ContentTranslation + TranslatedFacetCache)
Index size multiplied by 20 (must verify Algolia plan limits)
Phase 4: Frontend Locale Routing (1-2 weeks)
Goal: Filter Algolia queries by user's locale

4.1 Locale Resolution
TypeScript
// useAlgoliaSearch.ts
const ALGOLIA_LOCALE_MAP: Record<string, string> = {
  'en-us': 'en',
  'es-419': 'es',  // Latin American Spanish → Spain Spanish
  'zh-tw': 'zh-cn', // Traditional → Simplified until fully supported
  'fr-ca': 'fr',
  // ... 20+ mappings
};

function resolveAlgoliaLocale(locale: string): string {
  const lower = locale.toLowerCase();
  return ALGOLIA_LOCALE_MAP[lower] 
    || ALGOLIA_LOCALE_MAP[lower.split('-')[0]]  // Base language fallback
    || 'en';  // Ultimate fallback
}
4.2 Query Filtering
jsx
// SearchPage.jsx
const { baseFilter } = useAlgoliaSearch();
// baseFilter = "metadata_language:'es'"

<Configure filters={baseFilter} />
⚠️ CRITICAL DEPENDENCY: Phase 3 backend must be fully deployed with at least one non-English locale indexed in Algolia BEFORE deploying Phase 4 frontend. Otherwise, non-English users get zero search results.

✅ Pros:

Simple implementation (~50 lines of code)
Instant locale switching
No query performance impact (indexed field)
❌ Cons:

Hard dependency on backend deployment
Regional variant mapping requires maintenance
Fallback testing critical (missing locales must not break search)
Phase 5: QA & Edge Cases (2 weeks)
Test Case	Expected Behavior
RTL locales (ar, he)	Layout mirrors correctly, filters right-aligned
Locale switch mid-session	Facets fully re-render with new language
Missing translation	Graceful fallback to English (never blank)
Mixed language prevention	French user never sees English skill values
Algolia index overflow	Verify plan supports 20× object count
📊 Complete Data Flow: Input → Database → Algolia
Mermaid
graph LR
    A[course-discovery DB<br/>json_metadata<br/>skills: Python, ML] --> B[populate_translations cmd]
    B --> C{TranslatedFacetCache<br/>en: Python → es: Python<br/>en: ML → es: Aprendizaje Auto}
    B --> D[Xpert AI<br/>Batch 50 items]
    D --> C
    C --> E[create_localized_algolia_object]
    F[ContentTranslation<br/>title/description] --> E
    E --> G[Algolia Index<br/>objectID: ...-es<br/>metadata_language: es<br/>skill_names: Python, Aprendizaje Auto]
    H[Frontend<br/>locale=es] --> I[useAlgoliaSearch<br/>baseFilter: metadata_language:es]
    I --> G
    G --> J[FacetDropdown<br/>Shows: Python, Aprendizaje Automático]
🎯 Exact Requirements Summary
Translation Inputs Required:
Static UI Strings:

Filter titles (10 strings)
Placeholders (10 strings)
Aria labels (10 strings)
Source: Developer-defined in code
Translated by: Transifex/openedx-translations
Closed-Set Facets:

Level: Introductory, Intermediate, Advanced
Availability: Available Now, Upcoming, Starting Soon, Archived
Learning Type: Course, Program, Video, Pathway
Source: Hardcoded constants
Translated by: Existing messages.js
Semi-Dynamic Facets:

Subjects: ~50-100 values (e.g., Computer Science, Business)
Source: course_discovery.Subject model
Translated by: Phase 2 (messages.js) + Phase 3 (cache)
Fully Dynamic Facets:

Skills: 5,000-50,000 values (e.g., Machine Learning, PyTorch)
Source: json_metadata.skill_names across all courses
Translated by: Phase 3 (TranslatedFacetCache + Xpert AI)
Database Storage:
Table	Key	Values per Locale	Total Rows (20 locales)
ContentTranslation	(content_key, language_code)	100,000 courses	2,000,000
TranslatedFacetCache	(english_value, field_type, language_code)	5,000 unique skills	100,000
Algolia Storage:
Index	Objects	Differentiation
product (single index)	100,000 courses × 20 locales = 2,000,000 objects	metadata_language field + objectID suffix
⚠️ Critical Decision Points
Option A: Extend ContentTranslation (No New Table)
Pro: Zero new models, 1-day implementation
Con: 2M AI calls at bootstrap, no deduplication, 500 MB storage
Option B: TranslatedFacetCache (Recommended)
Pro: 100K AI calls, 5 MB storage, human corrections protected
Con: 2-day implementation, new model to maintain
Recommendation: Option B — the 20× AI cost savings and clean correctability justify the 2-day investment.

📅 Timeline Summary
Phase	Duration	Deliverable	Blockers
Phase 1	2-3 weeks	Static UI translated	None
Phase 2	1-2 weeks	Subjects translated	Phase 1 complete
Phase 3	4-6 weeks	Backend pipeline + cache	None (parallel with Phase 1-2)
Phase 4	1-2 weeks	Frontend locale routing	Phase 3 deployed + 1 locale indexed
Phase 5	2 weeks	QA all 20 locales	Phase 4 deployed
Total: ~12-15 weeks (3-4 months)

🚀 Getting Started Checklist
 Confirm Algolia plan supports 2M objects
 Verify Xpert AI batch translation API contract
 Enumerate all 50-100 subjects from course_discovery.Subject
 Estimate unique skill count: SELECT COUNT(DISTINCT jsonb_array_elements_text(json_metadata->'skill_names'))
 Identify regional locale priorities (es-419, pt-br, zh-cn, ar first?)
 Set up weekly cron job environment for populate_translations
 Create Datadog dashboard for translation coverage metrics
This roadmap provides everything needed to implement the solution systematically with clear inputs, outputs, and decision points at each phase.
