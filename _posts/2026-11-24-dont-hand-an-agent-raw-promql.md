---
layout: post
title: "Don't hand an agent raw PromQL"
date: 2026-11-24
tags: [ai-agents, mcp, prometheus, go, observability]
---

First version of our Prometheus MCP server had one tool. `query_range`, takes a PromQL string, start, end, step, passes it straight through. Took an afternoon. Agents loved it.

Then you watch what an agent actually sends when someone asks "why was checkout slow last month". Thirty days. Fifteen second step. A selector with no metric name and a regex on the namespace. It isn't being malicious, it's being thorough. Prometheus doesn't care about intent though. That query either times out, eats a chunk of memory on the box everybody's dashboards and alerts depend on, or comes back as a wall of series the model can't fit in context anyway.

A human writing PromQL has a feel for what's expensive because they've been burned. The model hasn't. So the server has to know for it.

## the rules I ended up with

All of these live in the MCP server, before anything hits Prometheus:

- cap the time range per call, and the range inside any `[...]` selector
- derive a minimum step from the range so a call can't ask for thousands of points
- every selector needs a metric name, no `{__name__=~".+"}` fishing
- hard timeout on the request
- cap how many series come back, and say so when you cut

None of that is clever. The point is it runs on every call, and the agent can't talk its way past it.

## parse it, don't regex it

First attempt at "no nameless selectors" was a regex on the query string. Lasted about a day. PromQL has enough syntax that string matching on it is a losing game. Use the real parser and walk the tree:

```go
import (
	"errors"
	"fmt"
	"time"

	"github.com/prometheus/prometheus/promql/parser"
)

const (
	maxRange     = 6 * time.Hour
	maxPoints    = 500
	maxSeries    = 200
	queryTimeout = 10 * time.Second
)

func checkQuery(q string) error {
	expr, err := parser.ParseExpr(q)
	if err != nil {
		return fmt.Errorf("invalid PromQL: %w", err)
	}
	var bad error
	parser.Inspect(expr, func(n parser.Node, _ []parser.Node) error {
		switch v := n.(type) {
		case *parser.VectorSelector:
			// empty Name means it's {__name__=~...} or labels only
			if v.Name == "" {
				bad = errors.New("every selector needs a metric name, e.g. http_requests_total{namespace=\"x\"}")
			}
		case *parser.MatrixSelector:
			if v.Range > maxRange {
				bad = fmt.Errorf("range [%s] is too wide, max is %s. use a shorter window inside rate()", v.Range, maxRange)
			}
		case *parser.SubqueryExpr:
			if v.Range > maxRange {
				bad = fmt.Errorf("subquery range %s is too wide, max is %s", v.Range, maxRange)
			}
		}
		return bad // non-nil stops the walk
	})
	return bad
}
```

Heads up, the parser package moves around between Prometheus releases. Some versions want you to go through a parser options struct instead of the package level `ParseExpr`. Check against whatever version you vendor. The idea doesn't change.

## clamp, don't reject, on time

Range and step I clamp instead of erroring. If the agent asks for 30 days I give it the last six hours and tell it that's what happened. Rejecting just makes it retry with 29 days.

```go
func clamp(start, end time.Time, step time.Duration) (time.Time, time.Duration, string) {
	note := ""
	if end.Sub(start) > maxRange {
		start = end.Add(-maxRange)
		note = fmt.Sprintf("range clamped to last %s. ", maxRange)
	}
	minStep := end.Sub(start) / maxPoints
	if step < minStep {
		step = minStep.Round(time.Second)
		note += fmt.Sprintf("step raised to %s. ", step)
	}
	return start, step, note
}
```

Prometheus itself refuses a range query over 11,000 points per series, so there's already a ceiling. It's just way too high for something that ends up as tokens. 500 points is plenty for a model to see a shape.

Then the call itself gets a context with `queryTimeout`. The `v1.API` client in `client_golang` takes the context, and newer versions also have a `WithTimeout` option that passes the timeout through to Prometheus so the server gives up too, not just your client. Check yours has it.

## the error message is the prompt

This is the part I didn't expect. The text you return when you block something is basically the most important prompt in the whole system. "query rejected" gets you a retry of the same query, slightly reworded. "range [30d] is too wide, max is 6h. use a shorter window inside rate()" gets you a correct query on the next turn, almost every time.

Same with truncation. When there are more series than `maxSeries` I sort by last value, keep the top ones, and append a line saying how many were dropped and to add a `sum by (...)` or `topk` if it needs the rest. The model reads that and fixes its own query. Silently truncating is worse than failing, because then it reasons confidently about half the data.

## Prometheus still needs its own limits

The MCP checks are the first line. They're not the only one. Somebody will eventually point some other tool at the same endpoint. So Prometheus gets `--query.timeout`, `--query.max-samples` and `--query.max-concurrency` set to numbers that protect it from everybody, agents included. And if you can, point the MCP server at a read replica or a separate query frontend, not the instance your alerting runs on. An agent investigation should never be able to delay a page.

Couple of smaller things:

- read only. The server only exposes query, labels and series endpoints. No admin API, no delete series, obviously.
- log every query with which agent and which user asked. Same reasoning as auditing what coding agents read. When something odd shows up on the Prometheus side you want to tie it back to a conversation.
- give it a `list_metrics` tool with a name filter. Half the bad queries were the model guessing metric names. Let it look them up instead.

The agent's not the problem. The tool you gave it is. If one tool call can hurt the thing that pages you at night, that's on whoever wrote the tool.
