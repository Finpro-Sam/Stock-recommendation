import pandas as pd
import plotly.graph_objs as go
import glob
import os
import re
from dash import Dash, dcc, html, Input, Output
import yagmail

# === CONFIGURATION ===
FOLDER_PATH = r'/data'  # Change to Render's persistent storage folder
OUTPUT_FILE = '/data/merged_stock_data.csv'

# === STEP 1: Load and Merge Data ===
csv_files = glob.glob(os.path.join(FOLDER_PATH, '*.csv'))
price_df = pd.DataFrame()
volume_df = pd.DataFrame()

for file in csv_files:
    try:
        match = re.search(r'_(\d{8})_', os.path.basename(file))
        if not match:
            print(f"Date not found in filename: {file}")
            continue

        raw_date = match.group(1)
        date = f"{raw_date[:4]}-{raw_date[4:6]}-{raw_date[6:]}"  # e.g., 2025-03-25

        df = pd.read_csv(file)

        if 'TckrSymb' not in df.columns or 'ClsPric' not in df.columns or 'TtlTradgVol' not in df.columns:
            print(f"Skipping file (missing columns): {file}")
            continue

        df.set_index('TckrSymb', inplace=True)
        price_df[date] = df['ClsPric']
        volume_df[date] = df['TtlTradgVol']

    except Exception as e:
        print(f"Error processing {file}: {e}")

# === STEP 2: Calculate Standard Deviations ===
price_df['price_std'] = price_df.std(axis=1)
price_df['mean_price'] = price_df.drop(columns=['price_std']).mean(axis=1)
volume_df['volume_std'] = volume_df.std(axis=1)

# === STEP 3: Merge and Save ===
merged_df = pd.concat([price_df, volume_df[['volume_std']]], axis=1)
merged_df.index.name = 'stock name'
merged_df.to_csv(OUTPUT_FILE)
print(f"Merged data saved to '{OUTPUT_FILE}'")

# === STEP 4: Helper Functions ===
def plot_stock_return_fig(stock_name):
    stock_prices = price_df.loc[stock_name].drop(['price_std', 'mean_price']).sort_index().astype(float)
    mean = price_df.loc[stock_name, 'mean_price']
    std = price_df.loc[stock_name, 'price_std']

    fig = go.Figure()

    fig.add_trace(go.Scatter(
        x=stock_prices.index,
        y=stock_prices.values,
        mode='lines+markers',
        name='Price',
        line=dict(color='royalblue', width=2),
        marker=dict(size=6)
    ))

    for i, sigma in enumerate([1, 2], start=1):
        fig.add_trace(go.Scatter(
            x=stock_prices.index,
            y=[mean + sigma * std] * len(stock_prices),
            mode='lines',
            name=f'+{i}σ',
            line=dict(color='green', dash='dot')
        ))
        fig.add_trace(go.Scatter(
            x=stock_prices.index,
            y=[mean - sigma * std] * len(stock_prices),
            mode='lines',
            name=f'-{i}σ',
            line=dict(color='red', dash='dot')
        ))

    fig.add_trace(go.Scatter(
        x=stock_prices.index,
        y=[mean] * len(stock_prices),
        mode='lines',
        name='Mean',
        line=dict(color='orange', dash='dash')
    ))

    fig.update_layout(
        title=f"📈 Price Trend with Standard Deviations: {stock_name}",
        xaxis_title='Date',
        yaxis_title='Price',
        template='plotly_white',
        hovermode='x unified'
    )

    return fig

def stocks_crossing_2nd_std():
    latest_date = price_df.columns[-3]  # Skip std and mean
    mean = price_df['mean_price']
    std = price_df['price_std']
    current_price = price_df[latest_date]
    return price_df[(current_price > (mean + 2 * std))].index.tolist()

def high_volume_spikes():
    volume_cols = volume_df.columns[:-1]  # Skip 'volume_std'
    last_day = volume_cols[-1]
    last_7_avg = volume_df[volume_cols[-7:]].mean(axis=1)
    return volume_df[volume_df[last_day] > last_7_avg].index.tolist()

def get_buzzing_stocks():
    std_symbols = stocks_crossing_2nd_std()
    vol_symbols = high_volume_spikes()
    unique_symbols = list(set(std_symbols).union(set(std_symbols).intersection(set(vol_symbols))))
    return unique_symbols

# === STEP 5: Dash Server App ===
app = Dash(__name__)

app.layout = html.Div([
    html.H2("📊 Stock Price Visualizer (Web Mode)", style={'color': 'white'}),

    dcc.Dropdown(
        id='stock-dropdown',
        options=[{'label': name, 'value': name} for name in price_df.index],
        placeholder="Select a stock...",
        style={'width': '50%'}
    ),
    dcc.Graph(id='price-graph'),

    html.Div([
        html.Div([
            html.H4("📈 2σ Breakouts", style={'color': 'white'}),
            html.Ul(id='std-list', style={'color': 'white'})
        ], style={'width': '32%', 'display': 'inline-block', 'verticalAlign': 'top'}),

        html.Div([
            html.H4("🔥 Volume Spikes", style={'color': 'white'}),
            html.Ul(id='volume-list', style={'color': 'white'})
        ], style={'width': '32%', 'display': 'inline-block', 'verticalAlign': 'top'}),

        html.Div([
            html.H4("🚀 Buzzing Stocks", style={'color': 'lightgreen'}),
            html.Ul(id='buzzing-list', style={'color': 'lightgreen'})
        ], style={'width': '32%', 'display': 'inline-block', 'verticalAlign': 'top'}),
    ], style={'marginTop': '30px'}),

    html.Div([
        dcc.Input(id='email-input', type='text', placeholder='Enter email(s), comma-separated'),
        html.Button('Send Report', id='send-btn', n_clicks=0),
        html.Div(id='send-status', style={'color': 'lightgreen'})
    ], style={'marginTop': '30px'})
], style={'backgroundColor': '#111111', 'padding': '20px', 'minHeight': '100vh'})

# === Callbacks ===
@app.callback(
    Output('price-graph', 'figure'),
    Input('stock-dropdown', 'value')
)
def update_graph(stock_name):
    if not stock_name:
        return go.Figure()
    return plot_stock_return_fig(stock_name)

@app.callback(
    Output('std-list', 'children'),
    Output('volume-list', 'children'),
    Output('buzzing-list', 'children'),
    Input('stock-dropdown', 'value')
)
def update_lists(_):
    std_list = [html.Li(stock) for stock in stocks_crossing_2nd_std()]
    vol_list = [html.Li(stock) for stock in high_volume_spikes()]
    buzz_list = [html.Li(stock) for stock in get_buzzing_stocks()]
    return std_list, vol_list, buzz_list

@app.callback(
    Output('send-status', 'children'),
    Input('send-btn', 'n_clicks'),
    Input('email-input', 'value'),
    prevent_initial_call=True
)
def send_email(n_clicks, email_str):
    if not email_str:
        return "Please enter at least one email."

    email_list = [e.strip() for e in email_str.split(',') if e.strip()]
    std_stocks = stocks_crossing_2nd_std()
    vol_stocks = high_volume_spikes()
    buzzing_stocks = get_buzzing_stocks()

    content = f"""
    2σ Breakouts:\n{', '.join(std_stocks) if std_stocks else 'None'}

    Volume Surges:\n{', '.join(vol_stocks) if vol_stocks else 'None'}

    Buzzing Stocks:\n{', '.join(buzzing_stocks) if buzzing_stocks else 'None'}
    """

    try:
        yag = yagmail.SMTP(user=os.getenv('EMAIL_USER'), password=os.getenv('EMAIL_PASSWORD'))
        yag.send(to=email_list, subject="📈 Stock Alert Summary", contents=content)
        return f"Email sent to {', '.join(email_list)}"
    except Exception as e:
        return f"Error sending email: {str(e)}"

# === RUN ===
if __name__ == "__main__":
    app.run(host='0.0.0.0', port=8080, debug=False)
