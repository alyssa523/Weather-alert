#!/usr/bin/env python3
"""
Weather Alert Bot for Minneapolis, MN area (15-mile radius outside city limits)

Checks Open-Meteo for temperature extremes, precipitation/storms, and air
quality, and posts a Discord alert when notable conditions are found.

SCHEDULING:
    This script checks conditions once per run. Run it every 30 minutes
    using one of:
      - cron:            */30 * * * * /usr/bin/python3 /path/to/weather_alert_bot.py
      - GitHub Actions:  schedule: - cron: "*/30 * * * *"
      - Built-in loop:   python3 weather_alert_bot.py --loop
                         (runs forever, checking every 30 minutes; use this
                         only if the process itself stays running, e.g. in
                         a container or background service)

The bot keeps a small state file so it only alerts when the situation
actually changes, instead of re-posting the same alert every run.

Env vars required:
    DISCORD_WEBHOOK_URL - Discord webhook URL to post alerts to

Dependencies:
    pip install requests
"""

import argparse
import json
import os
import sys
import time
from pathlib import Path

import requests

# ---------------------------------------------------------------------------
# Configuration
# ---------------------------------------------------------------------------

# Minneapolis city center. Open-Meteo returns a point forecast, so this single
# point is used to represent conditions across the ~15 mile radius around the
# city. If you want finer coverage, add more (lat, lon) pairs and check each.
LATITUDE = 44.9778
LONGITUDE = -93.2650

TEMP_LOW_THRESHOLD_F = 32
TEMP_HIGH_THRESHOLD_F = 85

# US AQI: 0-50 Good, 51-100 Moderate, 101-150 Unhealthy for Sensitive Groups,
# 151+ Unhealthy. "Poor" air quality alert fires at 101+.
AQI_POOR_THRESHOLD = 101

# How many hours ahead to scan for upcoming conditions.
FORECAST_HOURS = 24

# How often to poll when run in --loop mode.
CHECK_INTERVAL_SECONDS = 30 * 60  # 30 minutes

STATE_FILE = Path(__file__).with_name("weather_alert_state.json")

DISCORD_WEBHOOK_URL = os.environ.get("DISCORD_WEBHOOK_URL")

WEATHER_API_URL = "https://api.open-meteo.com/v1/forecast"
AIR_QUALITY_API_URL = "https://air-quality-api.open-meteo.com/v1/air-quality"
NWS_ALERTS_API_URL = "https://api.weather.gov/alerts/active"

# NWS requires a descriptive User-Agent identifying the app/contact.
NWS_HEADERS = {
    "User-Agent": "MinneapolisWeatherAlertBot (contact: alyssa.neely@my.uwrf.edu)",
    "Accept": "application/geo+json",
}

# WMO weather codes -> human description and category.
# NOTE: Open-Meteo has no distinct "tornado" weather code (severe storms only
# show up as thunderstorm codes 95/96/99). Actual tornado watches/warnings
# come from the NWS alerts API via fetch_tornado_alerts() below.
WEATHER_CODES = {
    0: ("clear sky", None),
    1: ("mainly clear", None),
    2: ("partly cloudy", None),
    3: ("overcast", None),
    45: ("fog", None),
    48: ("depositing rime fog", None),
    51: ("light drizzle", "rain"),
    53: ("moderate drizzle", "rain"),
    55: ("dense drizzle", "rain"),
    56: ("light freezing drizzle", "rain"),
    57: ("dense freezing drizzle", "rain"),
    61: ("slight rain", "rain"),
    63: ("moderate rain", "rain"),
    65: ("heavy rain", "rain"),
    66: ("light freezing rain", "rain"),
    67: ("heavy freezing rain", "rain"),
    71: ("slight snow fall", "snow"),
    73: ("moderate snow fall", "snow"),
    75: ("heavy snow fall", "snow"),
    77: ("snow grains", "snow"),
    80: ("slight rain showers", "rain"),
    81: ("moderate rain showers", "rain"),
    82: ("violent rain showers", "rain"),
    85: ("slight snow showers", "snow"),
    86: ("heavy snow showers", "snow"),
    95: ("thunderstorm", "storm"),
    96: ("thunderstorm with slight hail", "storm"),
    99: ("thunderstorm with heavy hail", "storm"),
}


# ---------------------------------------------------------------------------
# Data fetching
# ---------------------------------------------------------------------------

def fetch_weather():
    params = {
        "latitude": LATITUDE,
        "longitude": LONGITUDE,
        "hourly": "temperature_2m,precipitation,weathercode",
        "temperature_unit": "fahrenheit",
        "forecast_days": 2,
        "timezone": "America/Chicago",
    }
    resp = requests.get(WEATHER_API_URL, params=params, timeout=15)
    resp.raise_for_status()
    return resp.json()


def fetch_air_quality():
    params = {
        "latitude": LATITUDE,
        "longitude": LONGITUDE,
        "hourly": "us_aqi",
        "forecast_days": 2,
        "timezone": "America/Chicago",
    }
    resp = requests.get(AIR_QUALITY_API_URL, params=params, timeout=15)
    resp.raise_for_status()
    return resp.json()


def fetch_tornado_alerts():
    """Query NWS active alerts for the point and return tornado watches/warnings."""
    params = {
        "point": f"{LATITUDE},{LONGITUDE}",
        "status": "actual",
        "message_type": "alert",
    }
    resp = requests.get(NWS_ALERTS_API_URL, params=params, headers=NWS_HEADERS, timeout=15)
    resp.raise_for_status()
    data = resp.json()

    tornado_alerts = []
    for feature in data.get("features", []):
        props = feature.get("properties", {})
        event = props.get("event", "")
        if "tornado" in event.lower():
            tornado_alerts.append({
                "id": props.get("id"),
                "event": event,
                "headline": props.get("headline", event),
                "expires": props.get("expires"),
            })
    return tornado_alerts


# ---------------------------------------------------------------------------
# Condition checking
# ---------------------------------------------------------------------------

def check_conditions(weather_data, air_data, tornado_alerts):
    """Scan the next FORECAST_HOURS hours and return a list of alert dicts."""
    alerts = []

    times = weather_data["hourly"]["time"][:FORECAST_HOURS]
    temps = weather_data["hourly"]["temperature_2m"][:FORECAST_HOURS]
    codes = weather_data["hourly"]["weathercode"][:FORECAST_HOURS]
    precip = weather_data["hourly"]["precipitation"][:FORECAST_HOURS]
    aqi_values = air_data["hourly"]["us_aqi"][:FORECAST_HOURS]

    # Temperature extremes
    max_temp = max(temps)
    min_temp = min(temps)
    if max_temp > TEMP_HIGH_THRESHOLD_F:
        hot_time = times[temps.index(max_temp)]
        alerts.append({
            "type": "heat",
            "detail": f"a high of {max_temp:.0f}\u00b0F expected around {hot_time}",
        })
    if min_temp < TEMP_LOW_THRESHOLD_F:
        cold_time = times[temps.index(min_temp)]
        alerts.append({
            "type": "cold",
            "detail": f"a low of {min_temp:.0f}\u00b0F expected around {cold_time}",
        })

    # Precipitation / storm codes
    seen_categories = set()
    for t, code, p in zip(times, codes, precip):
        label, category = WEATHER_CODES.get(code, (f"code {code}", None))
        if category and category not in seen_categories and p > 0:
            seen_categories.add(category)
            alerts.append({
                "type": category,
                "detail": f"{label} expected around {t}",
            })

    # Air quality
    max_aqi = max(aqi_values)
    if max_aqi >= AQI_POOR_THRESHOLD:
        aqi_time = times[aqi_values.index(max_aqi)]
        alerts.append({
            "type": "air_quality",
            "detail": f"US AQI reaching {max_aqi:.0f} around {aqi_time}",
        })

    # Tornado watches/warnings from NWS
    for tornado in tornado_alerts:
        alerts.append({
            "type": f"tornado:{tornado['id']}",
            "detail": tornado["headline"].lower(),
        })

    return alerts


# ---------------------------------------------------------------------------
# State handling (avoid duplicate/repeat alerts)
# ---------------------------------------------------------------------------

def load_state():
    if STATE_FILE.exists():
        try:
            return json.loads(STATE_FILE.read_text())
        except (json.JSONDecodeError, OSError):
            return {}
    return {}


def save_state(state):
    STATE_FILE.write_text(json.dumps(state))


def filter_new_alerts(alerts, state):
    """Only keep alert types that weren't already flagged in the last run."""
    previous_types = set(state.get("active_types", []))
    current_types = {a["type"] for a in alerts}
    new_alerts = [a for a in alerts if a["type"] not in previous_types]
    state["active_types"] = list(current_types)
    return new_alerts


# ---------------------------------------------------------------------------
# Discord posting
# ---------------------------------------------------------------------------

def format_message(alerts):
    intro = "Weather alert for the Minneapolis area (city limits + 15 miles):"
    sentences = [intro]
    for alert in alerts:
        sentences.append(f"{alert['detail'].capitalize()}.")
    return " ".join(sentences)


def send_discord(message):
    if not DISCORD_WEBHOOK_URL:
        print("DISCORD_WEBHOOK_URL is not set; skipping post.", file=sys.stderr)
        return
    resp = requests.post(DISCORD_WEBHOOK_URL, json={"content": message}, timeout=15)
    resp.raise_for_status()


# ---------------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------------

def run_once():
    state = load_state()

    weather_data = fetch_weather()
    air_data = fetch_air_quality()
    tornado_alerts = fetch_tornado_alerts()

    alerts = check_conditions(weather_data, air_data, tornado_alerts)
    new_alerts = filter_new_alerts(alerts, state)

    if new_alerts:
        message = format_message(new_alerts)
        send_discord(message)
        print(message)
    else:
        print("No new notable weather conditions.")

    save_state(state)


def main():
    parser = argparse.ArgumentParser(description="Minneapolis-area weather alert bot")
    parser.add_argument(
        "--loop",
        action="store_true",
        help=f"Run continuously, checking every {CHECK_INTERVAL_SECONDS // 60} minutes "
             "instead of exiting after one check.",
    )
    args = parser.parse_args()

    if args.loop:
        while True:
            try:
                run_once()
            except requests.RequestException as exc:
                print(f"Request failed: {exc}", file=sys.stderr)
            time.sleep(CHECK_INTERVAL_SECONDS)
    else:
        run_once()


if __name__ == "__main__":
    main()
