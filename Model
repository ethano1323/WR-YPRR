# wr_projection_app.py

import streamlit as st
import pandas as pd
import numpy as np

st.set_page_config(page_title="NFL WR YPRR Projection", layout="wide")
st.title("NFL WR YPRR Projection Model")

# ------------------------
# Upload Data
# ------------------------
st.sidebar.header("Upload Data")
wr_file = st.sidebar.file_uploader("Upload WR Data CSV", type="csv")
def_file = st.sidebar.file_uploader("Upload Defense Data CSV", type="csv")

league_lead_routes = st.sidebar.number_input("League Leader Routes Run", min_value=1, value=100)

# ------------------------
# Interactive sliders
# ------------------------
st.sidebar.header("Model Adjustment Sliders")

sample_scaling_factor = st.sidebar.slider("Sample Size Scaling Factor", 0.0, 2.0, 1.0, 0.05)
coverage_weight_factor = st.sidebar.slider("Coverage Weight Factor", 0.0, 2.0, 1.0, 0.05)
historical_weight_adjust = st.sidebar.slider("Historical Weight Adjustment (1=current,2=last season/career scaling)", 0.0, 1.0, 1.0, 0.01)

# ------------------------
# Functions
# ------------------------

def historical_weighted_yprr(player_row, adjust_factor=1.0):
    games_played = player_row['games_played']
    last = player_row['last_season_yprr']
    career = player_row['career_yprr']
    current = player_row['base_yprr']
    
    if games_played < 4:  # first 4 games
        w_current = 0.2 if games_played > 0 else 0
        w_last = 0.45
        w_career = 0.35
        if w_current == 0:
            total = w_last + w_career
            w_last /= total
            w_career /= total
    else:
        w_current = 0.6 if games_played > 0 else 0
        w_last = 0.25
        w_career = 0.15
        if w_current == 0:
            total = w_last + w_career
            w_last /= total
            w_career /= total
    
    weighted = w_current*current + (w_last*adjust_factor)*last + (w_career*adjust_factor)*career
    return weighted

def sample_size_penalty(routes_played, league_lead_routes, scaling_factor=1.0):
    pct = routes_played / league_lead_routes
    if pct >= 0.75:
        return 1.0
    else:
        return max(0, pct/0.75 * scaling_factor)

def coverage_multiplier(yprr_split, base_yprr, coverage_routes, league_lead_routes, coverage_weight_factor=1.0):
    raw_multiplier = yprr_split / base_yprr
    penalty = sample_size_penalty(coverage_routes, league_lead_routes, scaling_factor=1.0)
    return raw_multiplier * penalty * coverage_weight_factor

def qb_adjustment(qb_epa_last8, qb_epa_last3yrs):
    return 0.7*qb_epa_last8 + 0.3*qb_epa_last3yrs

def compute_adjusted_yprr(wr_df, defense_profile, league_lead_routes, sample_scaling_factor, coverage_weight_factor, historical_weight_adjust):
    adjusted_results = []

    for _, row in wr_df.iterrows():
        hist_yprr = historical_weighted_yprr(row, adjust_factor=historical_weight_adjust)
        overall_sample_pen = sample_size_penalty(row['routes_played'], league_lead_routes, scaling_factor=sample_scaling_factor)

        # Coverage multipliers
        man_mult = coverage_multiplier(row['yprr_man'], row['base_yprr'], row['routes_man'], league_lead_routes, coverage_weight_factor)
        zone_mult = coverage_multiplier(row['yprr_zone'], row['base_yprr'], row['routes_zone'], league_lead_routes, coverage_weight_factor)
        onehigh_mult = coverage_multiplier(row['yprr_1high'], row['base_yprr'], row['routes_1high'], league_lead_routes, coverage_weight_factor)
        twohigh_mult = coverage_multiplier(row['yprr_2high'], row['base_yprr'], row['routes_2high'], league_lead_routes, coverage_weight_factor)
        blitz_mult = coverage_multiplier(row['yprr_blitz'], row['base_yprr'], row['routes_blitz'], league_lead_routes, coverage_weight_factor)
        standard_mult = coverage_multiplier(row['yprr_standard'], row['base_yprr'], row['routes_standard'], league_lead_routes, coverage_weight_factor)

        coverage_factor = (
            defense_profile['man_pct']*man_mult + defense_profile['zone_pct']*zone_mult
        ) * (
            defense_profile['onehigh_pct']*onehigh_mult + defense_profile['twohigh_pct']*twohigh_mult
        ) * (
            defense_profile['blitz_pct']*blitz_mult + defense_profile['noblitz_pct']*standard_mult
        )

        qb_adj = qb_adjustment(row['qb_epa_last8'], row['qb_epa_last3yrs'])

        adjusted_yprr = hist_yprr * overall_sample_pen * coverage_factor * row['season_route_share'] * qb_adj

        adjusted_results.append({
            "player": row['player'],
            "team": row['team'],
            "base_yprr": row['base_yprr'],
            "adjusted_yprr": adjusted_yprr
        })

    results_df = pd.DataFrame(adjusted_results)

    league_avg_base = wr_df['base_yprr'].mean()
    results_df['edge_over_base'] = results_df['adjusted_yprr'] - results_df['base_yprr']
    results_df['pct_edge_over_base'] = ((results_df['adjusted_yprr'] - results_df['base_yprr']) / results_df['base_yprr'] * 100).round(2)
    results_df['edge_vs_league'] = results_df['adjusted_yprr'] - league_avg_base

    results_df['rank_by_adj'] = results_df['adjusted_yprr'].rank(ascending=False, method='min').astype(int)
    results_df = results_df.sort_values('rank_by_adj')
    return results_df

# ------------------------
# Main App Execution
# ------------------------
if wr_file is not None and def_file is not None:
    wr_df = pd.read_csv(wr_file)
    defense_profile = pd.read_csv(def_file).iloc[0].to_dict()  # assume one row per defense

    # Add coverage-specific routes if not present
    for cov in ['man','zone','1high','2high','blitz','standard']:
        col_name = f'routes_{cov}'
        if col_name not in wr_df.columns:
            wr_df[col_name] = wr_df['routes_played'] * 0.5

    results_df = compute_adjusted_yprr(
        wr_df,
        defense_profile,
        league_lead_routes,
        sample_scaling_factor,
        coverage_weight_factor,
        historical_weight_adjust
    )

    st.subheader("Adjusted YPRR Rankings")
    st.dataframe(results_df)

    # Top 3 by adjusted YPRR
    st.subheader("Top 3 WRs by Adjusted YPRR")
    st.dataframe(results_df.head(3))

    # Top 3 by edge over base
    st.subheader("Top 3 WRs by Edge Over Base")
    st.dataframe(results_df.sort_values('pct_edge_over_base', ascending=False).head(3))

    # Top 10 positive edge "Targets"
    st.subheader("Top 10 Positive Edge WRs - 'Targets'")
    st.dataframe(results_df.sort_values('edge_over_base', ascending=False).head(10))

    # Top 10 negative edge "Fades"
    st.subheader("Top 10 Negative Edge WRs - 'Fades'")
    st.dataframe(results_df.sort_values('edge_over_base', ascending=True).head(10))

else:
    st.info("Please upload both WR data and Defense data CSV files.")
