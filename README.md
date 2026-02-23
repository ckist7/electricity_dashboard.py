# electricity_dashboard.py
"""
Electricity Usage Dashboard
Author: Chris
Course: Data Analytics Assignment

Description:
This script connects to PostgreSQL using environment variables,
loads electricity usage data, performs analysis, and creates
interactive visualizations using Plotly and Dash.
"""
"""
REFLECTION:

The most challenging part of this assignment was connecting the PostgreSQL
but using what I learned last week on the assignment I was able to overcome 
alot of it. 

How I overcame these challenges:
I overcame these challenges by reviewing previous working database scripts,
testing the connection step-by-step, and verifying the data types after loading.
Using pandas datetime functions helped transform the data into usable formats
for visualization.

New concepts and skills learned:
Through this assignment, I learned how to:
- Use environment variables securely
- Transform time-series data using pandas
- Create interactive visualizations with Plotly
- Build interactive dashboards using Dash
- Design dashboards useful for real-world data analysis

"""

# Import libraries


import os
import pandas as pd
from sqlalchemy import create_engine
from dotenv import load_dotenv

import plotly.express as px
import dash
from dash import dcc, html


# 1. Load environment variables (FROM YOUR CODE)

load_dotenv()

DB_USER = os.getenv("DB_USER")
DB_PASSWORD = os.getenv("DB_PASSWORD")
DB_HOST = os.getenv("DB_HOST")
DB_PORT = os.getenv("DB_PORT", "5432")
DB_NAME = os.getenv("DB_NAME")

if not all([DB_USER, DB_PASSWORD, DB_HOST, DB_NAME]):
    raise ValueError("Missing one or more required environment variables in .env file")


# Build connection string (FROM YOUR CODE)
connection_string = (
    f"postgresql+psycopg2://{DB_USER}:{DB_PASSWORD}@{DB_HOST}:{DB_PORT}/{DB_NAME}"
)

engine = create_engine(connection_string)



# 2. SQL SELECT to pull ALL data (From Previous Assignment)

query = """
SELECT *
FROM electricity.usage_data
ORDER BY interval_end_date;
"""

print("Loading all data from electricity.usage_data ...")

df = pd.read_sql(query, engine)

print(f"Successfully loaded {len(df):,} rows total.")

# 3. Show first 10 rows (From Previous Assignment)


print("\nFirst 10 rows of the data:")
print(df.head(10).to_string(index=False))

# 4. Data Preparation (EXPANDED FOR DASHBOARD)


df['interval_end_date'] = pd.to_datetime(df['interval_end_date'])

# Create additional time columns
df['date'] = df['interval_end_date'].dt.date
df['month'] = df['interval_end_date'].dt.to_period('M').astype(str)
df['day_of_week'] = df['interval_end_date'].dt.day_name()
df['hour'] = df['interval_end_date'].dt.hour


# 5. Monthly aggregation (From Previous Assignment)

monthly_usage = (
    df.groupby(df['interval_end_date'].dt.to_period('M'))['kwh']
    .sum()
    .reset_index(name='total_kwh')
)

monthly_usage['month'] = monthly_usage['interval_end_date'].astype(str)
monthly_usage = monthly_usage[['month', 'total_kwh']]

print("\nMonthly total electricity usage:")
print(monthly_usage.to_string(index=False))

total_usage = monthly_usage['total_kwh'].sum()

print(f"\nTotal usage across all months: {total_usage:,.1f} kWh")


# 6. Visualization 1: Trend over time


daily_usage = df.groupby('date')['kwh'].sum().reset_index()

fig_trend = px.line(
    daily_usage,
    x='date',
    y='kwh',
    title="Electricity Usage Trend Over Time"
)

# 7. Visualization 2: Usage by day of week


dow_usage = df.groupby('day_of_week')['kwh'].sum().reset_index()

days_order = [
    "Monday", "Tuesday", "Wednesday",
    "Thursday", "Friday", "Saturday", "Sunday"
]

dow_usage['day_of_week'] = pd.Categorical(
    dow_usage['day_of_week'],
    categories=days_order,
    ordered=True
)

dow_usage = dow_usage.sort_values('day_of_week')

fig_dow = px.bar(
    dow_usage,
    x='day_of_week',
    y='kwh',
    title="Electricity Usage by Day of Week"
)


# 8. Visualization 3: Usage by hour


hour_usage = df.groupby('hour')['kwh'].mean().reset_index()

fig_hour = px.line(
    hour_usage,
    x='hour',
    y='kwh',
    title="Average Electricity Usage by Hour"
)

# 9. Create Dash Dashboard


app = dash.Dash(__name__)

app.layout = html.Div([

    html.H1(
        "Electricity Usage Dashboard",
        style={'textAlign': 'center'}
    ),

    html.H3("Trend Over Time"),
    dcc.Graph(figure=fig_trend),

    html.H3("Usage by Day of Week"),
    dcc.Graph(figure=fig_dow),

    html.H3("Usage by Hour of Day"),
    dcc.Graph(figure=fig_hour),

])

# 10. Run App

if __name__ == "__main__":

    print("\nStarting Dash dashboard...")

    print("\nOpen your browser to:")
    print("http://127.0.0.1:8050")

    app.run(debug=True)
