# Design Document: Magic Mirror Compliments Plugin

## 1. The Problem
Magic Mirror users often compile rich, personalized datasets of compliments and inside jokes (stored in a `compliments.json` file) that rotate on their smart mirrors. However, these mirrors are usually stationary in specific locations—like a hallway or front door—meaning users only see the quotes in passing few times a day. 

We wanted to design a way to display these same curated quotes on an ambient, low-power TRMNL e-ink device placed in higher-traffic areas, like a living room or office desk. Currently, there is no TRMNL integration that seamlessly parses a standard Magic Mirror `compliments.json` file, scales the layout dynamically for varying quote sizes, and respects the original time-of-day category logic.

## 2. The Technical Plan
To solve this cleanly, we will build a "Recipe"—which is TRMNL's term for a serverless plugin that runs entirely on their own infrastructure.

The architecture relies on three pieces fitting seamlessly together:
1. **The Source Data:** The user's existing `compliments.json` file, typically hosted securely as a raw file on GitHub or a Gist.
2. **The Settings Form:** A simple, 1-click install form inside the TRMNL dashboard that asks the user to provide the link to their personal JSON file.
3. **The TRMNL Hardware:** The target e-ink display (e.g., TRMNL OG) is a 7.5" screen rendering at 800x480 resolution in 2-bit color. This constraint mandates high-contrast, scalable, edge-to-edge typography.
4. **The TRMNL Core Backend:** TRMNL's servers are responsible for making the HTTP request. We configure the Recipe's polling target by interpolating the user's setup variables (e.g., target URL: `{{ plugin.json_url }}`). TRMNL securely fetches this endpoint hourly and injects the parsed JSON tree into our template as the `payload` variable.
5. **The Liquid Rendering Engine:** The custom `markup.html` is strictly a stateless visual templating layer. It interprets the `payload`, dynamically filters the array categories based on the user's local timezone offset in `plugin.time_zone`, organically pseudo-randomizes the active array, and sizes the output cleanly into the 800x480 bounding box.

Because the data fetching and rendering layers natively couple through TRMNL's built-in platform tools, it requires zero active API maintenance, no web servers, and works seamlessly for any user.

### File Storage Strategy
The source code for this Recipe (including this Design Document, the `markup.html`, and `variables.yml`) will be developed locally in the workspace and synced to a matching public GitHub repository. This serves as the version-controlled source of truth, enabling transparency and community contributions.

### Publishing Process & Best Practices
To make this available to all Magic Mirror users as a 1-click install, we will submit the Recipe to the TRMNL public directory for their team's manual review process. To ensure we pass their automated linter ("Chef") and provide the best UX, we will strictly adhere to their documented best practices:
1. **Framework Classes over Inline Styles:** We will rigorously eliminate inline CSS `style` attributes (like `display`, `padding`, `font-size`, `text-align`) and exclusively use TRMNL's native Tailwind-based Framework UI utility classes.
2. **Optimized Form Links:** When directing users to external sites in our `variables.yml` configuration descriptions, we will use properly formatted HTML `<a>` tags with the `underline` class, rather than raw text URLs, ensuring links are accessible and open seamlessly in a new tab.

## 3. Alternatives Considered

**Third-Party Plugin (External Web Server)**
* **The Idea:** Build an external microservice (e.g., Google Apps Script, Vercel function) to fetch the `compliments.json`, process the rigorous array merging and randomness logic, and serve a pre-formatted payload to TRMNL.
* **Why it was ruled out:** While programming in a real server environment is more forgiving than templating engines, requiring a server ruins the community aspect. Sharing the plugin would mean either forcing users to deploy their own web servers (a huge barrier to entry) or hosting it centrally (which incurs maintenance/operations overhead). A native Recipe is completely serverless.

**Deterministic Cycling (vs. Randomness)**
* **The Idea:** Instead of randomly sprinkling in "anytime" or undocumented quote categories, track them to display sequentially on a rigorous schedule (e.g., exactly every 4th refresh).
* **Why it was ruled out:** TRMNL Recipes are entirely stateless and do not communicate sequentially with past executions. Guaranteeing an exact "every nth time" cycle would mandate persistent database storage. We opted for probabilistic randomness (using TRMNL's custom `sample` Liquid filter) to achieve a dynamic feel with zero state, fully mitigating the sequence-caching risks that normally plague math workarounds.

**Native Array Concatenation (The `concat` Filter)**
* **The Idea:** Iterate through the JSON object and use Liquid's `array1 | concat: array2` filter to merge unknown categories (e.g., `snowing`) directly into the `anytime` pool natively as objects.
* **Why it was ruled out:** In embedded Liquid environments, dynamically instantiating an empty array (`[]`) to act as the master pool, and strictly typing dynamic JSON node values as arrays before concatenating, is notoriously brittle and prone to type-mismatch crashes. We opted for the **String Append & `split` Hack**, which coerces everything safely to a string buffer and splits it back into an array at the end, guaranteeing type safety.

## 4. Detailed Implementation

We will initialize the project in the local `trmnl` workspace. This will serve as the source-of-truth repository for the TRMNL Recipe.

### Payload Strictness & Schema Integration
To guarantee flawless Liquid traversal, this Recipe enforces strict adherence to Magic Mirror's default JSON flat schema:
- The `compliments.json` file **MUST** be a flat, top-level dictionary object `{ "morning": ["value1"], "anytime": ["value2"] }`.
- Category keys **MUST** be strictly lowercase string iterables. Our Liquid iterator handles valid flat lists but structurally ignores misformatted inputs or complex nested JSON objects to prevent polluting the display string with casted `[object Object]` metadata errors.

### Proposed Files to Create

#### [NEW] `/home/joeromeo/Documents/Antigravity/projects/trmnl/markup.html`
* **Rationale:** This is the core engine and visual frontend of the Recipe. Because TRMNL evaluates Liquid natively, this single file serves as both the logic controller and the layout view.
* **Responsibilities:** 
  - **Native Timezone Resolution:** Manual integer hour math fails catastrophically for half-hour timezones (e.g., India) and Daylight Saving calculations. Instead, TRMNL provides a native `time_zone` field configured with standardized TZ string database names (e.g., `"America/Denver"`). The template will securely evaluate the UTC `'now'` object using either TRMNL's active template rendering context or explicitly routing it through Liquid's `in_time_zone: plugin.time_zone` filter to accurately resolve the user's localized hour without brittle modulo-wrapping math.
  - **Threshold Execution:** Execute the "Time of Day" logical routing natively against the standard MagicMirror² boundaries (Morning: 00:00–11:59, Afternoon: 12:00–16:59, Evening: 17:00–23:59).
  - **Fault Tolerance & Error UI:** If TRMNL attempts to fetch a standard GitHub or Gist HTML page (`gist.github.com` without `usercontent`), the `payload` fails silently. Instead of defaulting to generic text, the template explicitly renders a full-screen Error UI: *"Setup Error: Please edit your recipe and use the 'Raw' file URL."* (Unlike the settings page, this error UI contains no HTML `<a>` tags since e-ink screens are unclickable).
  - **Payload Extraction:** Bypassing standard iterator loops, the engine natively plucks target data arrays dynamically surfaced inside the global scope (e.g. `anytime` or `morning`) based entirely on local-time variables and explicit allow-lists. This flat payload pipeline prevents iterator bloating and completely avoids TRMNL API un-namespaced parsing flaws out of the box.
  - **String Buffering & Bounds:** To bypass Liquid concatenations casting payloads into `[object Object]` data errors, we bundle the raw quote alongside its explicit origin category key utilizing mathematically obscure delimiters (`~~~TRMNL_CAT~~~` and `~~~TRMNL_BOUND~~~`). Binding the metadata explicitly guarantees the badge footer accurately identifies the true active array of any randomly string sampled fallback logic without blindly assuming active system time contexts.
  - **Dynamic Typography & Layout Safety:** We assign font-size mapping stepping progressively down from `text-6xl` to `text-3xl` using tightly constrained size boundaries (115, 65, 32 points) securing the flex coordinates so extreme URL paragraphs never shatter TRMNL bounds limits. The final randomized output is explicitly piped through `strip_html` prior to `newline_to_br` stripping potentially malicious `<script>` boundaries out of raw payloads safely natively.
  - **Grid & Visual Layout:**
    - **Canvas Body:** A clean, container-bound flex interface with a centered primary responsive text area.
    - **Oversized Quotes:** Two explicitly sized (`width="120" height="120"`) hardcoded inline quotation mark SVGs natively positioned via classic absolute CSS positioning overriding TRMNL framework restrictions. To accurately align closing quote typographical bounds, the opening quote mirrors securely via inverted `-scale-x-100` path flipping avoiding baseline drifting mathematics.
    - **Stippled SVG Footer:** *(Currently commented out pending future refactoring)* The dedicated bottom bar and metadata badge components have been temporarily disabled due to aggressive TRMNL CSS sanitizer logic stripping inline pattern-dots and data-URI stipple implementations causing unpredictable overlaps.
    - **Footer Branding & Metadata:** Minimalist iconography alongside dynamic category badge UIs definitively matching the true array sampled UIs.
  
#### [NEW] `/home/joeromeo/Documents/Antigravity/projects/trmnl/variables.yml`
* **Rationale:** TRMNL allows developers to define dynamic configuration pages for their Recipes using YAML form field mappings. 
* **Responsibilities:** 
  - Define a `string` URL input labeled "Magic Mirror JSON URL" so the user can easily link their payload source. This input will explicitly leverage TRMNL's `description`, `help_text`, and `placeholder` options to warn the user against two critical failure paths: providing a standard `.com/blob` GitHub HTML page, and providing a raw Gist URL that still contains the frozen commit hash (which permanently prevents future JSON updates from syncing).
  - Define a native `time_zone` dropdown selector which captures the standard TZ database name (e.g., `"America/New_York"` or `"Asia/Kolkata"`), enabling the `markup.html` template to seamlessly handle DST and fractional timezone offsets natively without writing custom integer operators.

### Liquid Scope Namespaces
TRMNL executes the backend `polling_url` API interpolator natively with simplified top-level variables (e.g., `{{ json_url }}`). However, inside the actual `markup.html` sandbox rendering engine, custom interface variables do not persist natively at the root. Liquid logic must explicitly evaluate the full globally namespaced object mapping to map the configuration dictionary and bypass false `nil` branch failures:
- **`trmnl.plugin_settings.custom_fields_values.json_url`**
- **`trmnl.plugin_settings.custom_fields_values.time_zone`**

#### [EXISTING] `/home/joeromeo/Documents/Antigravity/projects/trmnl/DESIGN.md`
* **Rationale:** This is the document you are currently reading. It establishes a permanent source of truth for the project. **CRITICAL AI INSTRUCTION:** Do not overwrite, regenerate, or recreate this file during the execution phase. This file already exists and serves exclusively as your blueprint.
* **Responsibilities:** 
  - Preserve the foundational blueprint (The Problem, The Technical Plan, Alternatives).
  - Document the architecture and Liquid logic decisions for future contributors.
