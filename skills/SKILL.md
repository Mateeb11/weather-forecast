---
name: weather
description: >
  Fetches current weather conditions and forecasts for any location worldwide.
  Use this skill whenever the user asks about weather, temperature, rain, wind,
  forecast, climate conditions, or what to wear/pack for a trip. Triggers on
  queries like "what's the weather in X", "forecast for Y", "will it rain",
  "what should I pack for Z", "how hot/cold is it", "is it windy in X",
  or any travel-planning question where weather matters.
---

# Weather Skill

Provides current weather and forecasts for any location using **Open-Meteo** free APIs called directly via `WebFetch`:
- **Open-Meteo Geocoding** for resolving location names to coordinates
- **Open-Meteo Forecast** for weather data

---

## Workflow

1. **Understand the user's request.** Extract the location, time range, and whether they're planning travel. See examples below.
2. **Geocode the location** using `WebFetch` to call the Nominatim API (see below). If the result is ambiguous, ask the user to clarify.
3. **Fetch weather** using `WebFetch` to call the Open-Meteo API with the lat/lon and appropriate parameters (see below).
4. **Respond to the user** with a clear summary. See example outputs below.
5. **If travel intent is detected**, also read `references/clothing_guide.md` and include relevant clothing advice.

---

## API Calls

### Step 1: Geocode with Open-Meteo Geocoding

Use `WebFetch` to GET:

```
https://geocoding-api.open-meteo.com/v1/search?name={location}&count=3&language=en&format=json
```

Replace `{location}` with the URL-encoded location string (e.g., `Tokyo`, `Riyadh`).

**Response** is a JSON object with a `results` array. Each item has:
- `name` — location name
- `latitude`, `longitude` — coordinates (floats)
- `country` — country name
- `admin1` — state/region (optional)
- `population` — population count (optional, useful for disambiguation)

**Ambiguity logic:** If there are ≥ 2 results in different countries or regions and neither clearly dominates by population, treat the result as ambiguous. Present each candidate's `name`, `admin1`, and `country` and ask the user which they meant.

If the `results` key is missing or the array is empty, the location was not found — ask the user to be more specific.

### Step 2: Fetch Weather from Open-Meteo

Use `WebFetch` to GET the Open-Meteo forecast URL, building it from the resolved coordinates and the desired resolution.

**Base URL:**
```
https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&timezone=auto&forecast_days=16
```

**Always include** the `current` parameter:
```
&current=temperature_2m,apparent_temperature,weather_code,wind_speed_10m,relative_humidity_2m,precipitation
```

**Add parameters based on resolution:**

| Resolution | Additional params |
|---|---|
| `current` (now + next 24h) | `&hourly=temperature_2m,apparent_temperature,precipitation_probability,precipitation,weather_code,wind_speed_10m` |
| `hourly` (specific day / short range) | `&hourly=temperature_2m,apparent_temperature,precipitation_probability,precipitation,weather_code,wind_speed_10m&daily=temperature_2m_max,temperature_2m_min,apparent_temperature_max,apparent_temperature_min,precipitation_sum,precipitation_probability_max,weather_code,wind_speed_10m_max` |
| `daily` (multi-day / weekly) | `&daily=temperature_2m_max,temperature_2m_min,apparent_temperature_max,apparent_temperature_min,precipitation_sum,precipitation_probability_max,weather_code,wind_speed_10m_max` |

**Optional params:**
- `&past_days=N` — include N historical days (0–92). Use only when the user asks about past weather.
- `&timezone=Asia/Tokyo` — override auto timezone (rarely needed).

**Response structure:**
- `current` object: `time`, `temperature_2m`, `apparent_temperature`, `weather_code`, `wind_speed_10m`, `relative_humidity_2m`, `precipitation`
- `hourly` object: `time[]`, `temperature_2m[]`, `apparent_temperature[]`, `precipitation_probability[]`, `precipitation[]`, `weather_code[]`, `wind_speed_10m[]`
- `daily` object: `time[]`, `temperature_2m_max[]`, `temperature_2m_min[]`, `apparent_temperature_max[]`, `apparent_temperature_min[]`, `precipitation_sum[]`, `precipitation_probability_max[]`, `weather_code[]`, `wind_speed_10m_max[]`

### WMO Weather Code Table

Use this table to convert the numeric `weather_code` in API responses to human-readable descriptions:

| Code | Description |
|---|---|
| 0 | Clear sky |
| 1 | Mainly clear |
| 2 | Partly cloudy |
| 3 | Overcast |
| 45 | Foggy |
| 48 | Rime fog |
| 51 | Light drizzle |
| 53 | Moderate drizzle |
| 55 | Dense drizzle |
| 56 | Light freezing drizzle |
| 57 | Dense freezing drizzle |
| 61 | Slight rain |
| 63 | Moderate rain |
| 65 | Heavy rain |
| 66 | Light freezing rain |
| 67 | Heavy freezing rain |
| 71 | Slight snow |
| 73 | Moderate snow |
| 75 | Heavy snow |
| 77 | Snow grains |
| 80 | Slight showers |
| 81 | Moderate showers |
| 82 | Violent showers |
| 85 | Slight snow showers |
| 86 | Heavy snow showers |
| 95 | Thunderstorm |
| 96 | Thunderstorm w/ hail |
| 99 | Thunderstorm w/ heavy hail |

---

## How to Interpret User Requests

The agent should interpret the user's natural language to determine three things:

### 1. Location
Extract the place name from the query. Examples:
- "Weather in **Jeddah** now" → `"Jeddah"`
- "Forecast for **London** this weekend" → `"London"`
- "I'm going to **Tokyo** next week" → `"Tokyo"`
- "How windy will it be in **AlUla** on Friday?" → `"AlUla"`

If no location is mentioned, ask the user.

### 2. Time Range, Resolution, and Past Weather

| User says | Resolution | `past_days` | Notes |
|---|---|---|---|
| "now" / "current" / no time specified | `current` (also include hourly for next 24h) | `0` | Forecast request always uses 16 days. |
| "today" / "tonight" / "tomorrow" / "tomorrow morning" | `hourly` | `0` | Use hourly data to answer specific time windows. |
| "this weekend" / "on Friday" / "hourly for next 12 hours" | `hourly` | `0` | Filter the returned series in the response. |
| "next 3 days" | `hourly` | `0` | Summarize from hourly data. |
| "next 7 days" / "next week" / "first week of March" | `daily` | `0` | Summarize relevant days from daily data. |
| "March" / "in April" / "05/01 to 06/30" | `daily` | `0` | If outside forecast horizon, clearly state limits. |
| "past weather" / "last 3 days" / "yesterday" | `daily` (or `hourly` for short windows) | set from request | If user says "past" without a number, default to `7`. |

The user may specify dates in various formats (MM/DD, DD/MM, "May 1st", "1 March", etc.) — interpret based on context and filter the returned timeline accordingly. Use the current year unless another year is stated or implied.

Use `daily` for ranges > 3 days, `hourly` for shorter windows, `current` for right-now queries (but still include hourly for context).

**Default** (no time specified): current conditions + next 24 hours hourly.

**Remember:** Forecast is fixed to 16 days. For past requests, use `past_days` (`0..92`).

### 3. Travel Intent

Detect if the user is planning to go somewhere. Look for keywords like:
**pack, wear, travel, visiting, going to, headed to, trip, vacation, holiday, outing, flight, should I bring**

If travel intent is detected → also include clothing guidance (see below).

---

## Example Outputs

### Example 1: Simple current weather

**User:** "What's the weather in Jeddah now?"

**Response:**

> **Weather in Jeddah, Saudi Arabia**
>
> Right now it's **Clear sky** with a temperature of **28°C** (feels like 30°C). Wind is light at **12 km/h** from the north, and humidity is at 45%. No precipitation expected.
>
> **Next few hours:**
> - 3 PM: 29°C, Clear sky, 0% rain
> - 4 PM: 28°C, Partly cloudy, 0% rain
> - 5 PM: 26°C, Clear sky, 0% rain
> *(and so on...)*

### Example 2: Weekend forecast

**User:** "Forecast for London this weekend"

**Response:**

> **Weekend Forecast for London, UK**
>
> **Saturday, Mar 1:** Highs of 11°C, lows of 5°C. Moderate rain expected (70% probability, ~4mm). Wind up to 25 km/h.
>
> **Sunday, Mar 2:** Highs of 13°C, lows of 6°C. Partly cloudy, 20% rain chance. Lighter wind at 15 km/h.
>
> Overall: A wet Saturday clearing into a drier Sunday. Bring an umbrella for Saturday.

### Example 3: Travel planning with clothing advice

**User:** "I'm going to Tokyo next week — what should I pack?"

**Response:**

> **Tokyo, Japan — Next Week Forecast (Mar 3–9)**
>
> | Day | High | Low | Conditions | Rain | Wind |
> |---|---|---|---|---|---|
> | Mon | 12°C | 4°C | Partly cloudy | 10% | 15 km/h |
> | Tue | 10°C | 3°C | Slight rain | 65% | 20 km/h |
> | ... | ... | ... | ... | ... | ... |
>
> **What to Wear:**
> Expect **cold** conditions (lows near 3°C) with **rain** mid-week. Pack:
> - Warm layers: fleece or wool sweater + insulated jacket
> - Waterproof jacket and umbrella for Tuesday/Wednesday
> - Closed-toe waterproof shoes
> - Scarf and light gloves for mornings/evenings
>
> *See: clothing_guide.md — sections: cold, rainy*

### Example 4: Specific condition query

**User:** "How windy will it be in AlUla on Friday?"

**Response:**

> **Wind Forecast for AlUla, Saudi Arabia — Friday, Feb 28**
>
> Wind speeds will peak at **35 km/h** in the early afternoon, settling to around 15 km/h by evening. Morning winds are lighter at ~10 km/h.
>
> ⚠️ Moderately windy conditions expected midday — secure loose items and consider a windbreaker if you'll be outdoors.

---

## Warnings

Include warnings when conditions are notable:
- **Extreme heat:** temp ≥ 40°C
- **Extreme cold:** temp ≤ -10°C
- **Strong wind:** ≥ 50 km/h
- **Heavy rain:** ≥ 20mm precipitation

---

## Clothing Guidance

When travel intent is detected, read `references/clothing_guide.md` and include relevant advice in the response.

---

## Error Handling

- **Location not found:** The Nominatim response is an empty array. Ask the user to be more specific (add city, country).
- **Ambiguous location:** Present the alternatives and ask which one.
- **WebFetch failure:** If `WebFetch` returns an error or non-JSON response, inform the user that the weather service is temporarily unavailable and suggest retrying.
- **Beyond forecast horizon:** Inform the user of the 16-day limit, return what's available.
- **No time specified:** Default to current + next 24 hours.
