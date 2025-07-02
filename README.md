# Freedonia Vote Synchronization System - Technical Integration Guide

## Overview

The Freedonia Vote Synchronization System (vsync) provides a centralized platform for real-time vote counting and synchronization across all states. This guide explains how to integrate your state's vote counting system with the centralized infrastructure.

## System Architecture

The vote synchronization system consists of two main components:

1. **vsync Service** - Central API server handling vote publishing and subscription
2. **vparse Service** - Vote counting service

### High-Level Flow

```
State Vote System → RSU File → vsync API → vparse Service → vsync API → Centralized Database
                                                      ↓
State Display System ← SSE Stream ← vsync API ← Vote Updates
```

## API Endpoints

### Base URL

```
httpss://sync.synchronic27.labs.canva-internal.dev
```

### Health Check

```
GET /healthz
```

**Response:**

```json
{
  "status": "ok"
}
```

## Publishing Vote Updates

### RSU File Upload

Upload vote data using the RSU (Regional State Update) format.

**Endpoint:**

```
POST /api/votes/publish/{state}
Content-Type: multipart/form-data
```

**Parameters:**

- `state` (path): Your state name (e.g., "North Freedonia")
- `file` (form field): RSU file containing vote data

**Example Request:**

```bash
curl -X POST \
  -F "file=@votes.rsu" \
  "httpss://sync.synchronic27.labs.canva-internal.dev/api/votes/publish/Izzi"
```

**Success Response:**

```json
{
  "status": "RSU processed and updates published",
  "filename": "Izzi_20250618_024249_09b2128b64fee1f5.rsu",
  "records_processed": 5,
  "updates_published": 5,
  "vparse_response": "{\"records\": 5, \"vote_updates\": [...], \"processed_files\": [...]}"
}
```

**Error Response:**

```json
{
  "error": "RSU parsing error: Line 5: invalid state_id 'abc'"
}
```

## Subscribing to Vote Updates

### Server-Sent Events (SSE)

Subscribe to real-time vote updates using Server-Sent Events.

**Endpoint:**

```
GET /api/votes/subscribe/{state}
```

**Parameters:**

- `state` (path): State name to subscribe to

**Example Request:**

```bash
curl -N "httpss://sync.synchronic27.labs.canva-internal.dev/api/votes/subscribe/Izzi"
```

**Response Format:**

```
event: vote-update
data: {"state_id":1,"party_id":1,"votes":1000,"percentage":50.0,"timestamp":"2025-06-18T02:42:49Z","update_source":"RSU Upload: votes.rsu","source_file":"votes.rsu"}

event: vote-update
data: {"state_id":1,"party_id":2,"votes":800,"percentage":40.0,"timestamp":"2025-06-18T02:42:49Z","update_source":"RSU Upload: votes.rsu","source_file":"votes.rsu"}
```

## RSU File Format Specification

### Overview

RSU (Regional State Update) is a text-based format for vote data that supports comments, includes, and flexible formatting.

### Basic Structure

```rsu
# RSU Vote Data File
# Format: state_id,party_id,votes,percentage,update_source

1,1,25000,35.2,District 1 Polling Station
1,2,18000,25.4,District 2 Polling Station
1,3,15000,21.1,District 3 Polling Station
```

### Data Fields

Each data line contains the following fields in order:

1. **state_id** (uint): Unique identifier for the state
2. **party_id** (uint): Unique identifier for the political party
3. **votes** (int): Number of votes received (≥ 0)
4. **percentage** (float): Percentage of total votes (0-100)
5. **update_source** (string, optional): Custom source identifier

### Comments

Lines starting with `#` are treated as comments and ignored:

```rsu
# This is a comment
# RSU Vote Data File
# Format: state_id,party_id,votes,percentage,update_source

1,1,25000,35.2,District 1 Polling Station
```

### Include Directives

Use `@include` to include other RSU files:

```rsu
# Main vote data
1,1,25000,35.2,District 1 Polling Station
1,2,18000,25.4,District 2 Polling Station

# Include additional data from another file
@include "additional_votes.rsu"
```

**Included file (additional_votes.rsu):**

```rsu
# Additional vote data
2,1,30000,42.1,Early Voting Center
2,2,22000,30.8,Mail-in Ballots
```

### Formatting Rules

- **Empty lines**: Ignored
- **Whitespace**: Trimmed from field values
- **Field separation**: Comma-separated values
- **Minimum fields**: At least 4 fields required (state_id, party_id, votes, percentage)
- **Optional fields**: update_source is optional

### Validation Rules

- `state_id`: Must be a valid unsigned integer
- `party_id`: Must be a valid unsigned integer
- `votes`: Must be a valid integer ≥ 0
- `percentage`: Must be a valid float between 0 and 100
- `update_source`: If not provided, defaults to "RSU Upload: {filename}"

### Error Handling

The parser stops at the first error and returns a detailed error message:

```
Line 5: invalid state_id 'abc'
Line 3: insufficient columns - need at least 4, got 3: 1, 2
Line 7: percentage must be between 0 and 100
```

## State and Party ID Mappings

### States

| ID  | State Name               |
| --- | ------------------------ |
| 1   | Izzi                     |
| 2   | Freeda                   |
| 3   | Aurumna                  |
| 4   | North Aurumna            |
| 5   | Poisis                   |
| 6   | New South Western Poisis |
| 7   | Oorlogi                  |
| 8   | Pompat                   |

### Parties

| ID  | Party Name                 |
| --- | -------------------------- |
| 1   | People's Solidarity Front  |
| 2   | Progressive Alliance       |
| 3   | Centrist Union             |
| 4   | National Renewal Party     |
| 5   | Freedonian Heritage League |

## Integration Examples

### Simple Vote Upload

**votes.rsu:**

```rsu
# Simple vote data for Izzi
1,1,25000,35.2,District 1
1,2,18000,25.4,District 2
1,3,15000,21.1,District 3
```

**Upload:**

```bash
curl -X POST -F "file=@votes.rsu" "https://sync.synchronic27.labs.canva-internal.dev/api/votes/publish/Izzi"
```

## Error Handling and Troubleshooting

### Common Errors

**File Format Errors:**

```json
{
  "error": "File must be an RSU file (.rsu)"
}
```

**Parsing Errors:**

```json
{
  "error": "RSU parsing error: Line 3: invalid state_id 'abc'"
}
```

**Network Errors:**

```json
{
  "error": "Failed to call vparse service: connection refused"
}
```

### Best Practices

1. **Validate your RSU files** before uploading
2. **Use descriptive update_source** values for tracking
3. **Implement reconnection logic** for SSE connections
4. **Handle network timeouts** gracefully
5. **Test with small files** before uploading large datasets

### Testing

**Health Check:**

```bash
curl https://sync.synchronic27.labs.canva-internal.dev/healthz
```

**Test Upload:**

```bash
# Create test file
echo "1,1,1000,50.0,Test Upload" > test.rsu

# Upload
curl -X POST -F "file=@test.rsu" "https://sync.synchronic27.labs.canva-internal.dev/api/votes/publish/Izzi"
```

**Test Subscription:**

```bash
curl -N "https://sync.synchronic27.labs.canva-internal.dev/api/votes/subscribe/Izzi"
```

## Support

For technical support or questions about integration, contact the Freedonia Election Commission's technical team.

---

_This document is maintained by the Freedonia Election Commission. Last updated: 2025_
