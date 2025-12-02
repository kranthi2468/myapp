Here is the **Complete, Unabridged v20.0 Technical Design Document**.

It includes:

1.  **Full Database Schemas** (Every field defined).
2.  **All 7 APIs** detailed with Logic, Inputs, and Outputs.
3.  **Exact Corpus Construction Logic** (Code-level string building).
4.  **Final Meta-Prompts** (Full text).

-----

# **GEO Discovery Pipeline (v20.0) - Technical Design Document**

| **Document Version** | 20.0 |
| :--- | :--- |
| **Status** | Final |
| **Module** | `GeoDiscoveryModule` |

## **1.0 Architecture Overview**

### **1.1 Design Philosophy**

This module implements a **7-step atomic pipeline** to generate market-validated Search Topics and User Prompts. It follows a **Waterfall Data Model**: every API step generates a specific data artifact (stored in MongoDB) that serves as the input for the next step.

### **1.2 Architectural Flow**

1.  **API 1: `fetch-internal-data`** (Shopify GraphQL/Sitemap) -\> `shopify_raw_data`
2.  **API 2: `fetch-google-ads-data`** (Google Ads SiteSeed) -\> `google_ads_runs`
3.  **API 3: `prepare-strategy-corpus`** (Merges Internal + Ads Data) -\> `corpus_data`
4.  **API 4: `generate-strategy`** (**LLM 1**: Profile + Topics) -\> `strategy_runs`
5.  **API 5: `fetch-serp-context`** (SerpApi for Champions) -\> `serp_data_runs`
6.  **API 6: `prepare-final-corpus`** (Merges Strategy + SERP + Products) -\> `corpus_data`
7.  **API 7: `generate-topics-and-prompts`** (**LLM 2**: Final Output) -\> `topics_and_prompts`

-----

## **2.0 Database Schema (Full Definitions)**

**1. `shopify_raw_data`**

  * **Purpose:** Stores the raw snapshot of the merchant's inventory and policies.
  * **PK:** `_id` (String = `base_domain`)

<!-- end list -->

```typescript
interface ShopifyRawData {
  _id: string; // e.g., "koskii.com"
  base_domain: string; // "koskii.com"
  sitemap_collections: string[]; // List of all collection titles from XML
  top_products: Array<{
    title: string;
    description: string; // Plain text description
    priceRange: {
      minVariantPrice: { amount: string; currencyCode: string };
    };
    tags: string[];
    productType: string;
  }>;
  shop_policies: {
    name: string;
    description: string; // Meta description
    paymentSettings: {
      acceptedCardBrands: string[];
      supportedDigitalWallets: string[];
    };
    shippingPolicy: { body: string };
    refundPolicy: { body: string };
  };
  created_at: Date;
  updated_at: Date;
}
```

**2. `google_ads_runs`**

  * **Purpose:** Stores raw keyword data from Google Ads API.
  * **PK:** `_id` (ObjectId)
  * **Unique Key:** `ads_run_slug`

<!-- end list -->

```typescript
interface GoogleAdsRun {
  _id: ObjectId;
  base_domain: string;
  ads_run_slug: string; // e.g., "koskii.com_ads_1715000"
  raw_keywords: Record<string, number>; // Map: { "keyword": volume }
  total_keywords_fetched: number;
  created_at: Date;
}
```

**3. `corpus_data`**

  * **Purpose:** Stores the constructed text blocks sent to LLMs. Used for both Strategy and Final calls.
  * **PK:** `_id` (ObjectId)
  * **Unique Key:** `corpus_slug`

<!-- end list -->

```typescript
interface CorpusData {
  _id: ObjectId;
  base_domain: string;
  corpus_slug: string; // e.g., "koskii.com_strategy_corpus_1715000"
  type: 'STRATEGY_GENERATION' | 'FINAL_GENERATION';
  corpus_text: string; // The full, formatted string
  source_refs: {
    ads_run_slug?: string;
    strategy_slug?: string;
    serp_run_slug?: string;
  };
  created_at: Date;
}
```

**4. `strategy_runs`**

  * **Purpose:** Output of LLM Call 1 (Merchant Profile & Topic Strategy).
  * **PK:** `_id` (ObjectId)
  * **Unique Key:** `strategy_slug`

<!-- end list -->

```typescript
interface StrategyRun {
  _id: ObjectId;
  base_domain: string;
  strategy_slug: string; // e.g., "koskii.com_strategy_1715000"
  corpus_slug: string; // FK to corpus_data
  model_name: string;
  merchant_summary: string; // Human readable summary
  merchant_profile: {
    primary_categories: string[];
    target_audience: string;
    price_range_summary: string;
    key_attributes: string[];
    business_model_facts: string[];
  };
  topics: Array<{
    topic_text: string;
    rationale: string;
    champion_keywords: string[]; // 1-3 keywords
  }>;
  usage: { prompt_tokens: number; completion_tokens: number };
  cost: number;
  created_at: Date;
}
```

**5. `serp_data_runs`**

  * **Purpose:** Stores PAA and Discussion data fetched from SerpApi.
  * **PK:** `_id` (ObjectId)
  * **Unique Key:** `serp_run_slug`

<!-- end list -->

```typescript
interface SerpDataRun {
  _id: ObjectId;
  base_domain: string;
  serp_run_slug: string; // e.g., "koskii.com_serp_1715000"
  strategy_slug: string; // FK to strategy_runs
  serp_data: Record<string, { // Key is the champion keyword
    paa: string[]; // Related Questions
    discussions: Array<{ title: string; snippet: string }>;
  }>;
  created_at: Date;
}
```

**6. `topics_and_prompts`**

  * **Purpose:** Final Output of LLM Call 2.
  * **PK:** `_id` (ObjectId)
  * **Unique Key:** `generation_slug`

<!-- end list -->

```typescript
interface TopicsAndPrompts {
  _id: ObjectId;
  base_domain: string;
  generation_slug: string; // e.g., "koskii.com_final_gen_1715000"
  corpus_slug: string; // FK to corpus_data (Final Corpus)
  model_name: string;
  final_output: {
    topics: Array<{
      topic_text: string;
      prompts: string[];
    }>;
  };
  usage: { prompt_tokens: number; completion_tokens: number };
  cost: number;
  created_at: Date;
}
```

-----

## **3.0 API Specifications (Logic & Contracts)**

#### **API 1: `POST /fetch-internal-data`**

  * **Service:** `ShopifyDataService`
  * **DTO:** `{ base_domain: string, access_token: string }`
  * **Logic:**
    1.  Fetch `https://{base_domain}/sitemap_collections.xml`. Parse XML to get a list of Collection Titles.
    2.  Call Shopify Admin GraphQL (`SHOPIFY_PRODUCTS_QUERY`) to get the **Top 200 `BEST_SELLING` products**.
    3.  Call Shopify Admin GraphQL (`SHOPIFY_SHOP_QUERY`) to get policies.
    4.  Construct the `ShopifyRawData` object.
    5.  **Upsert** into `shopify_raw_data` collection: `{ _id: base_domain }` -\> `$set: data`.
  * **Output:** `{ status: "success", base_domain: "..." }`

#### **API 2: `POST /fetch-google-ads-data`**

  * **Service:** `MarketDataService`
  * **DTO:** `{ base_domain: string }`
  * **Logic:**
    1.  Call Google Ads API (`GenerateKeywordIdeas`) using `siteSeed = { site: base_domain }`.
    2.  Receive \~2,000-3,000 keywords.
    3.  **Filter (Code):** Remove items where `avg_monthly_searches < 20`.
    4.  **Filter (Code):** Remove items where keyword contains the `base_domain` string (e.g., "koskii").
    5.  Generate `ads_run_slug` = `${base_domain}_ads_${timestamp}`.
    6.  Save to `google_ads_runs`.
  * **Output:** `{ ads_run_slug: "..." }`

#### **API 3: `POST /prepare-strategy-corpus`**

  * **Service:** `CorpusService`
  * **DTO:** `{ base_domain: string, ads_run_slug: string }`
  * **Logic:**
    1.  Fetch `shopify_raw_data` by `base_domain`.
    2.  Fetch `google_ads_runs` by `ads_run_slug`.
    3.  **Build Corpus String (See Section 4.1).**
    4.  Generate `corpus_slug` = `${base_domain}_strategy_corpus_${timestamp}`.
    5.  Save to `corpus_data` with `type: "STRATEGY_GENERATION"`.
  * **Output:** `{ corpus_slug: "..." }`

#### **API 4: `POST /generate-strategy`**

  * **Service:** `GeoAiService`
  * **DTO:** `{ base_domain: string, corpus_slug: string, model_name: string }`
  * **Logic:**
    1.  Fetch `corpus_text` from `corpus_data`.
    2.  Call LLM (`model_name`) with `STRATEGY_GENERATION_PROMPT` + `corpus_text`.
    3.  Generate `strategy_slug` = `${base_domain}_strategy_${timestamp}`.
    4.  Calculate Cost.
    5.  Save to `strategy_runs`.
  * **Output:** `{ strategy_slug: "..." }`

#### **API 5: `POST /fetch-serp-context`**

  * **Service:** `MarketDataService`
  * **DTO:** `{ base_domain: string, strategy_slug: string }`
  * **Logic:**
    1.  Fetch `strategy_runs`. Extract all `champion_keywords` from the `topics` array.
    2.  Deduplicate keywords (e.g., 15-20 unique keywords).
    3.  **Loop** through keywords:
          * Call `SerpApi` (Google Search).
          * Extract `related_questions` (PAA) -\> `string[]`.
          * Extract `discussions_and_forums` + `perspectives` -\> `Array<{title, snippet}>`.
    4.  Generate `serp_run_slug` = `${base_domain}_serp_${timestamp}`.
    5.  Save to `serp_data_runs`.
  * **Output:** `{ serp_run_slug: "..." }`

#### **API 6: `POST /prepare-final-corpus`**

  * **Service:** `CorpusService`
  * **DTO:** `{ base_domain: string, strategy_slug: string, serp_run_slug: string }`
  * **Logic:**
    1.  Fetch `strategy_runs` (Profile/Summary/Topics).
    2.  Fetch `serp_data_runs` (Context).
    3.  **Build Corpus String (See Section 4.2).**
    4.  Generate `corpus_slug` = `${base_domain}_final_corpus_${timestamp}`.
    5.  Save to `corpus_data` with `type: "FINAL_GENERATION"`.
  * **Output:** `{ corpus_slug: "..." }`

#### **API 7: `POST /generate-topics-and-prompts`**

  * **Service:** `GeoAiService`
  * **DTO:** `{ base_domain: string, corpus_slug: string, model_name: string }`
  * **Logic:**
    1.  Fetch `corpus_text`.
    2.  Call LLM (`model_name`) with `FINAL_GENERATION_PROMPT` + `corpus_text`.
    3.  Generate `generation_slug` = `${base_domain}_gen_${timestamp}`.
    4.  Calculate Cost.
    5.  Save to `topics_and_prompts`.
  * **Output:** `{ generation_slug: "...", final_output: { ... } }`

-----

## **4.0 Corpus Construction Logic (String Builders)**

This is the exact logic `CorpusService` must implement to build the strings.

#### **4.1 Strategy Corpus (for API 3)**

```javascript
let text = `[INTERNAL_DATA]\n`;
text += `Shop Name: ${shopifyData.shop_policies.name}\n`;
text += `Description: ${shopifyData.shop_policies.description}\n`;
text += `\n[POLICIES]\n`;
text += `Shipping: ${shopifyData.shop_policies.shippingPolicy.body.slice(0, 300)}\n`;
text += `Returns: ${shopifyData.shop_policies.refundPolicy.body.slice(0, 300)}\n`;
text += `Payment: ${shopifyData.shop_policies.paymentSettings.acceptedCardBrands.join(', ')}\n`;

text += `\n[COLLECTION_LIST] (Total: ${shopifyData.sitemap_collections.length})\n`;
text += shopifyData.sitemap_collections.join('\n');

text += `\n\n[TOP_PRODUCTS] (Sample of Best Sellers)\n`;
shopifyData.top_products.forEach(p => {
   text += `- Title: ${p.title}\n`;
   text += `  Price: ${p.priceRange.minVariantPrice.amount} ${p.priceRange.minVariantPrice.currencyCode}\n`;
   text += `  Tags: ${p.tags.join(', ')}\n`; // Raw tags are fine here for the "Vibe Check"
   text += `  Description: ${p.description.slice(0, 200)}\n\n`;
});

text += `\n[MARKET_DATA] (Keywords sorted by Volume)\n`;
// googleAdsData.raw_keywords is a Map { keyword: volume }
Object.entries(googleAdsData.raw_keywords)
  .sort(([,a], [,b]) => b - a) // Sort Descending
  .forEach(([kw, vol]) => {
     text += `- ${kw}: ${vol}\n`;
  });

return text;
```

#### **4.2 Final Corpus (for API 6)**

```javascript
let text = `[MERCHANT_SUMMARY]\n${strategyRun.merchant_summary}\n\n`;

text += `[MERCHANT_PROFILE]\n`;
text += `- Primary Categories: ${strategyRun.merchant_profile.primary_categories.join(', ')}\n`;
text += `- Target Audience: ${strategyRun.merchant_profile.target_audience}\n`;
text += `- Price Point: ${strategyRun.merchant_profile.price_range_summary}\n`;
text += `- Key Attributes: ${strategyRun.merchant_profile.key_attributes.join(', ')}\n`;
text += `- Business Facts: ${strategyRun.merchant_profile.business_model_facts.join(', ')}\n\n`;

text += `[TOPIC_CONTEXT]\n`;
strategyRun.topics.forEach(topic => {
  text += `Topic: ${topic.topic_text}\n`;
  text += `- Rationale: ${topic.rationale}\n`;
  text += `- Champion Keywords: ${topic.champion_keywords.join(', ')}\n`;
  
  const champions = topic.champion_keywords;
  const paa = [];
  const discussions = [];

  // Aggregate SERP data for this topic's champions
  champions.forEach(kw => {
    if(serpRun.serp_data[kw]) {
      if(serpRun.serp_data[kw].paa) paa.push(...serpRun.serp_data[kw].paa);
      if(serpRun.serp_data[kw].discussions) discussions.push(...serpRun.serp_data[kw].discussions);
    }
  });

  if(paa.length > 0) {
    text += `- PAA Questions:\n  * ${paa.slice(0, 10).join('\n  * ')}\n`;
  }
  if(discussions.length > 0) {
    text += `- Discussions/Sentiment:\n`;
    discussions.slice(0, 5).forEach(d => {
      text += `  * Title: ${d.title}\n    Snippet: ${d.snippet}\n`;
    });
  }
  text += `\n`;
});

return text;
```

-----

## **5.0 Meta-Prompts**

#### **Prompt 1: `STRATEGY_GENERATION_PROMPT`**

```text
[SYSTEM]
You are an expert e-commerce and search marketing strategist. I have provided you with a corpus containing two datasets:
1. [INTERNAL_DATA]: The merchant's raw inventory (top products, prices) and policies.
2. [MARKET_DATA]: A raw list of ~3,000 real search keywords with volumes from Google Ads.

[DEFINITIONS]
1. "Search Topic": A high-level **user intent or occasion** (e.g., "Wedding Guest Outfit", "Home Office Setup"). It must be a broad concept backed by search volume, NOT just a specific product sub-type (e.g., "cotton shirt").
2. "Champion Keyword": The specific, high-volume search query that best represents a topic. We will use this to fetch "People Also Ask" questions later.

[TASK 1: MERCHANT PROFILING]
Analyze the [INTERNAL_DATA] to understand the business. Return a JSON object with:
1. "merchant_summary": A comprehensive 4-6 sentence overview of the business, niche, pricing, and target audience.
2. "merchant_profile": A structured object containing:
   - "primary_categories": (List 5-10 main categories)
   - "target_audience": (e.g., "Budget shoppers", "Luxury brides", "Tech enthusiasts")
   - "price_range_summary": (Infer from product prices, e.g., "Mid-Range (3k-15k INR)")
   - "key_attributes": (List 5-10 features like "Ready-to-wear", "Waterproof", "Organic")
   - "business_model_facts": (e.g., "COD Available", "Free Shipping")

[TASK 2: TOPIC DISCOVERY & CHAMPION SELECTION]
Analyze the [MARKET_DATA]. Your goal is to select **10-15 High-Value Search Topics**.

**Rules for Handling Data:**
1.  **Filtering:** The list contains noise.
    * **Ignore** navigational terms (e.g., "{brand_name} login", "contact number").
    * **Ignore** irrelevant keywords (products the merchant does not sell).
    * **Ignore** absolute garbage (gibberish or low relevance).
2.  **Volume vs. Diversity (The Balance):**
    * **High Volume Clusters:** Identify the massive demand clusters. If a cluster is huge (e.g., "Sarees"), **SPLIT IT** into semantically distinct sub-topics (e.g., "Silk Sarees", "Party Wear Sarees") to capture depth.
    * **Breadth:** Ensure you also select topics for distinct niches (e.g., "Haldi Outfits") even if their volume is lower than the top cluster, provided they are relevant.
3.  **Champion Keywords (1-3 per Topic):**
    * For *each* topic, identify **1 to 3 "Champion Keywords"** from the [MARKET_DATA].
    * These must be high-volume queries that best represent the user's intent for that topic.

[OUTPUT FORMAT]
Return a single, valid JSON object.

---
[EXAMPLE 1: Electronics Store]
[INTERNAL_DATA]: Products: Wireless Earbuds ($40), Waterproof Speakers ($60)...
[MARKET_DATA]: "best earbuds"(50k), "cheap bluetooth speaker"(20k), "waterproof audio"(10k)...
[Output]:
{
  "merchant_summary": "A budget-friendly electronics brand focusing on durable audio gear...",
  "merchant_profile": {
    "primary_categories": ["Audio", "Speakers"],
    "price_range_summary": "Budget (<$100)",
    "key_attributes": ["Waterproof", "Durable"],
    ...
  },
  "topics": [
    {
      "topic_text": "Portable Waterproof Audio",
      "rationale": "High volume cluster matching inventory attributes",
      "champion_keywords": ["waterproof bluetooth speaker", "shower speaker"]
    }
  ]
}
---
[EXAMPLE 2: Fashion Store]
[INTERNAL_DATA]: Products: Bridal Lehengas (25k INR), Silk Sarees (10k INR)...
[MARKET_DATA]: "lehenga for wedding"(50k), "bridal lehenga red"(15k), "saree"(100k)...
[Output]:
{
  "merchant_summary": "A premium ethnic wear brand targeting bridal shoppers...",
  "merchant_profile": {
    "primary_categories": ["Lehengas", "Sarees"],
    "price_range_summary": "Premium (10k-50k INR)",
    "key_attributes": ["Bridal", "Hand-embroidered"],
    ...
  },
  "topics": [
    {
      "topic_text": "Indian Wedding Outfits",
      "rationale": "High volume cluster split from main group",
      "champion_keywords": ["lehenga for wedding", "bridal lehenga design"]
    }
  ]
}
---
[TASK]
[CORPUS]:
{corpus_text}
[Brand Name]: "{brand_name}"

[Output (JSON Only)]
```

#### **Prompt 2: `FINAL_GENERATION_PROMPT`**

```text
[SYSTEM]
You are a GEO strategist. I have provided you with a corpus containing:
1. [MERCHANT_PROFILE] & [MERCHANT_SUMMARY]: The merchant's identity, price point, and constraints.
2. [TOPIC_CONTEXT]: A list of topics. For each topic, I have included the "Champion Keywords" used and the resulting "People Also Ask" (PAA) and "Discussions" data fetched from Google.

[DEFINITIONS]
1. "User Prompt": A **creative, conversational question** a user would ask an AI. It must be **context-aware** (using insights from the SERP data) and **non-formulaic** (avoiding simple templates like "What is X?").

[TASK]
For **each** topic in [TOPIC_CONTEXT]:
1.  Generate **5-7 User Prompts**.
2.  **CRITICAL (Realism):** Prompts must be inspired *directly* by the [PAA] and [DISCUSSIONS] snippets.
    * If users ask about "weight", "price", "itching", or "delivery" in the data, your prompts must reflect those specific concerns.
    * Do not just copy the PAA. Adapt it into a natural prompt (e.g., Change "Which fabric is best?" to "I need a breathable fabric for a summer wedding, what do you suggest?").
3.  **CRITICAL (Relevance):** Ensure prompts align with the [MERCHANT_PROFILE].
    * If the merchant is "Luxury", do not generate "cheap" or "DIY" prompts.
    * If the merchant is "Budget", do not generate "Couture" prompts.
4.  **Variety:** Use the 4 e-commerce intents to ensure diversity:
    * **Informational** ("How to style...", "Difference between...")
    * **Commercial** ("Best X for Y...", "Compare X and Y")
    * **Transactional** ("Where to buy authentic X online...")
    * **Navigational** (Generic only, e.g., "Top rated sites for X...")
5.  **Constraint:** **NO BRAND NAMES.** Prompts must be generic.

[OUTPUT FORMAT]
Return a single, valid JSON object.

---
[EXAMPLE 1: Electronics]
[MERCHANT_PROFILE]: Budget-friendly, Waterproof audio.
[TOPIC]: Waterproof Audio
[PAA]: "Can I swim with IPX7?"
[DISCUSSIONS]: "My earbuds fall out when running."
[Output Prompts]:
- "I'm looking for fully waterproof earbuds for swimming that actually stay in my ears."
- "What is the difference between IPX5 and IPX7 ratings for gym headphones?"
- "Suggest some durable, budget-friendly speakers I can take to the beach."

---
[EXAMPLE 2: Fashion]
[MERCHANT_PROFILE]: Premium, Bridal, Heavy Work.
[TOPIC]: Indian Wedding Outfits
[PAA]: "Is heavy lehenga comfortable?"
[DISCUSSIONS]: "Need a reception outfit that looks grand but isn't too heavy."
[Output Prompts]:
- "I need a grand reception outfit that is comfortable enough to dance in."
- "Can you suggest some premium bridal lehengas that feature intricate hand-embroidery?"
- "What are the latest trends for wedding guest outfits that aren't too heavy?"

---
[TASK]
[CORPUS]:
{corpus_text}

[Output (JSON Only)]
```
