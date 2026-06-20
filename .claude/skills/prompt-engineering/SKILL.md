---
name: prompt-engineering
description: Getting more out of an LLM — structured system prompts, chain-of-thought, few-shot examples, tool descriptions, and templates
---

# Prompt engineering

## A structured system prompt

```
You are a senior code reviewer. Your role is to analyze pull requests for:
1. Correctness - logic errors, edge cases, off-by-one errors
2. Security - injection, authentication, data exposure
3. Performance - N+1 queries, unnecessary allocations, missing indexes
4. Maintainability - naming, complexity, test coverage

For each issue found, respond with:
- Severity: critical | warning | suggestion
- File and line reference
- What is wrong
- How to fix it (with code snippet)

If the code is well-written, say so briefly. Do not invent problems.
```

Give a system prompt a role, a scope, an output format, and its constraints — and spell out what the model should *not* do.

## Chain-of-thought

```
Analyze this database query for performance issues.

Think step by step:
1. Identify the tables and joins involved
2. Check if appropriate indexes exist for the WHERE and JOIN conditions
3. Look for full table scans or cartesian products
4. Estimate the row count at each step
5. Suggest specific index creation or query restructuring

Query:
SELECT o.*, u.name, p.title
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN products p ON o.product_id = p.id
WHERE o.created_at > '2024-01-01'
AND u.country = 'US'
ORDER BY o.created_at DESC
LIMIT 50;
```

Asking for intermediate steps pushes the model to reason explicitly, which lifts accuracy on multi-step problems.

## Few-shot examples

```
Convert natural language to SQL. Follow these examples:

Input: "How many orders were placed last month?"
Output: SELECT COUNT(*) FROM orders WHERE created_at >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month') AND created_at < DATE_TRUNC('month', CURRENT_DATE);

Input: "Top 5 customers by total spending"
Output: SELECT customer_id, SUM(total_amount) AS total_spent FROM orders GROUP BY customer_id ORDER BY total_spent DESC LIMIT 5;

Input: "Products that have never been ordered"
Output: SELECT p.* FROM products p LEFT JOIN order_items oi ON p.id = oi.product_id WHERE oi.id IS NULL;

Now convert:
Input: "Average order value per country for the last quarter"
```

Offer three to five varied examples that pin down the format and cover the edge cases.

## Describing tools

```json
{
  "tools": [
    {
      "name": "search_codebase",
      "description": "Search for code patterns across the repository. Use when you need to find implementations, usages, or definitions.",
      "parameters": {
        "type": "object",
        "properties": {
          "query": {
            "type": "string",
            "description": "Regex pattern or keyword to search for"
          },
          "file_type": {
            "type": "string",
            "description": "File extension filter (e.g., 'ts', 'py')"
          }
        },
        "required": ["query"]
      }
    }
  ]
}
```

A good tool description says *when* to reach for the tool, not merely what it does.

## Templating prompts

```python
def build_review_prompt(diff: str, context: str, rules: list[str]) -> str:
    rules_text = "\n".join(f"- {rule}" for rule in rules)

    return f"""Review this code diff against the following rules:
{rules_text}

Context about the codebase:
{context}

Diff to review:
```
{diff}
```

Respond with a JSON array of findings. If no issues, return an empty array.
Each finding: {{"severity": "critical|warning|info", "line": number, "message": "string", "suggestion": "string"}}"""
```

## What to avoid

- Hand-waving instructions like "be helpful" or "do your best"
- Asking for "creativity" when what you need is deterministic output
- Leaving the output format unstated (JSON, markdown, plain text)
- Cramming several unrelated tasks into one prompt
- Stating "don't do X" without saying what to do instead
- Never stress-testing the prompt with adversarial or edge-case inputs

## Before you ship

- [ ] The system prompt sets role, scope, format, and constraints
- [ ] Multi-step reasoning tasks use chain-of-thought
- [ ] Few-shot examples cover both the common and the edge cases
- [ ] The output format is stated explicitly (JSON schema, markdown, …)
- [ ] Tool descriptions explain when and why to use each one
- [ ] Prompts have been tried against adversarial inputs
- [ ] Temperature and top_p suit the task
- [ ] Templates are parameterized, not hardcoded strings
