---
name: account-mapping
description: Use when managing Douyin account mapping between account_id and homepage_url for monitoring pipelines, including lookup and auto-create on first seen accounts.
metadata:
  openclaw:
    emoji: 📍
    requires:
      bins: []
---

# Account Mapping

Manage `account_id -> homepage_url` mapping in SQLite.

## Contract

- `lookup_or_create_account(homepage_url, account_id, account_name)`
- `get_account_by_homepage(homepage_url)`

## Storage Schema

`account_mapping(homepage_url PRIMARY KEY, account_id, account_name, created_at, updated_at)`
