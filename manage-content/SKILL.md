---
name: manage-content
description: Create and manage OFBiz content data. Use when defining Content, DataResource, or linking content to products/categories, and when fetching or rendering content in Java/Groovy, FreeMarker, or XML Widgets.
---

# Skill: manage-content

## Goal
To correctly manage the OFBiz Content Management System (CMS) data, including data resources, metadata, and their associations, and to fetch and render them optimally.

## Triggers
**ALWAYS** read this skill when:
- Creating content-related data files (e.g., `DataResource`, `Content`, `ElectronicText`).
- Linking images, documents, or text to entities (e.g., `ProductContent`, `CategoryContent`, `ContentAssoc`).
- Fetching content in Groovy or Java.
- Rendering content in FTL (FreeMarker) templates or XML Form/Screen Widgets.
- Troubleshooting content visibility, rendering formatting, or retrieval issues.

## Rules & Procedures
### 1. Data Management
1. **The Separation of Concerns**: Keep in mind the strict separation: `DataResource` = Raw File/Text/URL, `Content` = Metadata, and `Association` (e.g., `ProductContent`) = The Link.
2. **DataResource Types**:
    - `SHORT_TEXT`: Use for strings under 255 chars. Store directly in the `objectInfo` field.
    - `ELECTRONIC_TEXT`: Use for long strings or HTML. **Must** be paired with an `<ElectronicText>` tag containing `textData`.
    - `URL_RESOURCE`: Use for referencing an external file through `objectInfo` (e.g., `objectInfo="component://componentname/path/file.txt"`).
3. **Generic Content Definition (`contentTypeId`)**: Defines the core purpose of the content in the engine.
    - `DOCUMENT`: The standard type for generic articles, text, or file-based documents.
    - `TEMPLATE`: Used for Freemarker (FTL) templates or widgets that define the layout of a page.
4. **Domain-Specific Content Types**: When associating content with specific entities, use the appropriate domain content type to instruct the application on how to render or process it:
    - **ProductContentType** (eCommerce/Catalog): Heavily used for UI rendering. Examples include `PRODUCT_NAME`, `DESCRIPTION`, `LONG_DESCRIPTION`, `SMALL_IMAGE_URL`, `LARGE_IMAGE_URL`, `DETAIL_IMAGE_URL`, and `ADDITIONAL_IMAGE_1`.
    - **PartyContentType** (CRM/User Management): Associates documents or images with a person or company. Examples include `INTERNAL` (private internal notes), `USERDEF` (user-defined uploaded documents), and `LGOIMGURL` (company logo).
    - **WorkEffortContentType**: Used for tasks, events, or workflows.
5. **FreeMarker (FTL) Rendering Option**: If the stored text itself contains FreeMarker instructions, you **must** set `dataTemplateTypeId="FTL"` on the `DataResource` so it is parsed before rendering.

### 2. Rendering Strategies: When to Use What
**Summary Rule of Thumb:**
- Localized data explicitly attached to a record (like Product Name) ➡️ Use a **ContentWrapper**.
- Independent CMS block, banner, or image where you just know the ID ➡️ Use **`<@ofbizContent>`**.
- Requires external API calls, complex entity aggregations, or pre-processing compute the final HTML string ➡️ Prepare in **Groovy** and output with **`${variable?no_esc!}`**.

1. **FTL Directives (`<@ofbizContent>`, `<@ofbizContentUrl>`)**: Use for generic content that doesn't require complex business logic to locate.
    - **When to use**: Static or known Content IDs (e.g., hardcoded IDs like standard company logo, terms & conditions), native CMS usage (generic content blocks), or Asset URLs (direct CDN/server URLs for resources).
    - **Benefits**: Keeps Groovy scripts clean of presentation concerns; macros automatically handle DataResource lookups, MIME-type checking, and permission validation.
2. **Entity Content Wrappers (e.g., `productContentWrapper.get()`)**: Use when content is tightly coupled to a dedicated domain entity (Product, Category, Party, WorkEffort, etc.).
    - **When to use**: Entity-Associated Content mapped via bridge tables, Localization/i18n (automatic locale fallback), and Time-Based Filtering (fetches active content based on `fromDate`/`thruDate`).
    - **Benefits**: Extremely efficient built-in caching specifically for entity-content mappings; prevents complex SQL/Entity queries in Groovy.
3. **Groovy-Fetched & Processed Context Variables (`${renderedHtml?no_esc!}`)**: Use when content payload requires data transformation, aggregation, or complex decision-making before rendering.
    - **When to use**: Complex Conditional Logic requiring entitlements/API data, Data Transformation (e.g., Markdown to HTML, injecting dynamic variables), or Bulk Processing for performance optimization (batching database lookups for lists).
    - **Benefits**: Adheres to MVC principles by keeping complex logic in Groovy; allows safe output of validated HTML.
4. **Widget Rendering**: For pure UI composition, prefer the native OFBiz widget tags. Use `<content content-id="MY_CONTENT_ID"/>` in Screen Widgets, or the `<display-entity>`/`<content>` field types in Form Widgets.

## Guardrails
- **HTTP vs HTTPS and Relative URLs**:
  - For internal images or documents (`LOCAL_FILE`), **ALWAYS** use OFBiz-relative paths (e.g., `/images/products/myproduct.jpg`). Never hardcode `http://localhost:8080/` or absolute server paths.
  - For external URLs (`URL_RESOURCE`), **ALWAYS** use `https://` to prevent "Mixed Content" security warnings on secure checkout or account pages. Never use HTTP unless explicitly told the target server does not support SSL.
- **Naming**: Use clear, descriptive names for `contentId`.
- **Status**: Always set `statusId` (e.g., `CTNT_PUBLISHED`) to ensure visibility if logic filters by status.
- **The `ElectronicText` Rule**: If `dataResourceTypeId="ELECTRONIC_TEXT"`, you **must** also create an `<ElectronicText>` associated record or you will get silent blank values.
- **The `fromDate` Requirement**: Every associative entity (`ProductContent`, `ContentAssoc`, `CategoryContent`, etc.) requires a `fromDate` for its Primary Key. 
- **Caching Required**: Content fetching requires multiple database hops. ALWAYS invoke caching signatures (`useCache=true` in `ContentWorker` or rely on Wrappers which cache by default) unless explicitly rendering highly dynamic templates, to prevent severe performance degradation.

## Anti-Patterns
- **🚨 Direct ElectronicText Querying**: Writing an `EntityQuery` directly against the `ElectronicText` entity to fetch text. This skips locale fallback, mime-type conversion, and caching entirely.
- **🚨 Missing `ElectronicText`**: Creating a `DataResource` with type `ELECTRONIC_TEXT` but omitting the `<ElectronicText>` tag.
- **🚨 Hardcoded Absolute/HTTP Paths**: Using `dataResourceTypeId="LOCAL_FILE"` with an absolute OS path (`/var/www/ofbiz/...`) or hardcoding `http://...`.
- **🚨 Missing `fromDate` in Associations**: Omitting the `fromDate` on a `<ProductContent>` record, resulting in instant SQL constraint violations.
- **🚨 FTL Macros in Heavy Loops**: Using `<@ofbizContent>` inside heavy loops when determining the contentId requires complex FreeMarker logic or repeated service calls (e.g., `dispatcher.runSync` in FTL).
- **🚨 Presentation HTML in Groovy**: Stitching together pure presentation HTML in Groovy (like `String html = "<div><h2>" + name + "</h2></div>";`) when standard FTL templates can achieve the same result. Groovy should only prepare the data (or data-driven HTML), while FTL controls the layout.

## Examples
### Example: Data Management (Image and Electronic Text)
```xml
<entity-engine-xml>
    <!-- 1. Data Resource pointing to an internal file -->
    <DataResource dataResourceId="MY_PROD_IMAGE" dataResourceTypeId="LOCAL_FILE" 
                  objectInfo="/images/products/myproduct-large.jpg" statusId="DR_AVAILABLE"/>
    <!-- Content metadata linking to the resource -->
    <Content contentId="MY_PROD_IMG_CNT" contentTypeId="DOCUMENT" 
             dataResourceId="MY_PROD_IMAGE" statusId="CTNT_PUBLISHED" contentName="Large Image"/>
    <!-- Association -->
    <ProductContent productId="PROD_1001" contentId="MY_PROD_IMG_CNT" 
                    productContentTypeId="LARGE_IMAGE" fromDate="2026-01-01 00:00:00.000"/>

    <!-- 2. Electronic Text -->
    <DataResource dataResourceId="MY_PROD_DESC" dataResourceTypeId="ELECTRONIC_TEXT"/>
    <ElectronicText dataResourceId="MY_PROD_DESC" textData="Detailed product description..."/>
    <Content contentId="MY_PROD_DESC_CNT" contentTypeId="DOCUMENT" 
             dataResourceId="MY_PROD_DESC" statusId="CTNT_PUBLISHED"/>
</entity-engine-xml>
```

### Example: Groovy Fetching (Data Prep & Wrappers)
```groovy
// 1. Using Wrapper (Best for Entity-Associated Content)
import org.apache.ofbiz.content.product.ProductContentWrapper

ProductContentWrapper pcw = new ProductContentWrapper(product, request)
context.longDescription = pcw.get("LONG_DESCRIPTION", "html") // Add to context for FTL

// 2. Pre-processing complex content or aggregation before rendering
import org.apache.ofbiz.content.content.ContentWorker

Map templateContext = [userLogin: userLogin, request: request, customVar: "Dynamic Value"]
String rawHtml = ContentWorker.renderContentAsText(dispatcher, "COMPLEX_CONTENT_ID", templateContext, locale, "text/html", true)

// ... complex transformation, sanitization, or markdown parsing here ...
String finalProcessedHtml = rawHtml.replace("{{dynamic_tag}}", "computed_value")

context.renderedHtml = finalProcessedHtml // Add to context for FTL (?no_esc!)
```

### Example: FTL Rendering Strategies
```ftl
<!-- 1. Wrappers: Localized, entity-coupled data -->
<div class="product-header">
    <h2>${productContentWrapper.get("PRODUCT_NAME", "html")!}</h2>
</div>

<!-- 2. Directives: Generic, independent CMS blocks & URLs -->
<div class="generic-content">
    <img src="<@ofbizContentUrl contentId='MY_PROD_IMG_CNT'/>" alt="Product Image">
    <@ofbizContent contentId="MY_PROD_DESC_CNT"/>
</div>

<!-- 3. Groovy-Fetched Context: Complex pre-processed HTML -->
<div class="custom-html-content">
    <!-- Must use ?no_esc! to output validated HTML tags without escaping -->
    ${renderedHtml?no_esc!}
</div>
```

### Example: Screen & Form Widget Rendering
```xml
<!-- Screen Widget: Native rendering of content through widget definition -->
<screen name="ViewProductContent">
    <section>
        <widgets>
            <!-- Native rendering by ID -->
            <content content-id="MY_PROD_DESC_CNT"/>
            
            <!-- Render a string pre-fetched in Groovy from Context -->
            <label text="${renderedHtml}" type="html"/>
        </widgets>
    </section>
</screen>

<!-- Form Widget: Rendering pre-fetched content in a field -->
<form name="ViewProductForm" type="single">
    <field name="customContentInfo">
        <display description="${renderedHtml}" type="html"/>
    </field>
</form>
```
