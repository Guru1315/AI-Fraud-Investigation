# 🔍 AI Fraud Investigation Agent

Live demo: https://ai-investigation-age-wiry.bolt.host

An AI agent that investigates licensed childcare providers using public data. It cross-references each provider's licensing record against physical evidence about the building, and flags places where the paperwork and the evidence do not match.

It is a Streamlit demo of [Surelock Homes](https://github.com/oso95/Surelock-Homes), an open-source fraud investigation system (see [Credit](#credit)).

> **Findings are leads, not conclusions.** The agent can only show that something looks inconsistent in public records. It is told never to call anything "fraud" and to use wording such as "anomaly" and "requires further investigation". Any flag needs checking by a person against official sources before anyone acts on it.

## Features

- Searches the Illinois DCFS licensing database for providers by ZIP code
- Compares licensed capacity with the building's square footage from Cook County property records
- Applies a building-code formula to work out the maximum number of children the building could legally hold
- Captures Google Street View images to see whether the address looks like a childcare facility
- Checks Google Places for the business at the address, its status, rating and reviews
- Checks business registration with the Illinois Secretary of State
- Looks for patterns across providers: shared owners, shared addresses, entities with no public footprint
- Streams its reasoning live, so you can follow the investigation as it happens

## How it works

You pick a ZIP code and click **Start Investigation**. An [Agno](https://github.com/agno-agi/agno) agent, running a model of your choice through [OpenRouter](https://openrouter.ai), then works through the providers in that ZIP. For each large Day Care Center it does a deep check; for small Day Care Homes it does a quick comparison of capacity against the legal limit. It writes up what it finds as it goes and ends with a summary of flagged providers and patterns.

The agent has seven tools, all built on public data:

| Tool | What it does | Source |
|---|---|---|
| `search_childcare_providers` | Lists active providers in a ZIP code with capacity, license type and status (shows up to 100 per search) | Illinois DCFS provider lookup |
| `get_property_data` | Building square footage, lot size, property class and year built | Cook County open data (Socrata, no key needed) |
| `calculate_max_capacity` | Maximum legal capacity for a given building size, with the working shown | IL DCFS Title 89, Part 407 |
| `get_street_view` | Four Street View images (headings 0°, 90°, 180°, 270°) | Google Street View API |
| `get_places_info` | Current business at the address, its status, rating and reviews | Google Places API |
| `geocode_address` | Turns an address into coordinates | Google Geocoding API |
| `check_business_registration` | Checks whether the provider is a registered legal entity. If the site blocks the request, it reports that and suggests a manual lookup | Illinois Secretary of State |

### The capacity formula

```
max legal children = (building square feet × 0.65) ÷ 35
```

The 35 square feet per child comes from the regulation the code cites. The 0.65 "usable space" ratio is the tool's own estimate of how much of a building is usable for childcare, not a figure from the regulation, so treat results near the limit with caution. A 900 sq ft building works out to 16 children, so a license for 50 stands out clearly.

## What it can and cannot detect

**Can flag**

- Licensed capacity that is physically impossible for the building size
- Addresses where Google shows a different business, or a closed or vacant building
- Providers with no Google listing, no reviews or no business registration
- Owner names or agents that appear across several providers

**Cannot detect**

- Attendance fraud (billing for children who did not attend). That needs non-public CCAP billing records
- Anything that needs internal DHS or county billing data

## Geographic scope

The demo covers **Cook County, Illinois** only, because the property data comes from the Cook County Assessor. The ZIP code list is limited to 10 high-density Chicago neighborhoods:

| ZIP | Neighborhood |
|---|---|
| 60623 | Little Village / North Lawndale |
| 60629 | Chicago Lawn |
| 60644 | Austin |
| 60621 | Englewood |
| 60628 | Roseland |
| 60619 | Chatham / Auburn Gresham |
| 60636 | West Englewood |
| 60612 | Near West Side |
| 60620 | Auburn Gresham |
| 60624 | Garfield Park |

The full Surelock Homes system supports Minnesota and additional Illinois counties, with a FastAPI backend, streaming dashboard and offline mode.

## Run it yourself

```bash
git clone https://github.com/Guru1315/AI-Fraud-Investigation.git
cd AI-Fraud-Investigation
pip install streamlit agno requests beautifulsoup4 openai
streamlit run fraud_investigation_agent.py
```

`requirements.txt` in this repository does not list these packages (see [Notes](#notes)), so install them as shown above. `openai` is included because Agno's OpenRouter model is built on the OpenAI client.

### Settings in the sidebar

| Setting | Required | Notes |
|---|---|---|
| OpenRouter API Key | Yes | From openrouter.ai. The **Start Investigation** button stays disabled until you enter it |
| Google Maps API Key | No | Needs the Geocoding, Places and Street View APIs enabled. Without it the agent skips the visual and Google Places checks |
| Model | Yes | `anthropic/claude-sonnet-4.6` (default), `anthropic/claude-opus-4.6`, `google/gemini-3.1-flash-lite-preview`, `openai/gpt-5.4` or `openai/gpt-4o` |
| ZIP Code | Yes | One of the 10 Cook County ZIPs above (default 60623) |

Keys are typed into the page and kept in the browser session. They are not saved to disk by the app. Use your own keys and watch your usage: a full ZIP code investigation makes many model and API calls.

## Project structure

```
fraud_investigation_agent.py   The whole app: tools, system prompt, Streamlit UI and agent
requirements.txt               See the note below
.github/workflows/             GitHub Actions workflow (lint and pytest)
README.md                      This file
```

## Notes

- **`requirements.txt` is out of date.** It lists `bolt host` and `google-genai`. The app actually imports `streamlit`, `agno`, `requests` and `beautifulsoup4`. A line like `bolt host` is not a valid requirement, so `pip install -r requirements.txt` is likely to fail. Replace the file with the packages in the install command above.
- **The GitHub Actions workflow probably fails.** It installs `requirements.txt` and then runs `pytest`, and the repository has no tests. Fixing the requirements and adding at least one test would let it pass.
- **Slow government site.** The Illinois DCFS site can be slow, so that lookup waits up to 30 seconds; other web requests time out after 10 seconds.
- **Some public sources are unreliable.** The Secretary of State lookup uses an endpoint that the code notes may return `403` from a CDN firewall. When that happens the tool returns a "blocked" result and points to https://apps.ilsos.gov/corporatellc/ for a manual check, so registration findings may be missing. The DCFS lookup reads a web page, so it can break if that site changes its layout.
- **Street View images** are shown below the narration after the investigation finishes.

## Credit

This is a demo built around the open-source [Surelock Homes](https://github.com/oso95/Surelock-Homes) project by its original authors. Check that repository for its license and for the full system.
