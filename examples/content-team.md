# Content Team: Research + Outline + Write + Edit + SEO

A five-stage pipeline for producing long-form, publication-ready content. Each stage consumes the previous stage's output. The pipeline produces a final article with SEO metadata.

---

## Team Structure

```
Stage 1: Researcher          (sequential - first to run)
    |
    v
Stage 2: Outliner            (sequential - needs researcher output)
    |
    v
Stage 3: Writer              (sequential - needs outliner output)
    |
    +-- (parallel starts here)
    |
    |-- Stage 4a: Editor     (parallel - both receive writer output)
    |-- Stage 4b: SEO Agent  (parallel - both receive writer + outline output)
    |
    v (after both 4a and 4b complete)
Stage 5: Final Publisher     (sequential - merges editor + SEO outputs)
    |
    v
Publication-ready article
```

**Optimization note:** Editor and SEO can run in parallel because the editor improves prose while the SEO agent works on metadata and keyword strategy. These are independent concerns on the same draft.

---

## Input: Content Brief

```json
{
  "title_direction": "How AI is changing software architecture decisions",
  "audience": "CTOs and senior architects at mid-size companies",
  "target_length": 1800,
  "publication": "Technical blog",
  "tone": "Authoritative but not academic. Direct. Evidence-based.",
  "key_message": "AI is shifting architecture from performance-first to reliability-of-AI-output-first",
  "keywords_to_target": ["AI software architecture", "LLM architecture patterns", "AI system design"],
  "avoid": ["hype", "predictions without evidence", "vendor names as examples"]
}
```

---

## Stage 1: Task Call - Researcher

```
description: "Content researcher - gather material for AI architecture article"

prompt: |
  You are a Content Researcher agent.

  ## Your Job
  Gather current, factual information to support a long-form article on how AI (specifically LLMs) is changing software architecture decisions. Your research becomes the raw material that the writing pipeline will use.

  ## Article Direction
  Title direction: How AI is changing software architecture decisions
  Audience: CTOs and senior architects at mid-size companies
  Key message to support: AI is shifting architecture from performance-first to reliability-of-AI-output-first

  ## Tools You May Use
  WebSearch

  ## Research Angles to Cover
  Run searches for each:
  1. Concrete architecture changes companies are making for AI (vector databases, orchestration layers, etc.)
  2. Statistics on AI system reliability, failure rates, hallucination rates in production
  3. New architecture patterns emerging specifically for LLM-based systems
  4. Expert perspectives from engineers/architects on lessons learned
  5. Contrast: pre-AI architecture priorities vs. post-AI architecture priorities

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "researcher",
    "result": {
      "facts": [
        {"claim": "string", "source": "string", "relevance": "high|medium|low"}
      ],
      "architecture_patterns": [
        {"pattern": "string", "why_emerging": "string", "example_context": "string"}
      ],
      "quotes_or_perspectives": [
        {"perspective": "string", "attribution": "string"}
      ],
      "statistics": [
        {"stat": "string", "source": "string"}
      ],
      "contrasts": [
        {"before_ai": "string", "after_ai": "string"}
      ],
      "research_quality": "high|medium|low",
      "gaps": ["string"]
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Stage 2: Task Call - Outliner (receives Researcher output)

```
description: "Content outliner - build article structure"

prompt: |
  You are a Content Outliner agent.

  ## Your Job
  Build a logical, compelling article structure that will guide a writer to produce an 1800-word article. The outline must flow naturally, use the research provided, and avoid a listicle structure (this is a proper article, not a numbered list post).

  ## Content Brief
  - Audience: CTOs and senior architects at mid-size companies
  - Target length: 1800 words
  - Tone: Authoritative but not academic. Direct. Evidence-based.
  - Key message: AI is shifting architecture from performance-first to reliability-of-AI-output-first

  ## Research to Build From
  [INSERT researcher_output["result"] as JSON here]

  ## Outline Requirements
  - 5-7 sections (not too many; this is an article, not a textbook chapter)
  - Opening section must not start with definitions or history (open with a problem or observation)
  - Each section has a working title and 3-4 specific points to cover (taken from the research)
  - Closing section must be prescriptive (what should the reader do differently after reading this)
  - Flag which research facts/stats go in which section

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "outliner",
    "result": {
      "working_title": "string",
      "sections": [
        {
          "section_number": number,
          "working_title": "string",
          "points_to_cover": ["string", ...],
          "research_to_use": ["quote or fact from research", ...],
          "estimated_words": number,
          "section_purpose": "hook|context|problem|solution|case-study|prescription"
        }
      ],
      "total_estimated_words": number,
      "narrative_thread": "string (the through-line connecting all sections)"
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Stage 3: Task Call - Writer (receives Outliner + Researcher output)

```
description: "Content writer - draft the full article"

prompt: |
  You are a Content Writer agent.

  ## Your Job
  Write a complete, polished first draft of the article. Follow the outline exactly. Use the research provided. Write for CTOs and senior architects who have built real systems and have limited patience for vague claims.

  ## Writing Rules
  - Never use the word "revolutionary", "game-changing", "paradigm", "transformative"
  - No vendor name-dropping as evidence (say "a major cloud provider" not "AWS")
  - Every claim needs either research support or qualifies itself ("in most cases", "for teams without...")
  - Paragraphs max 4 sentences
  - Transitions between sections must feel earned, not formulaic
  - The opening sentence of the article must be specific and arresting, not a question or generic statement

  ## Tone: Authoritative but not academic. Direct. Evidence-based.

  ## Outline to Follow
  [INSERT outliner_output["result"] as JSON here]

  ## Research Available
  [INSERT researcher_output["result"] as JSON here]

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "writer",
    "result": {
      "title": "string",
      "sections": [
        {
          "section_title": "string",
          "content": "string (full prose)"
        }
      ],
      "word_count": number,
      "outline_adherence": "full|mostly|partial",
      "research_used": ["string identifiers of research claims used"]
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Stage 4a: Task Call - Editor (parallel, receives Writer output)

```
description: "Content editor - improve draft quality and flow"

prompt: |
  You are a Content Editor agent.

  ## Your Job
  Edit the article draft for quality, clarity, and impact. You are line-editing, not rewriting. Improve weak sentences. Cut redundancy. Sharpen transitions. Do not change the argument structure or add new content.

  ## Article Draft to Edit
  [INSERT writer_output["result"] as JSON here]

  ## Editing Priorities (in order)
  1. Opening sentence: must be immediately specific and compelling
  2. Section transitions: each section should flow from the last without a mechanical bridge phrase
  3. Sentence-level: eliminate passive voice where it weakens the point
  4. Cut: any sentence that repeats information covered elsewhere
  5. Closing: must be a clear, memorable prescription, not a summary

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "editor",
    "result": {
      "title": "string",
      "sections": [
        {
          "section_title": "string",
          "content": "string (edited prose)"
        }
      ],
      "final_word_count": number,
      "edits_log": [
        {"section": "string", "edit_type": "cut|rewrite|strengthen|transition", "summary": "string"}
      ],
      "quality_score": number
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Stage 4b: Task Call - SEO Agent (parallel, receives Writer + Outliner output)

```
description: "SEO agent - optimize metadata and keyword strategy"

prompt: |
  You are an SEO Specialist agent.

  ## Your Job
  Analyze the article draft and produce SEO metadata, keyword optimization recommendations, and structured data that will maximize search visibility for the target keywords.

  ## Target Keywords
  Primary: "AI software architecture"
  Secondary: "LLM architecture patterns", "AI system design"

  ## Article Draft
  [INSERT writer_output["result"] as JSON here]

  ## Article Outline (for context on structure)
  [INSERT outliner_output["result"] as JSON here]

  ## SEO Requirements
  - Title tag (60 chars max, primary keyword near front)
  - Meta description (155 chars max, includes primary keyword, has call to action)
  - H1 (article title - can differ from title tag)
  - Suggested URL slug
  - Internal linking opportunities (describe what type of content to link from/to)
  - Keyword density analysis (are target keywords used naturally, over-used, or under-used?)
  - Content gaps for SEO (topics that competing articles likely cover that this one doesn't)

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "seo-specialist",
    "result": {
      "title_tag": "string (max 60 chars)",
      "meta_description": "string (max 155 chars)",
      "h1": "string",
      "url_slug": "string",
      "keyword_analysis": {
        "primary_keyword_count": number,
        "secondary_keyword_counts": {"keyword": number},
        "density_assessment": "under-optimized|good|over-optimized",
        "recommendation": "string"
      },
      "internal_linking": [
        {"link_from": "string description", "link_to": "string description", "anchor_text_suggestion": "string"}
      ],
      "content_gaps": ["string"],
      "schema_markup_type": "Article|BlogPosting|TechArticle",
      "estimated_search_intent": "informational|commercial|navigational"
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Stage 5: Final Publisher (receives Editor + SEO outputs)

```python
import json

editor_output = json.loads(editor_result)["result"]
seo_output = json.loads(seo_result)["result"]

# Assemble final article with metadata
sections_text = []
for section in editor_output["sections"]:
    sections_text.append(f"## {section['section_title']}\n\n{section['content']}")

body = "\n\n".join(sections_text)

final_article = f"""---
title: "{seo_output['title_tag']}"
description: "{seo_output['meta_description']}"
slug: "{seo_output['url_slug']}"
schema: "{seo_output['schema_markup_type']}"
---

# {editor_output['title']}

{body}
"""

# Write final article
with open("/tmp/content-team/final-article.md", "w") as f:
    f.write(final_article)

# Write SEO brief for publisher
seo_brief = {
    "url_slug": seo_output["url_slug"],
    "title_tag": seo_output["title_tag"],
    "meta_description": seo_output["meta_description"],
    "h1": seo_output["h1"],
    "keyword_status": seo_output["keyword_analysis"]["density_assessment"],
    "content_gaps": seo_output["content_gaps"],
    "internal_linking_opportunities": seo_output["internal_linking"],
    "final_word_count": editor_output["final_word_count"],
    "quality_score": editor_output["quality_score"]
}

with open("/tmp/content-team/seo-brief.json", "w") as f:
    f.write(json.dumps(seo_brief, indent=2))

print(f"Content pipeline complete.")
print(f"Article: /tmp/content-team/final-article.md ({editor_output['final_word_count']} words)")
print(f"SEO brief: /tmp/content-team/seo-brief.json")
print(f"Quality score: {editor_output['quality_score']}/10")
print(f"Content gaps to address in future articles: {len(seo_output['content_gaps'])}")
```

---

## Adapting This Template

- Change the content brief at the top (audience, tone, topic, keywords)
- Keep the pipeline structure. Do not parallelize stages that need prior output.
- The Editor and SEO agent parallelization is a key efficiency win - preserve it.
- For shorter content (under 800 words), merge the Outliner into the Writer's prompt to reduce stages.
- For longer content (over 3000 words), split the Writer into multiple agents (one per major section) and add a second editorial pass.
